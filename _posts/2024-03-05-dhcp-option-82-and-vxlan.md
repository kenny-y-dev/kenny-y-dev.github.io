---
title:  "DHCP Option 82 with EVPN-VXLAN"
layout: post
---

Recently I came across a perplexing DHCP issue in an EVPN-VXLAN deployment where Windows Server was incorrectly handling DHCP requests that contained Option 82, a necessary component when using Anycast Gateways. Since the DHCP configuration with EVPN/VXLAN/Anycast a little abstract, let's do a quick refresher that builds to the problematic implementation.


# The Architecture
The setup we're using today is pretty basic, but representative of components you might see in a typical campus network: an instance of Windows Server 2022, a Layer 3 capable switch running Aruba's AOS-CX, and a client device. The client is also running Windows Server 2022 to slightly streamline the setup, as it'll generate DHCP activity similar to a typical Windows PC an end user may have.

# The normal DHCP Process, Relays, and IP Helper
The basic functionality of DHCP is most easily seen on a flat Layer 2 network, where DHCP server and DHCP client reside on the same network segment. Imagine a new client joins the network, but does not yet have an IP address. Since the client does not currently have a unique way to identify itself at Layer 3, it cannot talk to other devices via unicast, so it uses broadcast broadcast to communicate. The client sends its `DHCPDISCOVER` message to `255.255.255.255` to trigger a broadcast of the packet across the entire network segment, and continues using broadcast to talk to the server until it locks in its IP address after a `DHCPACK` from the DHCP server.

But what if there isn't a DHCP server on the client's network segment?

Enter DHCP relays, commonly seen as `ip-helper` entries in switch configurations. This setting maps one or more DHCP servers to a particular network segment, even if the DHCP server is part of a completely separate network. When a switch interface on the network segment with the `ip-helper` configuration sees a DHCP client's broadcast messages, it rewrites them as unicast packets using its own IP address as the source interface, adds its address to a `giaddr` field in the DHCP message, and then sends them to the serves defined in the `ip-helper` entries. The DHCP server sees this request, and uses the address in the `giaddr` field to identify the client's network origin, and from there the DHCP process works the same way. The network switch handles this DHCP relay function seamlessly in the background, and the relay process is transparent to the client device. This allows for an incredible amount of flexibility in deploying centralized DHCP services across large networks with multiple disparate segments.

[picture of DHCP relay and rewrites]
*DHCP relay rewrites the source/destination IPs from relay to server, but maintains all of the same data on the connection to the client*

# DHCP Option 82
DHCP options are used to encode data in DHCP messages, allowing configuration to be exchanged between client and server during the DHCP session. Some common DHCP options are Option 3 to provide the default gateway, or Option 6 to provide DNS servers.

DHCP option 82 was introduced in RFC 3046 as a way to for DHCP relays to add additional network context data, and later RFC 3527 expanded option 82 to allow explicit specification of an address that should be used for subnet selection during the DHCP process.

Thinking back to DHCP relays, why would we need this functionality if the `giaddr` field is used to determine the client's network? Well, occasionally there are network configurations where the DHCP relay must be originated from an interface different than the one that resides on a DHCP client's network. Common use cases are when the client is on a different VRF than the DHCP server, or EVPN deployments where anycast gateways are used, and there is not a clear return path for DHCP server traffic due to the duplicated anycast IPs throughout a network segment. In these cases, DHCP requests are relayed out of a address on the switch that can reach the DHCP server, but as a side effect, this address is set as the `giaddr`, disrupting the usual address selection mechanisms of DHCP. The expansion of option 82 in RFC 3527 allows for the DHCP relay to add an additional address from the source network, prompting a client address selection from the correct network on the DHCP server.

