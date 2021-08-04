# ExpressRoute migration guide

# Contents

- [Introduction](#introduction)
- [Option 1 – Code the complexity](#option-1---code-the-complexity)
- [Option 2 - 3rd party GSLB Network Virtual Appliance (NVA)](#option-2---3rd-party-gslb-network-virtual-appliance--nva-)
- [Option 3 - Azure Traffic Manager](#option-3---azure-traffic-manager)
- [Option 4 - Azure Front Door](#option-4---azure-front-door)
- [Option 5 - Azure Load Balancer](#option-5---azure-load-balancer)
- [Option 6 - Application Gateway](#option-6---application-gateway)
- [Closing](#closing)

# Introduction

Many customers on Azure leverage ExpressRoute for reliable hybrid connectivity. Sometimes these same customers may need to implement a new ExpressRoute circuit, and have a desire to decommission their old circuit. Common drivers for this requirement include:

- Mergers and acquisitions resulting on consolidation of network providers
- A change of ExpressRoute provider
- Downgrading an ER circuit bandwidth requires a new circuit https://docs.microsoft.com/en-us/azure/expressroute/expressroute-faqs#can-i-change-the-bandwidth-of-an-expressroute-circuit
- Upgrading an ER circuit can sometimes require a new circuit, depending if the existing underlying port has capacity to support the bandwidth increase https://docs.microsoft.com/en-us/azure/expressroute/expressroute-faqs#can-i-change-the-bandwidth-of-an-expressroute-circuit

This guide provides step-by-step instructions on how to approach this circuit migration process.

# Context

When having any ExpressRoute discussion its beneficial to agree on terms. This is best visualized in layers, highlighting the different components in any ExpressRoute design; 

- Gateway
- Connection
- Circuit
- Partner
- Customer

<image>

# Existing configuration

Before removing the old connectivity, we should capture its logical configuration and form a baseline diagram to iterate from. Important information shown in the example diagram below includes;

- Which Azure regions are being used, E.g. West Europe
- ExpressRoute Gateway SKU, e.g. "high", or "ERGw1Az" https://docs.microsoft.com/en-us/azure/expressroute/expressroute-about-virtual-network-gateways#gwsku
- ExpressRoute connection objects (the things that link a _gateway_ to a _circuit_ within the same subscriptions)
- ExpressRoute authorisations (the things that link a _gateway_ to a _circuit_ within a different subscription) https://docs.microsoft.com/en-us/azure/expressroute/expressroute-howto-linkvnet-arm#circuit-owner-operations
- ExpressRoute circuit attributes
  - Peering location, e.g. London1
  - Bandwidth, e.g. 1Gbps
  - Provider, e.g. Equinix
  - SKU, e.g. Standard/Premium/Local
  - Billing Model, e.g. Unlimited/Metered
- ExpressRoute circuit peering types
  - Private peering *(the majority of customers will only be using this peering type)*
  - Public Peering (now deprecated, but may still be in use in your network)
    - Public Peering utilises Microsoft managed SNAT ranges. You will need to raise a support ticket to find out your SNAT address. https://docs.microsoft.com/en-us/azure/expressroute/expressroute-nat#nat-requirements-for-azure-public-peering
  - Microsoft Peering
    - Microsoft peering utilises customer managed SNAT, you or your provider will have this information. https://docs.microsoft.com/en-us/azure/expressroute/expressroute-nat#nat-requirements-for-microsoft-peering
    - Route filters associated with Microsoft peering https://docs.microsoft.com/en-us/azure/expressroute/how-to-routefilter-portal
- BGP AS numbers and /30 peer IP addresses for all peerings https://docs.microsoft.com/en-us/azure/expressroute/expressroute-routing

Once this is complete, you should be able to produce some diagrams like the following examples, I have split out the Private Peering and the Public Peering in to separate diagrams for clarity. The remainder of the document will follow this structure, as these peering types have marked different considerations.

<image>
<image>

# Private Peering Migration

## Create new ER circuit

https://docs.microsoft.com/en-us/azure/expressroute/expressroute-howto-circuit-portal-resource-manager

Follow the above guide to create the new circuit with your requirement parameters. This includes working with your provider to turn-up the new circuit and configure the BGP peerings to your network. The /30 BGP links will utilise new, unique IP addressing.

<image>

## Create test Virtual Network and link to circuit

https://docs.microsoft.com/en-us/azure/expressroute/expressroute-howto-linkvnet-arm

Create a new Virtual Network with some test IP address space that does not overlap with either your current On-Premises address space, or your existing Azure VNet(s) address space. Deploy a Virtual Machine for testing. The purposes of this step is to prove out end-to-end connectivity across the new circuit, prior to using it within your production environment. If done correctly, your On-Premises network can now route traffic to this new test VNet, and vice versa. Carry out your required commissioning tests, latency, throughput etc

<image>

> Take this opportunity to become familiar with all the rich ExpressRoute information that is available from the CLI. For example, a good idea at this step is to verify you are advertising all the required routes from On-Premises with the expected AS-PATH manipulation, if any. A great guide to get started https://blog.cloudtrooper.net/2021/07/12/cli-based-analysis-of-an-expressroute-private-peering/

## Prep circuit authorsations

If you are using circuit authorsations for cross-subscription gateway attachment (you checked that already right? ;) Then you are able to prep these ahead of time on the new circuit. Generate the new authorsations and they will remain in the "available" state until redeemed at the gateway level. One less job to do during the migration window itself. 

![](images/2021-08-04-13-24-58.png)

## Migration options

At this point you have verified the new circuit is ready for production workloads, now you have to decide on your migration approach;

6666-11



### Delete and attach; more downtime but simple

- Delete the connection objects that links your old circuit to your ER Gateways
- As soon as you initiate this process, you will lose connectivity via the ER circuit
- Create a new connection object that links your new circuit to your ER Gateways

Plan for 10-30 minutes change window.

If you have multiple existing ER circuits

### Attach and influence, less downtime but more complex


# Public/Microsoft Peering Migration

https://docs.microsoft.com/en-us/azure/expressroute/how-to-move-peering



# Closing

Despite this document not proposing an entirely elegant solution, I do hope it helps you approach the topic and understand the current options and constraints.

PS. Hat tip to new feature in the portal that can kick start your decision-making process in this area. https://docs.microsoft.com/en-us/azure/architecture/guide/technology-choices/load-balancing-overview#choose-a-load-balancing-solution-using-azure-portal
