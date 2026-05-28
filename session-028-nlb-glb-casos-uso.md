# Sessão 28 — NLB e GLB: casos de uso, preserve client IP e inline inspection

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-027-alb-oidc-mtls-waf

---

## Objetivo

Ao final, você conseguirá explicar os casos onde NLB é necessário (TCP/UDP, preserve client IP,
IP estático, PrivateLink), configurar um Gateway Load Balancer para rotear tráfego por appliances
de inspeção inline (IDS/IPS/firewall), e desenhar a topologia correta de uma VPC com GLB para
inspeção de tráfego Norte-Sul.

---

## Contexto

[FATO] A família ELB tem quatro membros: ALB (L7/HTTP), NLB (L4/TCP-UDP), GLB (L3/IP) e CLB
(legado). Cada um resolve um problema diferente — escolher o errado não é apenas ineficiente, é
tecnicamente inviável em certos casos. O ALB não consegue rotear tráfego TCP puro (como databases,
gRPC sem HTTP/2 wrapper, ou protocolos binários não-HTTP). O NLB não faz inspeção de conteúdo L7.
O GLB não roteia diretamente para aplicações — ele é exclusivamente um "bumper" de inspeção.

[CONSENSO] A tendência nos últimos anos é usar NLB como backing de PrivateLink endpoints para
expor serviços internos de forma segura entre contas e VPCs, e usar GLB como camada de inspeção
centralizada em arquiteturas hub-and-spoke com Transit Gateway. Entender NLB e GLB é pré-requisito
para arquiteturas de rede AWS além do básico.

---

## Conceitos principais

### 1. Network Load Balancer — características e modelo de operação

[FATO] O NLB opera na **camada 4 (transporte) do modelo OSI**. Ele não lê nem modifica o conteúdo
HTTP — apenas encaminha conexões TCP, UDP ou TLS com base em endereços IP e portas. Por operar em
L4, o NLB tem latência extremamente baixa (tipicamente sub-milissegundo) e consegue escalar para
**milhões de requisições por segundo** sem pré-aquecimento.

[FATO] Protocolos suportados por target group do NLB:

```
TCP       → conexão TCP pura; fluxo roteado pelo tempo de vida da conexão
TLS       → NLB faz TLS termination (ou passthrough) em L4
UDP       → sem handshake; fluxo identificado por 5-tuple
TCP_UDP   → target group aceita ambos na mesma porta (ex: DNS porta 53)
QUIC      → usa Connection ID no CID para routing; fallback para flow hash
TCP_QUIC  → target group aceita TCP e QUIC na mesma porta
```

**Algoritmo de roteamento por protocolo:**

```
TCP / TLS:
  Flow hash: protocolo + src IP + src port + dst IP + dst port + TCP sequence number
  → A mesma conexão TCP é sempre roteada para o mesmo target durante toda sua vida
  → Conexões diferentes do mesmo cliente (src port diferente) PODEM ir para targets distintos

UDP:
  Flow hash: protocolo + src IP + src port + dst IP + dst port
  → Sem TCP sequence number (UDP é connectionless)
  → Um UDP flow é consistente para o mesmo target pela vida do flow

QUIC:
  → Usa Server ID no Connection ID (CID) após estabelecimento
  → Permite migração de rede (conexão QUIC persiste mesmo mudando de IP/porta do cliente)
```

[FATO] Para cada Availability Zone habilitada, o NLB cria uma **interface de rede** (network
interface) com um endereço IP estático. Em load balancers internet-facing, é possível associar
um **Elastic IP** por AZ, tornando o endereço IP do NLB completamente previsível e fixo —
característica impossível no ALB (que usa IPs dinâmicos).

### 2. Casos de uso do NLB — quando ALB não basta

Os cinco cenários onde o NLB é a escolha correta ou única:

```
┌─────────────────────────────────────────────────────────────────────┐
│ 1. PROTOCOLOS NÃO-HTTP                                               │
│    Banco de dados (PostgreSQL 5432, MySQL 3306), Redis 6379,         │
│    MQTT (IoT), gRPC com HTTP/2 não-padrão, protocolos binários      │
│    → ALB não consegue inspecionar nem rotear esses protocolos        │
├─────────────────────────────────────────────────────────────────────┤
│ 2. PRESERVE CLIENT IP                                                │
│    Aplicações que precisam ver o IP real do cliente (firewalls,      │
│    geolocalização, auditoria, rate limiting por IP no backend)       │
│    → NLB pode preservar o IP de origem; ALB sempre usa proxy         │
├─────────────────────────────────────────────────────────────────────┤
│ 3. IP ESTÁTICO / ELASTIC IP                                          │
│    Clientes com whitelist de IP, integração com on-premises que      │
│    não aceita FQDN, compliance que exige IP fixo                     │
│    → NLB suporta Elastic IP por AZ; ALB usa IPs dinâmicos           │
├─────────────────────────────────────────────────────────────────────┤
│ 4. AWS PRIVATELINK (endpoint service)                                │
│    Expor serviços internos para outras contas/VPCs via endpoint      │
│    → PrivateLink exige NLB como backing; não suporta ALB             │
├─────────────────────────────────────────────────────────────────────┤
│ 5. LATÊNCIA ULTRA-BAIXA / THROUGHPUT EXTREMO                         │
│    Trading, telemetria de IoT, streaming de alto volume              │
│    → NLB não adiciona overhead L7; ALB tem processamento adicional   │
└─────────────────────────────────────────────────────────────────────┘
```

### 3. Preserve Client IP — comportamento por target type

[FATO] O comportamento do preserve client IP varia de acordo com o **target type** do target group
e o **protocolo**:

```
Target type INSTANCE:
  TCP / TLS    → client IP preservation HABILITADO por padrão (pode desabilitar)
  UDP / QUIC   → client IP preservation HABILITADO por padrão (NÃO pode desabilitar)

Target type IP:
  TCP / TLS    → client IP preservation DESABILITADO por padrão (pode habilitar)
  UDP / QUIC   → client IP preservation HABILITADO por padrão (NÃO pode desabilitar)
```

[FATO] Quando client IP preservation está **habilitado**, o NLB faz IP passthrough: o pacote que
chega ao target tem o IP de origem igual ao IP real do cliente. O target "vê" o cliente diretamente.

Quando está **desabilitado**, o src IP do pacote é o **endereço IP privado do NLB node** na AZ.
O target não tem visibilidade do IP real do cliente.

**Casos onde preserve client IP NÃO funciona (restrições):**

```
1. Targets alcançados via Transit Gateway          → não suportado
2. Tráfego originado do AWS PrivateLink            → src IP sempre é o IP do NLB
   (PrivateLink substitui o IP do cliente pelo IP do NLB — por design)
3. Target type = ALB (NLB na frente de ALB)        → ALB usa X-Forwarded-For para IP do cliente
4. Cross-zone routing para target em outra AZ      → pode funcionar, mas gera assimetria de fluxo
   quando o target responde de volta sem passar pelo NLB
```

[FATO] Uma armadilha clássica: se client IP preservation está habilitado e a **security group**
do target não permite tráfego vindo do IP do cliente (apenas do NLB), as conexões falham sem
mensagem de erro clara. O NLB não tem security group próprio — ele passa o tráfego transparentemente
quando IP preservation está on.

**Cross-zone load balancing no NLB:**

[FATO] No ALB, cross-zone load balancing está **habilitado por padrão** e não há custo adicional.
No NLB, está **desabilitado por padrão** — e quando habilitado, cobra por **data transfer entre
AZs** (diferente do ALB). O padrão desabilitado garante que cada NLB node distribui tráfego apenas
para targets na mesma AZ, evitando latência adicional e custos inter-AZ.

### 4. NLB e PrivateLink — exposição de serviços entre contas

[FATO] O AWS PrivateLink permite expor um serviço interno de uma conta/VPC para consumidores em
outras contas/VPCs sem precisar de peering, sem rotear tráfego pela internet, e sem expor o CIDR
inteiro da VPC. O backing de um PrivateLink endpoint service é **obrigatoriamente um NLB** (GLB
também é suportado para appliances, mas não ALB).

```
Conta do Produtor (Service Provider)                  Conta do Consumidor (Service Consumer)
┌────────────────────────────────────────┐            ┌──────────────────────────────────────┐
│                                        │            │                                      │
│  Targets (EC2, ECS, Lambda)            │            │  Consumidor faz requests para        │
│       ↑                                │            │  vpce-xxxxx.vpce-svc-yyyy.           │
│  NLB (backing do endpoint service)     │◄───────────│  us-east-1.vpce.amazonaws.com        │
│       ↑                                │  PrivateLink│       ↑                             │
│  VPC Endpoint Service                  │  (ENI)     │  VPC Endpoint (Interface)            │
│  (vpce-svc-xxxxxxxx)                   │            │  → cria ENI na subnet do consumidor  │
└────────────────────────────────────────┘            └──────────────────────────────────────┘
```

[FATO] O tráfego PrivateLink sempre apresenta o IP do NLB como src IP no target — o IP real do
consumidor é perdido. Isso é uma limitação de design do PrivateLink, não uma configuração do NLB.

### 5. Gateway Load Balancer — arquitetura e protocolo GENEVE

[FATO] O GLB opera na **camada 3 (rede) do modelo OSI**. Ele não termina conexões TCP nem lê
cabeçalhos de transporte — simplesmente recebe *todos* os pacotes IP em todas as portas e os
encaminha para appliances de segurança (firewalls, IDS/IPS, DPI). A appliance inspeciona o
pacote e o devolve ao GLB, que então encaminha ao destino original.

[FATO] A comunicação entre o GLB e as appliances usa o protocolo **GENEVE (Generic Network
Virtualization Encapsulation)** na porta UDP **6081**. O GLB encapsula o pacote original do
cliente dentro de um envelope GENEVE e o envia para a appliance. A appliance processa, mantém
(ou modifica) o conteúdo, e devolve com o mesmo envelope GENEVE. O GLB desencapsula e
encaminha ao destino.

```
Pacote original:
  [ IP header: src=203.0.113.1, dst=10.0.1.5 ][ TCP ][ HTTP payload ]

Encapsulado com GENEVE (enviado para a appliance):
  [ IP header: src=NLB-IP, dst=appliance-IP ][ UDP 6081 ][ GENEVE header ]
  [ IP header: src=203.0.113.1, dst=10.0.1.5 ][ TCP ][ HTTP payload ]
                ↑ pacote original preservado integralmente dentro do envelope
```

[FATO] O **GENEVE header** contém metadados que identificam o fluxo (TLVs — Type-Length-Value),
incluindo o ARN do GLB endpoint. Isso permite que appliances multi-tenant saibam de qual endpoint
o tráfego veio.

**Flow stickiness no GLB:**

[FATO] O GLB mantém stickiness de fluxo para garantir que pacotes de ida e volta de uma mesma
conexão passem pela mesma appliance (essencial para appliances stateful como firewalls). O
algoritmo pode ser configurado:

```
5-tuple (padrão): src IP + src port + dst IP + dst port + protocolo
3-tuple:          src IP + dst IP + protocolo
2-tuple:          src IP + dst IP
```

Quanto menos granular o tuple, mais tráfego é agrupado na mesma appliance — útil quando a
appliance precisa correlacionar tráfego de múltiplos fluxos do mesmo par IP.

### 6. Topologia GLB — inspeção Norte-Sul

"Norte-Sul" refere-se a tráfego que entra ou sai da VPC através do Internet Gateway (IGW).
A topologia GLB para inspeção Norte-Sul usa dois VPCs e **ingress routing** na tabela de rotas
do IGW.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          SERVICE CONSUMER VPC                                │
│                                                                              │
│  ┌──────────────────────┐           ┌──────────────────────────────────┐    │
│  │  Subnet do IGW       │           │  Subnet do GWLB Endpoint (GWLBe) │    │
│  │  (Ingress routing)   │           │  Route table:                    │    │
│  │                      │           │    0.0.0.0/0 → IGW               │    │
│  │  Route table:        │           └────────────┬─────────────────────┘    │
│  │  10.0.1.0/24 → GWLBe │                        │                         │
│  └──────────┬───────────┘      ┌──────────────────┘                        │
│             │                  │                  ┌──────────────────────┐  │
│  ┌──────────▼───────────┐      │                  │  Subnet da App       │  │
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
│               │   UDP 6081 deve estar aberto        │                         │
│               └────────────────────────────────────┘                         │
└───────────────────────────────────────────────────────────────────────────────┘
```

**Fluxo de tráfego de entrada (Internet → App):**

```
1. Pacote entra pelo Internet Gateway (IGW)
2. Ingress routing na route table do IGW:
   destino 10.0.1.0/24 (subnet da app) → target: GWLBe (vpc-endpoint-id)
3. GWLBe encaminha para o GLB no Service Provider VPC (via PrivateLink)
4. GLB encapsula com GENEVE e envia para uma appliance
5. Appliance inspeciona, aprova/bloqueia, devolve ao GLB
6. GLB desencapsula e devolve ao GWLBe
7. GWLBe encaminha à subnet da app (local route)
```

**Fluxo de tráfego de saída (App → Internet):**

```
1. Pacote sai da app
2. Route table da subnet da app: 0.0.0.0/0 → GWLBe
3. GWLBe → GLB → appliance (inspeciona) → GLB → GWLBe
4. Route table da subnet do GWLBe: 0.0.0.0/0 → IGW
5. IGW → Internet
```

[FATO] A **ingress routing** (tabela de rotas associada ao IGW) é um recurso específico para esse
padrão — permite que o IGW roteie tráfego *de entrada* para um endpoint antes de chegar à subnet
de destino. Sem isso, não é possível interceptar tráfego Norte-Sul de forma transparente.

[FATO] O GWLB endpoint é **zonal** — um por AZ. A AWS recomenda criar um GWLBe por AZ para
garantir que o fluxo de inspeção seja intra-AZ (evitando latência e custo de cross-AZ).

---

## Exemplo prático

**Cenário:** empresa de fintech precisa de:
1. NLB fronteando um serviço de processamento de pagamentos TCP (protocolo proprietário, porta 8443)
   com IP estático para whitelist dos parceiros
2. GLB inspecionando TODO o tráfego que entra na VPC antes de chegar às aplicações

### CDK Python — NLB com IP estático e preserve client IP

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

        # ── Elastic IPs para IPs estáticos do NLB ──────────────────────────
        eip_az1 = ec2.CfnEIP(self, "EipAz1", domain="vpc")
        eip_az2 = ec2.CfnEIP(self, "EipAz2", domain="vpc")

        # ── NLB internet-facing com Elastic IPs ────────────────────────────
        nlb = elbv2.NetworkLoadBalancer(self, "PaymentNLB",
            vpc=vpc,
            internet_facing=True,
            # Elastic IPs associados por AZ via subnet mappings
            # (via L1 CfnLoadBalancer quando CDK L2 não suporta diretamente)
        )

        # ── Target Group TCP com preserve client IP habilitado ──────────────
        tg = elbv2.NetworkTargetGroup(self, "PaymentTG",
            vpc=vpc,
            port=8443,
            protocol=elbv2.Protocol.TCP,
            target_type=elbv2.TargetType.IP,   # IP target type
            # Para IP target type + TCP, client IP é DESABILITADO por padrão
            # Para habilitar, usar preserve_client_ip=True
            health_check=elbv2.HealthCheck(
                protocol=elbv2.Protocol.TCP,
                interval=Duration.seconds(10),
                healthy_threshold_count=2,
                unhealthy_threshold_count=2,
            ),
        )

        # Habilitar preserve client IP no target group (via escape hatch L1)
        cfn_tg = tg.node.default_child
        cfn_tg.add_property_override("TargetGroupAttributes", [
            {"Key": "preserve_client_ip.enabled", "Value": "true"}
        ])

        # ── Listener TCP 8443 ───────────────────────────────────────────────
        nlb.add_listener("TcpListener",
            port=8443,
            protocol=elbv2.Protocol.TCP,
            default_target_groups=[tg],
        )

        # ── Security group do target — permite o IP do cliente diretamente ─
        # (não o IP do NLB, porque client IP preservation está ON)
        target_sg = ec2.SecurityGroup(self, "TargetSG", vpc=vpc)
        target_sg.add_ingress_rule(
            ec2.Peer.ipv4("0.0.0.0/0"),  # em produção: CIDR dos parceiros
            ec2.Port.tcp(8443),
            "Allow payment clients direct access (NLB client IP preservation)",
        )
```

### CDK Python — GLB para inspeção Norte-Sul (L1 CfnResources)

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

        # ── Service Provider VPC (appliances) ──────────────────────────────
        provider_vpc = ec2.Vpc(self, "ProviderVpc",
            max_azs=2,
            subnet_configuration=[
                ec2.SubnetConfiguration(
                    name="Appliance",
                    subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS,
                )
            ],
        )

        # ── Security group para appliances (deve aceitar UDP 6081) ─────────
        appliance_sg = ec2.SecurityGroup(self, "ApplianceSG", vpc=provider_vpc)
        appliance_sg.add_ingress_rule(
            ec2.Peer.any_ipv4(),
            ec2.Port.udp(6081),
            "GENEVE protocol from GLB",
        )

        # ── Target Group GENEVE para o GLB ──────────────────────────────────
        # (L1 porque CDK L2 não tem suporte completo ao GLB)
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

        # ── Gateway Load Balancer ───────────────────────────────────────────
        glb = elbv2.CfnLoadBalancer(self, "GLB",
            name="inspection-glb",
            type="gateway",
            subnets=[s.subnet_id for s in provider_vpc.private_subnets],
        )

        # Listener do GLB (escuta todos os IPs em todas as portas)
        elbv2.CfnListener(self, "GlbListener",
            load_balancer_arn=glb.ref,
            default_actions=[{
                "type": "forward",
                "targetGroupArn": glb_tg.ref,
            }],
        )

        # ── VPC Endpoint Service (PrivateLink backing) ──────────────────────
        endpoint_service = ec2.CfnVPCEndpointService(self, "EndpointService",
            gateway_load_balancer_arns=[glb.ref],
            acceptance_required=False,
        )
```

### CLI — comandos essenciais

**Verificar se client IP preservation está habilitado no TG:**
```bash
aws elbv2 describe-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/payment-tg/... \
  --query 'Attributes[?Key==`preserve_client_ip.enabled`]'
```

**Habilitar cross-zone load balancing no NLB:**
```bash
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn arn:aws:elasticloadbalancing:... \
  --attributes Key=load_balancing.cross_zone.enabled,Value=true
```

**Criar GLB via CLI e registrar appliance:**
```bash
# Criar GLB
aws elbv2 create-load-balancer \
  --name inspection-glb \
  --type gateway \
  --subnets subnet-aaaa subnet-bbbb

# Criar target group GENEVE
aws elbv2 create-target-group \
  --name inspection-appliances \
  --protocol GENEVE \
  --port 6081 \
  --vpc-id vpc-xxxx \
  --target-type instance \
  --health-check-protocol HTTP \
  --health-check-port 80 \
  --health-check-path /health

# Registrar appliance no TG
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/inspection-appliances/... \
  --targets Id=i-0123456789abcdef0

# Criar listener do GLB
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:...:loadbalancer/gwy/inspection-glb/... \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...

# Criar endpoint service
aws ec2 create-vpc-endpoint-service-configuration \
  --gateway-load-balancer-arns arn:aws:elasticloadbalancing:...:loadbalancer/gwy/inspection-glb/... \
  --no-acceptance-required

# Criar GWLB endpoint no consumer VPC
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-consumer \
  --service-name com.amazonaws.vpce.us-east-1.vpce-svc-xxxx \
  --vpc-endpoint-type GatewayLoadBalancer \
  --subnet-ids subnet-endpoint-az1

# Configurar ingress routing: IGW → GWLBe para tráfego destinado à subnet da app
aws ec2 create-route \
  --route-table-id rtb-igw-xxxxx \
  --destination-cidr-block 10.0.1.0/24 \
  --vpc-endpoint-id vpce-yyyy

# Configurar rota padrão na subnet da app → GWLBe (tráfego de saída)
aws ec2 create-route \
  --route-table-id rtb-app-xxxxx \
  --destination-cidr-block 0.0.0.0/0 \
  --vpc-endpoint-id vpce-yyyy
```

---

## Armadilhas comuns

### Armadilha 1: security group no target não permite o IP do cliente quando preserve client IP está ON

Quando client IP preservation está habilitado no NLB, o target recebe o pacote com o src IP igual
ao IP real do cliente — não o IP do NLB. Se o security group do target foi configurado para aceitar
tráfego apenas do NLB (ou do CIDR do NLB), todas as conexões falham silenciosamente. O NLB não
tem security group próprio, então não há "Allow from NLB security group" possível como no ALB.
O diagnóstico é: health checks passam (porque partem do IP do NLB node), mas tráfego real falha.
Verificar via CloudWatch `TCP_ELB_Reset_Count` — se alto, geralmente indica rejeição pelo target
ou security group.

### Armadilha 2: cross-zone no NLB tem custo e é off por padrão

No ALB, cross-zone load balancing é on por padrão e gratuito. No NLB, está off por padrão e
**cobra por data transfer entre AZs** quando habilitado. Times que migram do ALB para NLB e
habilitam cross-zone sem atenção descobrem surpresas na fatura. Além disso, com cross-zone off e
client IP preservation on, se um cliente cujo IP é roteado para o node do NLB na AZ-A (por DNS
ou Elastic IP) mas todos os targets saudáveis estão na AZ-B, as requisições falham — o NLB não
faz cross-zone e não tem targets disponíveis na AZ-A.

### Armadilha 3: assimetria de rotas na topologia GLB derruba a inspeção

Na topologia GLB, o tráfego de entrada e de saída de um mesmo fluxo deve passar pela **mesma
appliance** (firewalls stateful mantêm estado da conexão). Se as route tables estiverem
incorretas — por exemplo, a rota de retorno do tráfego saindo da app não passar pelo GWLBe — o
fluxo fica assimétrico: a appliance vê apenas metade da conversa e descarta os pacotes (ou
"resets" a conexão). O diagnóstico: `nc` ou `curl` para o alvo funciona mas a conexão é
derrubada logo em seguida; os logs da appliance mostram pacotes sem estado correspondente. A
correção é garantir que a route table da subnet da app tenha `0.0.0.0/0 → GWLBe` para forçar
todo o tráfego de saída também pelo endpoint.

---

## Exercício de reflexão

Você trabalha em uma empresa que precisa expor um serviço interno (autenticação de transações,
protocolo TCP proprietário, porta 9443) para 50 parceiros externos. Cada parceiro precisa usar o
IP do serviço em uma whitelist — FQDN não é aceito. Além disso, você precisa garantir que o
backend veja o IP de origem real de cada parceiro para fins de auditoria e geolocalização.

Descreva a arquitetura que você montaria: qual tipo de load balancer, como configurar os Elastic
IPs, como lidar com o preserve client IP (incluindo as implicações no security group do target),
e como você exporia esse serviço para parceiros de forma isolada (sem expor a VPC inteira ou criar
peering com cada um dos 50 parceiros). Considere também o que acontece com os health checks
quando client IP preservation está habilitado — de onde vêm os health checks do NLB e o que isso
significa para a configuração do security group.

---

## Recursos para aprofundar

**1. What is a Network Load Balancer?**
URL: https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html
O que você vai encontrar: visão geral dos componentes, algoritmos de flow hash por protocolo
(TCP vs UDP vs QUIC), diferenças entre cross-zone on/off, e referência para client IP preservation.
Por que é a fonte certa: documentação primária com especificações exatas de comportamento do NLB.

**2. Edit target group attributes for your Network Load Balancer**
URL: https://docs.aws.amazon.com/elasticloadbalancing/latest/network/edit-target-group-attributes.html
O que você vai encontrar: tabela completa de comportamento padrão do client IP preservation por
target type e protocolo, como habilitar/desabilitar, e limitações (Transit Gateway, PrivateLink).
Por que é a fonte certa: a tabela de defaults é a referência canônica para evitar a armadilha
mais comum do NLB.

**3. What is a Gateway Load Balancer?**
URL: https://docs.aws.amazon.com/elasticloadbalancing/latest/gateway/introduction.html
O que você vai encontrar: overview do GLB, GENEVE protocol, flow stickiness, e modelo de VPC
endpoints para troca de tráfego entre VPCs.
Por que é a fonte certa: documentação primária que explica o modelo conceitual e as relações
entre GLB, GWLB endpoint e PrivateLink.

**4. Getting started with Gateway Load Balancers**
URL: https://docs.aws.amazon.com/elasticloadbalancing/latest/gateway/getting-started.html
O que você vai encontrar: passo a passo da topologia Norte-Sul completa — criação do GLB,
endpoint service, GWLB endpoint, e as três route tables necessárias (IGW, subnet da app,
subnet do GWLBe) com exemplos exatos de entradas de rota.
Por que é a fonte certa: o único lugar na documentação oficial que mostra as três route tables
juntas e explica a direção do tráfego com diagrama numerado.

**5. Inspecting inbound traffic from the internet using firewall appliances with Gateway Load Balancer**
URL: https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/inspecting-inbound-traffic-fa.html
O que você vai encontrar: whitepaper com topologias avançadas de inspeção Norte-Sul e Leste-Oeste
usando GLB com Transit Gateway para arquiteturas hub-and-spoke.
Por que é a fonte certa: vai além do caso básico e mostra como escalar a inspeção para múltiplas
VPCs centralizadas, que é o caso de uso real em enterprises.

---

