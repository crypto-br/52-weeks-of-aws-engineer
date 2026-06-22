# Session 28 — NLB and GLB: use cases, preserve client IP and inline inspection

**Estimated duration:** 60 minutes
**Prerequisites:** session-027-alb-oidc-mtls-waf

---

## Objective

By the end, you will be able to explain the cases where NLB is necessary (TCP/UDP, preserve client IP,
static IP, PrivateLink), configure a Gateway Load Balancer to route traffic through inline
inspection appliances (IDS/IPS/firewall), and design the correct topology of a VPC with GLB for
North-South traffic inspection.

---

## Context

[FACT] The ELB family has four members: ALB (L7/HTTP), NLB (L4/TCP-UDP), GLB (L3/IP) and CLB
(legacy). Each one solves a different problem — choosing the wrong one is not just inefficient, it's
technically unfeasible in certain cases. The ALB cannot route pure TCP traffic (such as databases,
gRPC without HTTP/2 wrapper, or non-HTTP binary protocols). The NLB does not perform L7 content
inspection. The GLB does not route directly to applications — it is exclusively an inspection "bumper".

[CONSENSUS] The trend in recent years is to use NLB as backing for PrivateLink endpoints to
expose internal services securely between accounts and VPCs, and to use GLB as a centralized
inspection layer in hub-and-spoke architectures with Transit Gateway. Understanding NLB and GLB is a
prerequisite for AWS network architectures beyond the basics.

---

## Main concepts

### 1. Network Load Balancer — characteristics and operating model

[FACT] The NLB operates at **layer 4 (transport) of the OSI model**. It does not read or modify HTTP
content — it only forwards TCP, UDP or TLS connections based on IP addresses and ports. By operating at
L4, the NLB has extremely low latency (typically sub-millisecond) and can scale to
**millions of requests per second** without pre-warming.

[FACT] Protocols supported per NLB target group:

```
TCP       → pure TCP connection; flow routed for the lifetime of the connection
TLS       → NLB performs TLS termination (or passthrough) at L4
UDP       → no handshake; flow identified by 5-tuple
TCP_UDP   → target group accepts both on the same port (e.g., DNS port 53)
QUIC      → uses Connection ID in CID for routing; fallback to flow hash
TCP_QUIC  → target group accepts TCP and QUIC on the same port
```

**Routing algorithm by protocol:**

```
TCP / TLS:
  Flow hash: protocol + src IP + src port + dst IP + dst port + TCP sequence number
  → The same TCP connection is always routed to the same target during its entire lifetime
  → Different connections from the same client (different src port) MAY go to different targets

UDP:
  Flow hash: protocol + src IP + src port + dst IP + dst port
  → No TCP sequence number (UDP is connectionless)
  → A UDP flow is consistent to the same target for the lifetime of the flow

QUIC:
  → Uses Server ID in Connection ID (CID) after establishment
  → Allows network migration (QUIC connection persists even when changing client IP/port)
```

[FACT] For each enabled Availability Zone, the NLB creates a **network interface** with a static
IP address. For internet-facing load balancers, it is possible to associate an **Elastic IP** per AZ,
making the NLB's IP address completely predictable and fixed —
a characteristic impossible with ALB (which uses dynamic IPs).

### 2. NLB use cases — when ALB is not enough

The five scenarios where NLB is the correct or only choice:

```
┌─────────────────────────────────────────────────────────────────────┐
│ 1. NON-HTTP PROTOCOLS                                                │
│    Database (PostgreSQL 5432, MySQL 3306), Redis 6379,               │
│    MQTT (IoT), gRPC with non-standard HTTP/2, binary protocols      │
│    → ALB cannot inspect or route these protocols                     │
├─────────────────────────────────────────────────────────────────────┤
│ 2. PRESERVE CLIENT IP                                                │
│    Applications that need to see the real client IP (firewalls,      │
│    geolocation, auditing, rate limiting by IP on the backend)        │
│    → NLB can preserve the source IP; ALB always uses proxy           │
├─────────────────────────────────────────────────────────────────────┤
│ 3. STATIC IP / ELASTIC IP                                            │
│    Clients with IP whitelist, on-premises integration that           │
│    does not accept FQDN, compliance requiring fixed IP               │
│    → NLB supports Elastic IP per AZ; ALB uses dynamic IPs           │
├─────────────────────────────────────────────────────────────────────┤
│ 4. AWS PRIVATELINK (endpoint service)                                │
│    Expose internal services to other accounts/VPCs via endpoint      │
│    → PrivateLink requires NLB as backing; does not support ALB       │
├─────────────────────────────────────────────────────────────────────┤
│ 5. ULTRA-LOW LATENCY / EXTREME THROUGHPUT                            │
│    Trading, IoT telemetry, high-volume streaming                     │
│    → NLB does not add L7 overhead; ALB has additional processing     │
└─────────────────────────────────────────────────────────────────────┘
```

### 3. Preserve Client IP — behavior by target type

[FACT] The preserve client IP behavior varies according to the **target type** of the target group
and the **protocol**:

```
Target type INSTANCE:
  TCP / TLS    → client IP preservation ENABLED by default (can be disabled)
  UDP / QUIC   → client IP preservation ENABLED by default (CANNOT be disabled)

Target type IP:
  TCP / TLS    → client IP preservation DISABLED by default (can be enabled)
  UDP / QUIC   → client IP preservation ENABLED by default (CANNOT be disabled)
```

[FACT] When client IP preservation is **enabled**, the NLB does IP passthrough: the packet that
arrives at the target has the source IP equal to the real client IP. The target "sees" the client directly.

When it is **disabled**, the src IP of the packet is the **private IP address of the NLB node** in the AZ.
The target has no visibility of the real client IP.

**Cases where preserve client IP does NOT work (restrictions):**

```
1. Targets reached via Transit Gateway          → not supported
2. Traffic originated from AWS PrivateLink      → src IP is always the NLB IP
   (PrivateLink replaces the client IP with the NLB IP — by design)
3. Target type = ALB (NLB in front of ALB)      → ALB uses X-Forwarded-For for client IP
4. Cross-zone routing to target in another AZ   → may work, but creates flow asymmetry
   when the target responds back without going through the NLB
```

[FACT] A classic pitfall: if client IP preservation is enabled and the **security group**
of the target does not allow traffic from the client IP (only from the NLB), connections fail without
a clear error message. The NLB does not have its own security group — it passes traffic transparently
when IP preservation is on.

**Cross-zone load balancing on NLB:**

[FACT] On ALB, cross-zone load balancing is **enabled by default** and there is no additional cost.
On NLB, it is **disabled by default** — and when enabled, it charges for **data transfer between
AZs** (different from ALB). The disabled default ensures that each NLB node distributes traffic only
to targets in the same AZ, avoiding additional latency and inter-AZ costs.

### 4. NLB and PrivateLink — exposing services between accounts

[FACT] AWS PrivateLink allows exposing an internal service from one account/VPC to consumers in
other accounts/VPCs without needing peering, without routing traffic through the internet, and without exposing the
entire VPC CIDR. The backing of a PrivateLink endpoint service is **mandatorily an NLB** (GLB
is also supported for appliances, but not ALB).

```
Producer Account (Service Provider)                   Consumer Account (Service Consumer)
┌────────────────────────────────────────┐            ┌──────────────────────────────────────┐
│                                        │            │                                      │
│  Targets (EC2, ECS, Lambda)            │            │  Consumer makes requests to           │
│       ↑                                │            │  vpce-xxxxx.vpce-svc-yyyy.           │
│  NLB (backing of endpoint service)     │◄───────────│  us-east-1.vpce.amazonaws.com        │
│       ↑                                │  PrivateLink│       ↑                             │
│  VPC Endpoint Service                  │  (ENI)     │  VPC Endpoint (Interface)            │
│  (vpce-svc-xxxxxxxx)                   │            │  → creates ENI in consumer subnet    │
└────────────────────────────────────────┘            └──────────────────────────────────────┘
```

[FACT] PrivateLink traffic always presents the NLB IP as src IP at the target — the real
consumer IP is lost. This is a PrivateLink design limitation, not an NLB configuration.

### 5. Gateway Load Balancer — architecture and GENEVE protocol

[FACT] The GLB operates at **layer 3 (network) of the OSI model**. It does not terminate TCP connections nor read
transport headers — it simply receives *all* IP packets on all ports and forwards them to security appliances (firewalls, IDS/IPS, DPI). The appliance inspects the
packet and returns it to the GLB, which then forwards it to the original destination.

[FACT] Communication between the GLB and appliances uses the **GENEVE (Generic Network
Virtualization Encapsulation)** protocol on UDP port **6081**. The GLB encapsulates the original client
packet inside a GENEVE envelope and sends it to the appliance. The appliance processes, maintains
(or modifies) the content, and returns it with the same GENEVE envelope. The GLB decapsulates and
forwards to the destination.

```
Original packet:
  [ IP header: src=203.0.113.1, dst=10.0.1.5 ][ TCP ][ HTTP payload ]

Encapsulated with GENEVE (sent to the appliance):
  [ IP header: src=NLB-IP, dst=appliance-IP ][ UDP 6081 ][ GENEVE header ]
  [ IP header: src=203.0.113.1, dst=10.0.1.5 ][ TCP ][ HTTP payload ]
                ↑ original packet preserved entirely inside the envelope
```

[FACT] The **GENEVE header** contains metadata that identifies the flow (TLVs — Type-Length-Value),
including the ARN of the GLB endpoint. This allows multi-tenant appliances to know from which endpoint
the traffic came.

**Flow stickiness on GLB:**

[FACT] The GLB maintains flow stickiness to ensure that packets going to and from the same
connection pass through the same appliance (essential for stateful appliances like firewalls). The
algorithm can be configured:

```
5-tuple (default): src IP + src port + dst IP + dst port + protocol
3-tuple:           src IP + dst IP + protocol
2-tuple:           src IP + dst IP
```

The less granular the tuple, the more traffic is grouped on the same appliance — useful when the
appliance needs to correlate traffic from multiple flows of the same IP pair.

### 6. GLB topology — North-South inspection

"North-South" refers to traffic that enters or leaves the VPC through the Internet Gateway (IGW).
The GLB topology for North-South inspection uses two VPCs and **ingress routing** on the IGW's
route table.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          SERVICE CONSUMER VPC                                │
│                                                                              │
│  ┌──────────────────────┐           ┌──────────────────────────────────┐    │
│  │  IGW Subnet          │           │  GWLB Endpoint (GWLBe) Subnet    │    │
│  │  (Ingress routing)   │           │  Route table:                    │    │
│  │                      │           │    0.0.0.0/0 → IGW               │    │
│  │  Route table:        │           └────────────┬─────────────────────┘    │
│  │  10.0.1.0/24 → GWLBe │                        │                         │
│  └──────────┬───────────┘      ┌──────────────────┘                        │
│             │                  │                  ┌──────────────────────┐  │
│  ┌──────────▼───────────┐      │                  │  App Subnet          │  │
│  │  Internet Gateway    │      │                  │  (Application)       │  │
│  └──────────────────────┘      │                  │  Route table:        │  │
│                                │                  │    0.0.0.0/0 → GWLBe │  │
│                                │                  └──────────────────────┘  │
└────────────────────────────────┼────────────────────────────────────────────┘
                                  │ PrivateLink
┌─────────────────────────────────┼────────────────────────────────────────────┐
│                        SERVICE PROVIDER VPC                                   │
│                                 │                                             │
│               ┌─────────────────▼──────────────────┐                         │
│               │     Gateway Load Balancer (GLB)     │                         │
│               │     (GENEVE, port 6081, all ports)  │                         │
│               └─────────────┬──────────────────────┘                         │
│                             │  GENEVE encapsulation                           │
│               ┌─────────────▼──────────────────────┐                         │
│               │   Security Appliances (EC2/Fargate) │                         │
│               │   (firewall / IDS / IPS / DPI)      │                         │
│               │   UDP 6081 must be open             │                         │
│               └────────────────────────────────────┘                         │
└───────────────────────────────────────────────────────────────────────────────┘
```

**Inbound traffic flow (Internet → App):**

```
1. Packet enters through the Internet Gateway (IGW)
2. Ingress routing on the IGW route table:
   destination 10.0.1.0/24 (app subnet) → target: GWLBe (vpc-endpoint-id)
3. GWLBe forwards to the GLB in the Service Provider VPC (via PrivateLink)
4. GLB encapsulates with GENEVE and sends to an appliance
5. Appliance inspects, approves/blocks, returns to GLB
6. GLB decapsulates and returns to GWLBe
7. GWLBe forwards to app subnet (local route)
```

**Outbound traffic flow (App → Internet):**

```
1. Packet leaves the app
2. App subnet route table: 0.0.0.0/0 → GWLBe
3. GWLBe → GLB → appliance (inspects) → GLB → GWLBe
4. GWLBe subnet route table: 0.0.0.0/0 → IGW
5. IGW → Internet
```

[FACT] **Ingress routing** (route table associated with the IGW) is a specific resource for this
pattern — it allows the IGW to route *inbound* traffic to an endpoint before reaching the destination
subnet. Without this, it is not possible to intercept North-South traffic transparently.

[FACT] The GWLB endpoint is **zonal** — one per AZ. AWS recommends creating one GWLBe per AZ to
ensure that the inspection flow is intra-AZ (avoiding latency and cross-AZ cost).

---

## Practical example

**Scenario:** fintech company needs:
1. NLB fronting a TCP payment processing service (proprietary protocol, port 8443)
   with static IP for partner whitelisting
2. GLB inspecting ALL traffic entering the VPC before reaching applications

### CDK Python — NLB with static IP and preserve client IP

```python
from aws_cdk import (
    Stack, Duration,
    aws_ec2 as ec2,
    aws_elasticloadbalancingv2 as elbv2,
)
from constructs import Construct

class PaymentNlbStack(Stack):
    def __init__(self, scope: Construct, id: str, **kwargs):
        super().__init__(scope, id, **kwargs)

        vpc = ec2.Vpc(self, "Vpc", max_azs=2)

        # ── Elastic IPs for NLB static IPs ──────────────────────────────
        eip_az1 = ec2.CfnEIP(self, "EipAz1", domain="vpc")
        eip_az2 = ec2.CfnEIP(self, "EipAz2", domain="vpc")

        # ── Internet-facing NLB with Elastic IPs ────────────────────────
        nlb = elbv2.NetworkLoadBalancer(self, "PaymentNLB",
            vpc=vpc,
            internet_facing=True,
            # Elastic IPs associated per AZ via subnet mappings
            # (via L1 CfnLoadBalancer when CDK L2 does not directly support)
        )

        # ── TCP Target Group with preserve client IP enabled ────────────
        tg = elbv2.NetworkTargetGroup(self, "PaymentTG",
            vpc=vpc,
            port=8443,
            protocol=elbv2.Protocol.TCP,
            target_type=elbv2.TargetType.IP,   # IP target type
            # For IP target type + TCP, client IP is DISABLED by default
            # To enable, use preserve_client_ip=True
            health_check=elbv2.HealthCheck(
                protocol=elbv2.Protocol.TCP,
                interval=Duration.seconds(10),
                healthy_threshold_count=2,
                unhealthy_threshold_count=2,
            ),
        )

        # Enable preserve client IP on the target group (via L1 escape hatch)
        cfn_tg = tg.node.default_child
        cfn_tg.add_property_override("TargetGroupAttributes", [
            {"Key": "preserve_client_ip.enabled", "Value": "true"}
        ])

        # ── TCP 8443 Listener ───────────────────────────────────────────
        nlb.add_listener("TcpListener",
            port=8443,
            protocol=elbv2.Protocol.TCP,
            default_target_groups=[tg],
        )

        # ── Target security group — allows client IP directly ───────────
        # (not the NLB IP, because client IP preservation is ON)
        target_sg = ec2.SecurityGroup(self, "TargetSG", vpc=vpc)
        target_sg.add_ingress_rule(
            ec2.Peer.ipv4("0.0.0.0/0"),  # in production: partner CIDRs
            ec2.Port.tcp(8443),
            "Allow payment clients direct access (NLB client IP preservation)",
        )
```

### CDK Python — GLB for North-South inspection (L1 CfnResources)

```python
import aws_cdk as cdk
from aws_cdk import (
    Stack,
    aws_ec2 as ec2,
    aws_elasticloadbalancingv2 as elbv2,
)
from constructs import Construct

class InspectionStack(Stack):
    def __init__(self, scope: Construct, id: str, **kwargs):
        super().__init__(scope, id, **kwargs)

        # ── Service Provider VPC (appliances) ──────────────────────────
        provider_vpc = ec2.Vpc(self, "ProviderVpc",
            max_azs=2,
            subnet_configuration=[
                ec2.SubnetConfiguration(
                    name="Appliance",
                    subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS,
                )
            ],
        )

        # ── Security group for appliances (must accept UDP 6081) ───────
        appliance_sg = ec2.SecurityGroup(self, "ApplianceSG", vpc=provider_vpc)
        appliance_sg.add_ingress_rule(
            ec2.Peer.any_ipv4(),
            ec2.Port.udp(6081),
            "GENEVE protocol from GLB",
        )

        # ── GENEVE Target Group for the GLB ─────────────────────────────
        # (L1 because CDK L2 does not have full GLB support)
        glb_tg = elbv2.CfnTargetGroup(self, "ApplianceTG",
            name="inspection-appliances",
            protocol="GENEVE",
            port=6081,
            vpc_id=provider_vpc.vpc_id,
            target_type="instance",
            health_check_protocol="HTTP",
            health_check_port="80",
            health_check_path="/health",
        )

        # ── Gateway Load Balancer ───────────────────────────────────────
        glb = elbv2.CfnLoadBalancer(self, "GLB",
            name="inspection-glb",
            type="gateway",
            subnets=[s.subnet_id for s in provider_vpc.private_subnets],
        )

        # GLB Listener (listens on all IPs on all ports)
        elbv2.CfnListener(self, "GlbListener",
            load_balancer_arn=glb.ref,
            default_actions=[{
                "type": "forward",
                "targetGroupArn": glb_tg.ref,
            }],
        )

        # ── VPC Endpoint Service (PrivateLink backing) ──────────────────
        endpoint_service = ec2.CfnVPCEndpointService(self, "EndpointService",
            gateway_load_balancer_arns=[glb.ref],
            acceptance_required=False,
        )
```

### CLI — essential commands

**Check if client IP preservation is enabled on the TG:**
```bash
aws elbv2 describe-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/payment-tg/... \
  --query 'Attributes[?Key==`preserve_client_ip.enabled`]'
```

**Enable cross-zone load balancing on the NLB:**
```bash
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn arn:aws:elasticloadbalancing:... \
  --attributes Key=load_balancing.cross_zone.enabled,Value=true
```

**Create GLB via CLI and register appliance:**
```bash
# Create GLB
aws elbv2 create-load-balancer \
  --name inspection-glb \
  --type gateway \
  --subnets subnet-aaaa subnet-bbbb

# Create GENEVE target group
aws elbv2 create-target-group \
  --name inspection-appliances \
  --protocol GENEVE \
  --port 6081 \
  --vpc-id vpc-xxxx \
  --target-type instance \
  --health-check-protocol HTTP \
  --health-check-port 80 \
  --health-check-path /health

# Register appliance in TG
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/inspection-appliances/... \
  --targets Id=i-0123456789abcdef0

# Create GLB listener
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:...:loadbalancer/gwy/inspection-glb/... \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...

# Create endpoint service
aws ec2 create-vpc-endpoint-service-configuration \
  --gateway-load-balancer-arns arn:aws:elasticloadbalancing:...:loadbalancer/gwy/inspection-glb/... \
  --no-acceptance-required

# Create GWLB endpoint in consumer VPC
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-consumer \
  --service-name com.amazonaws.vpce.us-east-1.vpce-svc-xxxx \
  --vpc-endpoint-type GatewayLoadBalancer \
  --subnet-ids subnet-endpoint-az1

# Configure ingress routing: IGW → GWLBe for traffic destined to app subnet
aws ec2 create-route \
  --route-table-id rtb-igw-xxxxx \
  --destination-cidr-block 10.0.1.0/24 \
  --vpc-endpoint-id vpce-yyyy

# Configure default route on app subnet → GWLBe (outbound traffic)
aws ec2 create-route \
  --route-table-id rtb-app-xxxxx \
  --destination-cidr-block 0.0.0.0/0 \
  --vpc-endpoint-id vpce-yyyy
```

---

## Common pitfalls

### Pitfall 1: security group on target does not allow client IP when preserve client IP is ON

When client IP preservation is enabled on the NLB, the target receives the packet with the src IP equal
to the real client IP — not the NLB IP. If the target's security group was configured to accept
traffic only from the NLB (or from the NLB's CIDR), all connections fail silently. The NLB does
not have its own security group, so there is no "Allow from NLB security group" possible as with ALB.
The diagnostic is: health checks pass (because they originate from the NLB node IP), but real traffic fails.
Check via CloudWatch `TCP_ELB_Reset_Count` — if high, it usually indicates rejection by the target
or security group.

### Pitfall 2: cross-zone on NLB has cost and is off by default

On ALB, cross-zone load balancing is on by default and free. On NLB, it is off by default and
**charges for data transfer between AZs** when enabled. Teams that migrate from ALB to NLB and
enable cross-zone without attention discover surprises on the bill. Additionally, with cross-zone off and
client IP preservation on, if a client whose IP is routed to the NLB node in AZ-A (by DNS
or Elastic IP) but all healthy targets are in AZ-B, requests fail — the NLB does not
do cross-zone and has no available targets in AZ-A.

### Pitfall 3: route asymmetry in GLB topology breaks inspection

In the GLB topology, inbound and outbound traffic from the same flow must pass through the **same
appliance** (stateful firewalls maintain connection state). If the route tables are
incorrect — for example, the return route for traffic leaving the app does not go through the GWLBe — the
flow becomes asymmetric: the appliance sees only half of the conversation and drops packets (or
"resets" the connection). The diagnostic: `nc` or `curl` to the target works but the connection is
dropped shortly after; the appliance logs show packets without a corresponding state. The
fix is to ensure that the app subnet's route table has `0.0.0.0/0 → GWLBe` to force
all outbound traffic through the endpoint as well.

---

## Reflection exercise

You work at a company that needs to expose an internal service (transaction authentication,
proprietary TCP protocol, port 9443) to 50 external partners. Each partner needs to use the
service IP in a whitelist — FQDN is not accepted. Additionally, you need to ensure that the
backend sees the real source IP of each partner for auditing and geolocation purposes.

Describe the architecture you would build: which type of load balancer, how to configure the Elastic
IPs, how to handle preserve client IP (including the implications on the target's security group),
and how you would expose this service to partners in an isolated manner (without exposing the entire VPC or creating
peering with each of the 50 partners). Also consider what happens with health checks
when client IP preservation is enabled — where do the NLB health checks come from and what does this
mean for the security group configuration.

---

## Resources for further study

**1. What is a Network Load Balancer?**
URL: https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html
What you'll find: overview of components, flow hash algorithms by protocol
(TCP vs UDP vs QUIC), differences between cross-zone on/off, and reference for client IP preservation.
Why it's the right source: primary documentation with exact NLB behavior specifications.

**2. Edit target group attributes for your Network Load Balancer**
URL: https://docs.aws.amazon.com/elasticloadbalancing/latest/network/edit-target-group-attributes.html
What you'll find: complete table of default client IP preservation behavior by
target type and protocol, how to enable/disable, and limitations (Transit Gateway, PrivateLink).
Why it's the right source: the defaults table is the canonical reference to avoid the most
common NLB pitfall.

**3. What is a Gateway Load Balancer?**
URL: https://docs.aws.amazon.com/elasticloadbalancing/latest/gateway/introduction.html
What you'll find: GLB overview, GENEVE protocol, flow stickiness, and VPC
endpoints model for exchanging traffic between VPCs.
Why it's the right source: primary documentation that explains the conceptual model and relationships
between GLB, GWLB endpoint and PrivateLink.

**4. Getting started with Gateway Load Balancers**
URL: https://docs.aws.amazon.com/elasticloadbalancing/latest/gateway/getting-started.html
What you'll find: step-by-step of the complete North-South topology — creating the GLB,
endpoint service, GWLB endpoint, and the three necessary route tables (IGW, app subnet,
GWLBe subnet) with exact route entry examples.
Why it's the right source: the only place in official documentation that shows all three route tables
together and explains traffic direction with numbered diagram.

**5. Inspecting inbound traffic from the internet using firewall appliances with Gateway Load Balancer**
URL: https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/inspecting-inbound-traffic-fa.html
What you'll find: whitepaper with advanced North-South and East-West inspection topologies
using GLB with Transit Gateway for hub-and-spoke architectures.
Why it's the right source: goes beyond the basic case and shows how to scale inspection for multiple
centralized VPCs, which is the real use case in enterprises.

---
