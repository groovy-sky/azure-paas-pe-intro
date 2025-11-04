### Azure cloud service models — quick comparison

| Model | What you get | Who manages infrastructure | Customization level | Typical use case |
|---|---:|---|---|---|
| **IaaS** | VMs, storage, virtual networking | Customer manages OS/apps; Azure manages hardware | High; full OS control | Lift-and-shift workloads, custom middleware |
| **PaaS** | Managed runtime and platform (app hosting, DB services) | Azure manages OS and platform; customer manages app code/config | Medium; app-level control but no OS | Web apps, APIs, managed databases |
| **SaaS** | Complete application delivered over Internet (e.g., Office 365) | Azure/vendor manages everything | Low; user-level settings only | Email, collaboration, line-of-business apps |

Key difference summary:
- Responsibility shifts left-to-right: IaaS needs most customer ops; SaaS needs least.
- Use IaaS when you need full control; PaaS to focus on code and speed; SaaS for ready-made applications.

---

### Azure Virtual Network (VNet) — core concepts

- **What a VNet is**  
  An isolated, private network in Azure where you place VMs and other resources. VNets are the unit of network isolation and routing in Azure.

- **VNet address space**  
  The CIDR block(s) assigned to a VNet (example: 10.0.0.0/16). All subnets must be subsets of this space and address ranges must not overlap with connected networks unless designed intentionally.

- **Subnets**  
  Logical subdivisions of the VNet address space (example: 10.0.1.0/24 for frontend, 10.0.2.0/24 for backend). Subnets let you apply network security groups (NSGs), route tables, and service endpoints or host private endpoints.

- **VNet peering**  
  A private, low-latency link between two VNets (same region or different regions) that routes traffic through Microsoft’s backbone. Peered VNets communicate using private IPs and do not require a gateway; optional transitive routing requires hub-and-spoke design with virtual appliances or route configuration.

---

### Private access to Azure PaaS — two main mechanisms

1. **Private Endpoint**
   - Provides a network interface (an ENI) in your VNet with a private IP address that represents a specific Azure PaaS resource (for example, an Azure SQL database, Storage account, or Key Vault).
   - The PaaS service’s FQDN resolves to that private IP (normally via a Private DNS Zone), so traffic from your VNet (and connected networks) goes directly over private IPs to the service backend inside Azure, never traversing the public internet.
   - Scope: resource-level; supports inbound and inbound-like connectivity patterns where service receives requests addressed to a private IP.

2. **VNet Integration**
   - Applies mainly to Azure App Service (web apps, functions) and some other PaaS features.
   - Allows the PaaS *client* (the app) to make outbound calls into a VNet (to VMs, private endpoints, on-premises resources). This does not assign the PaaS service a private IP for inbound client calls.
   - Two flavours: regional VNet Integration (for same region VNets) and gateway-required integration (for cross-region or older setups).
   - Scope: outbound connectivity from the app into the VNet; not intended to make the app itself reachable on a private IP.

Comparison and when to use each:
- Use **Private Endpoint** when you want the PaaS service itself to be reached privately (e.g., clients or on-prem services call an Azure SQL instance via private IP). It reduces exposure to public endpoints and supports fine-grained network controls.
- Use **VNet Integration** when the PaaS app needs to access resources inside your VNet or on-prem (e.g., the web app calls an internal database or API). If you need both (app reachable privately and app accessing VNet resources), combine Private Endpoint (for the service you want private) with VNet Integration (for the app’s outbound needs).
- Private Endpoint is about private inbound resolution to the service; VNet Integration is about the service’s outbound access into private networks.

---

### Making Private Endpoints accessible from on-premises networks

Required components:
- **Hybrid connectivity**  
  - VPN Gateway (site-to-site VPN) or ExpressRoute establish secure, private network links between your on-premises network and your Azure VNet. ExpressRoute offers higher bandwidth, lower latency, and optionally private peering through Microsoft.
- **DNS resolution**  
  - **Private DNS Zones**: When you create a Private Endpoint, Azure usually creates or links a Private DNS Zone (for example, privatelink.database.windows.net) containing A records that map the service’s FQDN to the private IP of the endpoint in the VNet.
  - **On-prem DNS visibility**: On-prem clients must resolve the PaaS service FQDN to the Azure private IP. Options:
    - Forward on-prem DNS queries for the PaaS domain to Azure (Azure DNS Private Resolver or a DNS forwarder VM) so they resolve against the Private DNS Zone.
    - Configure conditional forwarders on on-prem DNS servers to point queries for the privatelink domain to Azure DNS/IPs.
    - Use custom DNS servers in Azure (VM-based) and forward those from on-prem.
  - **Private DNS Resolver (Azure DNS Private Resolver)**: A managed DNS service that can accept DNS queries from on-prem via inbound endpoints (or conditional forwarding) and resolve names in Azure Private DNS zones, avoiding the need for public DNS exposure.
- **Network path and security**  
  - Route on-prem traffic over the VPN/ExpressRoute to the VNet containing the Private Endpoint. Ensure there are no conflicting routes or overlapping CIDRs that prevent routing to the private IP.
  - Apply NSGs and firewall rules to allow expected ports and source networks. If you use a network virtual appliance (NVA) or Azure Firewall in a hub VNet, ensure proper UDRs (user-defined routes) and peering propagate traffic correctly.

How the pieces work together (sequence):
1. Create Private Endpoint in Azure VNet; Private DNS Zone gets A record mapping service FQDN to private IP.
2. Establish VPN Gateway or ExpressRoute connection between on-prem and Azure VNet.
3. Ensure on-prem DNS queries for the service FQDN are forwarded to Azure DNS Private Resolver or an Azure-based DNS server that can answer from the Private DNS Zone.
4. On-prem client resolves service FQDN → receives Azure private IP.
5. Client opens connection to the private IP; packets traverse the VPN/ExpressRoute to Azure VNet and reach the Private Endpoint, which connects to the PaaS backend internally.

Notes and gotchas:
- DNS is the most frequent failure point; if FQDN still resolves to public IP, traffic will try the public path and may be blocked or misrouted.
- Avoid overlapping IP ranges between on-prem and Azure VNets; if overlap exists, name/route workarounds are required.
- For multi-region or central-hub designs, consider deploying Private Endpoints in a peered hub VNet and use hub DNS/resolver patterns.
- Some PaaS services also allow service endpoints or regional VNet service endpoints; these are older patterns and differ from Private Endpoints (service endpoints keep the service public but allow inbound policy-based control; private endpoints map to private IPs).

---

### High-level architecture description and DNS flow

Architecture components:
- On-premises network (clients, on-prem DNS)
- Hybrid link: VPN Gateway or ExpressRoute (connects on-prem to Azure virtual network)
- Azure Hub VNet (optionally) with Azure Firewall/NVA, DNS resolver inbound endpoint
- Azure Spoke VNet containing the Private Endpoint (or Private Endpoint in hub if appropriate)
- Azure PaaS service (backed by Private Endpoint)
- Private DNS Zone for the PaaS FQDN (linked to VNets hosting endpoints or resolvers)

DNS and traffic flow (simple step list):
1. On-prem client queries for mydb.database.windows.net.
2. On-prem DNS server forwards queries for *.database.windows.net (or privatelink subdomain) to Azure DNS Private Resolver or to a conditional forwarder that points to an Azure-based DNS VM.
3. Azure resolver answers with the Private Endpoint’s private IP from the Private DNS Zone.
4. Client opens TCP connection to that private IP; network traffic goes across VPN/ExpressRoute to the VNet.
5. The Private Endpoint receives traffic and forwards to the PaaS service backend over Azure private infrastructure.
6. Response returns the same way; no public internet traversal.

