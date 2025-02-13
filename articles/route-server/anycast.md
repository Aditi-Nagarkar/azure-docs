---
title: 'Propagating anycast routes to on-premises'
description: Learn about how to advertise the same route from different regions with Azure Route Server.
services: route-server
author: halkazwini
ms.service: route-server
ms.topic: conceptual
ms.date: 02/03/2022
ms.author: halkazwini
---

# Anycast routing with Azure Route Server

Although spreading an application across Availability Zones in a single Azure region will result in a higher availability, often times applications need to be deployed in multiple regions, either to achieve a higher resiliency, a better performance for users across the globe, or better business continuity. There are different approaches that can be taken to direct users to one of the locations where a multi-region application is deployed to: DNS-based approaches such as [Azure Traffic Manager](../traffic-manager/traffic-manager-overview.md) or routing-based services like [Azure Front Door](../frontdoor/front-door-overview.md) or the [Azure Cross-Regional Load Balancer](../load-balancer/cross-region-overview.md).

The aforementioned services are recommended for getting users to the best application location over the public Internet using public IP addressing, but they don't support private networks and IP addresses. This article will explore the usage of a route-based approach (IP anycast) to provide multi-regional, private-networked application deployments.

IP anycast essentially consists of advertising exactly the same IP address from more than one location, so that packets from the application users are routed to the closest (in terms of routing) region. Providing multi-region reachability over anycast offers some advantages over DNS-based approaches, such as not having to rely on clients not caching their DNS answers, and not requiring to modify the DNS design for the application.

## Topology

In the design covered in this scenario, the same IP address will be advertised from VNets in different Azure regions, where NVAs will advertise the application's IP address through Azure Route Server. The following diagram depicts two simple hub and spoke topologies, each in a different Azure region. A Network Virtual Appliance (NVA) in each region advertises the same route (`a.b.c.d/32` in this example, it could be any prefix that ideally does not overlap with the Azure and on-premises networks) to its local Azure Route Server. The routes will be further propagated to the on-premises network through ExpressRoute. When application users want to access the application from on-premises, the DNS infrastructure (not covered by this document) will resolve the DNS name of the application to the anycast IP address (`a.b.c.d` in the diagram), which the on-premises network devices will route to one of the two regions.

:::image type="content" source="media/scenarios/anycast.png" alt-text="Diagram of anycast with Route Server.":::

The decision of which of the available regions is selected is entirely based on routing attributes. If the routes from both regions are identical, the on-premises network will typically use Equal Cost MultiPathing (ECMP) to send each application flow to each region. It is possible as well to modify the advertisements generated by each NVA in Azure to make one of the regions preferred, for example with BGP AS Path prepending, establishing a deterministic path from on-premises to the azure workload.

> [!IMPORTANT]
> The NVAs advertising the routes should include some health check mechanism to stop advertising the route when the application is not available in their respective regions, to avoid blackholing traffic.

## Return traffic

When the application traffic from the on-premises client arrives to one of the NVAs in Azure, the NVA will either reverse-proxy the connection or perform Destination Network Address Translation (DNAT). It will then send the packets to the actual application, which will typically reside in a spoke VNet peered to the hub VNet where the NVA is deployed. Traffic back from the application needs to go back through the NVA, which would happen naturally if the NVA is reverse-proxying the connection (or performs Source NAT additionally to Destination NAT).

Otherwise, traffic arriving to the application will still be sourced from the original on-premises client's IP address. In this case, packets can be routed back to the NVA with User-Defined Routes. Special care needs to be taken if there are more than one NVA instance in each region, since traffic could be asymmetric (the inbound and outbound traffic going through different NVA instances). Asymmetric traffic is typically not an issue if NVAs are stateless, but it will result in errors if the NVAs instead need to keep track of connection states, such as firewalls. 

## Next steps

* [Learn how Azure Route Server works with ExpressRoute](expressroute-vpn-support.md)
* [Learn how Azure Route Server works with a network virtual appliance](resource-manager-template-samples.md)
