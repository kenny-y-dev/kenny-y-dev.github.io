---
title:  "DHCP Option 82 with EVPN-VXLAN"
layout: post
---

Recently I came across a perplexing DHCP issue in an EVPN-VXLAN deployment where Windows Server was incorrectly handling DHCP requests that contained Option 82, a necessary component when using Anycast Gateways. Since the DHCP configuration with EVPN/VXLAN/Anycast a little abstract, let's do a quick refresher that builds to the problematic implementation.


# The Architecture
The setup we're using today is pretty basic, but representative of components you might see in a typical campus network: an instance of Windows Server 2022, a Layer 3 capable switch running Aruba's AOS-CX, and a client device. The client is also running Windows Server 2022 to slightly streamline the setup, as it'll generate DHCP activity similar to a typical Windows PC an end user may have.

{% include image.html url="/assets/images/2024-03-05-dhcp-option-82-and-vxlan/0_dhcp-option-82-lab-diagram.png" description="Three device setup to generate DHCP traffic for testing. Addresses listed for additional clarity in packet captures below." %}

# The Normal DHCP Process, Relays, and IP Helper
The basic functionality of DHCP is most easily seen on a flat Layer 2 network, where DHCP server and DHCP client reside on the same network segment. Imagine a new client joins the network, but does not yet have an IP address. Since the client does not currently have a unique way to identify itself at Layer 3, it cannot talk to other devices via unicast, so it uses broadcast broadcast to communicate. The client sends its `DHCPDISCOVER` message to `255.255.255.255` to trigger a broadcast of the packet across the entire network segment, and continues using broadcast to talk to the server until it locks in its IP address after a `DHCPACK` from the DHCP server.

{% include image.html url="/assets/images/2024-03-05-dhcp-option-82-and-vxlan/1_regular-router-client.png" description="DHCP exchange from the perspective of the client device." %}

But what if there isn't a DHCP server on the client's network segment?

Enter DHCP relays, commonly seen as `ip-helper` entries in switch configurations. This setting maps one or more DHCP servers to a particular network segment, even if the DHCP server is part of a completely separate network. When a switch interface on the network segment with the `ip-helper` configuration sees a DHCP client's broadcast messages, it rewrites them as unicast packets using its own IP address as the source interface, adds its address to a `giaddr` field in the DHCP message, and then sends them to the serves defined in the `ip-helper` entries. The DHCP server sees this request, and uses the address in the `giaddr` field to identify the client's network origin, and from there the DHCP process works the same way. The network switch handles this DHCP relay function seamlessly in the background, and the relay process is transparent to the client device. This allows for an incredible amount of flexibility in deploying centralized DHCP services across large networks with multiple disparate segments.

{% include image.html url="/assets/images/2024-03-05-dhcp-option-82-and-vxlan/2_regular-router-server.png" description="The same exchange from above, but captured between the network router and DHCP server. The DHCP relay feature translates the source of client messages so they can be exchanged outside of the network segment." %}

# DHCP Option 82
DHCP options are used to encode data in DHCP messages, allowing configuration to be exchanged between client and server during the DHCP session. Some common DHCP options are Option 3 to provide the default gateway, or Option 6 to provide DNS servers.

DHCP option 82 was introduced in RFC 3046 as a way to for DHCP relays to add additional network context data from the DHCP relay itself, and later RFC 3527 expanded option 82 to allow explicit specification of an address that should be used for subnet selection during the DHCP process.

Thinking back to DHCP relays, why would we need to explicitly specify subnet selection if the `giaddr` field is used to determine the client's network? Well, occasionally there are network configurations where the DHCP relay must be sourced from an interface different than the one that resides on a DHCP client's network. Perhaps you have different VRFs for clients, but still want centralized DHCP services on a separate VRF. Or you may be using EVPN with anycast gateways duplicated throughout the network, in which case there is not a clear return path if the anycast gateway is used to relay DHCP.

{% include image.html url="/assets/images/2024-03-05-dhcp-option-82-and-vxlan/3_router-server-option-82-working.png" description="Relayed DHCP session with option 82 values. The gateway from the source network and the VRF name are encoded as sub-options." %}

In these cases, DHCP requests are relayed out of a address on the switch that can reach the DHCP server, usually a loopback address on a different network than any DHCP clients. As a side effect, this address is set as the `giaddr` in the DHCP messages, which disrupts the usual address selection mechanisms of DHCP. This is where option 82 comes into play! The expansion of option 82 in RFC 3527 allows for the DHCP relay to add an additional address from the source network of the DHCP request, prompting a client address selection from the correct network on the DHCP server!

# Where things start to go wrong...
This is all great in theory, but does it actually work when everything is configured? Well... maybe not if you're deploying somewhere with heavy use of Windows domain services, where Windows Server's DHCP implementation is handing out the addresses.

According to Microsoft, Windows Server [supports](https://learn.microsoft.com/en-us/windows-server/networking/technologies/dhcp/dhcp-subnet-options) DHCP option 82, and sub-option 5 for link selection, as long as you're on Server 2016 or later; awesome! This works as expected, and addresses are handed out using the network information from option 82 sub-option 5.

Issues arise, however, when the network connected to the DHCP client changes. This could be from a change of VLAN on the port, or in the case of a mobile client, maybe a connection to a WiFi network in a different location. Events like a network change or client reboot often streamline the DHCP process, with a client skipping to a `DHCPREQUEST` in an attempt to renew its old address. But what if that address is no longer valid on the new network the client has connected to?

{% include image.html url="/assets/images/2024-03-05-dhcp-option-82-and-vxlan/4-0_bad-dhcp-request.png" description="DHCP request sent in an attempt to renew current address, but the network reported in sub-option 5 has changed from the initial request." %}

According to the language in [RFC 3527](https://datatracker.ietf.org/doc/html/rfc3527), when this occurs "the server MUST NOT offer an address that is not on the requested subnet or the link (network segment) with which that subnet is associated."

But...

{% include image.html url="/assets/images/2024-03-05-dhcp-option-82-and-vxlan/4-1_bad-dhcp-ack.png" description="Windows DHCP Server ACKs the DHCPREQUEST message, even though the address is invalid on the current network segment." %}

It seems that the DHCP implementation on Windows Server 2022 does not behave in the way RFC 3527 designates. Thankfully, this issue was raised on the [Microsoft Q&A](https://learn.microsoft.com/en-us/answers/questions/432543/dhcp-server-ignores-link-selection-option-when-cli) site, and was answered with an undocumented registry key `DhcpFlagSubnetChangeDHCPRequest` that needs set to make the server behave according to the RFC. After adding that registry key per the linked discussion, let's retry.

{% include image.html url="/assets/images/2024-03-05-dhcp-option-82-and-vxlan/5_good-dhcp-ack.png" description="Windows DHCP Server behavior after DhcpFlagSubnetChangeDHCPRequest is set in the registry." %}

Nice, it's working! The server realizes that the `DHCPREQUEST` is for an address that will not be valid on the network segment specified in sub-option 5, and sends a `DHCPNAK`, prompting the client to restart the DHCP process. From there, everything works as expected, and the client receives an IP address that is valid for its network segment!

# Wrapping Up

Unfortunately, this seems to be an example of an implementation simply not working as it should, with a workaround configuration option added to nudge it back in the right direction without breaking existing behavior. If anyone knows why Microsoft would choose to implement this way and go against the RFC, please reach out as I am very curious to hear the reasoning! Also, there is currently an [issue](https://github.com/MicrosoftDocs/windowsserverdocs/issues/7525) open with MicrosoftDocs to add the aforementioned registry setting to the documentation, which may help alleviate further confusion as option 82 becomes more widely used.

To sanity check myself and my reading of the RFC, I tried the same setup as above but swapped out the DHCP server for one running Debian 12 with the package isc-dhcp-server version 4.4.3-P1-2 delivering addresses. With the default configuration, other than adding the scopes and address ranges, the server handled the network changes without issue, delivering a swift `NAK` when an address was requested that didn't match the network in sub-option 5. Looks like another win for Linux!