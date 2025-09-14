# Arquitetura de Infraestrutura

## Visão Geral da Arquitetura

Esta arquitetura híbrida foi projetada especificamente para uma **empresa de pneus** que desenvolveu um webapp de e-commerce crítico, responsável por **mais de 80% do lucro da empresa**. A solução mantém **serviços críticos no ambiente local** (Active Directory sincronizado com Azure, FileServer, PrintServer) enquanto a aplicação de e-commerce (.NET 8 backend + Node.js frontend + MySQL) está hospedada na nuvem pública, garantindo **resiliência de dados**, **segurança robusta**, **observabilidade completa** e **gestão de custos eficiente**.

### Requisitos Específicos Atendidos
- **Stack Tecnológico:** Backend .NET 8 + Frontend Node.js + MySQL na nuvem
- **Usuários:** 500 usuários (230 escritório administrativo + 270 armazém)
- **Conectividade:** 100% Wi-Fi para todos os usuários
- **SSO Externo:** Autenticação única para clientes do e-commerce
- **Backoffice Restrito:** Acesso apenas por IPs internos da empresa
- **AD Híbrido:** Active Directory local sincronizado com Azure AD
- **Serviços Locais:** FileServer e PrintServer mantidos on-premises

### Princípios de Design
- **Híbrido por Design:** Combinação otimizada de recursos locais e em nuvem
- **Cloud First para Aplicações:** Aplicações críticas hospedadas na nuvem pública para melhor escalabilidade
- **On-Premises para Infraestrutura Base:** AD, DNS, Wi-Fi mantidos localmente para controle e performance
- **Conectividade Multi-Provedor:** Redundância completa de conectividade para alta disponibilidade
- **Segurança Zero Trust:** Implementação de segurança em todas as camadas da arquitetura

---

## 1. INFRAESTRUTURA LOCAL (ON-PREMISES)

### 1.1. Conectividade e SD-WAN
**Provedores de Internet (Redundância Tripla):**
- **Primário:** Fibra óptica (Backbone A) - conexão principal de alta velocidade
- **Secundário:** Fibra óptica (Backbone B) - backup automático para continuidade
- **Terciário:** 4G/5G - backup móvel para situações críticas

**Benefícios do SD-WAN:**
- Failover automático em menos de 1 segundo entre provedores
- Roteamento inteligente baseado em aplicação (priorização do tráfego e-commerce)
- Criptografia ponta a ponta automática para segurança de dados
- Gestão centralizada de políticas de rede e qualidade de serviço

### 1.2. Segmentação de Rede Interna
```
Internet (3 ISPs) → SD-WAN Edge → Firewall → Switch Core
    ├── VLAN 10: Gerenciamento (Equipamentos de rede)
    ├── VLAN 20: Servidores (AD, Arquivos, Impressão, DNS)
    ├── VLAN 30: Administrativo (230 usuários)
    ├── VLAN 40: Armazém (270 usuários)
    ├── VLAN 50: Convidados (Rede isolada para visitantes)
    └── VLAN 99: DMZ (Conectividade com nuvem)
```

### 1.3. Wi-Fi 6E Corporativo
**Especificações Técnicas:**
- **Padrão:** 802.11ax (Wi-Fi 6E) com banda de 6GHz para melhor performance
- **Autenticação:** WPA3-Enterprise + 802.1X via RADIUS para segurança máxima
- **Capacidade:** Suporte para 500 usuários simultâneos com QoS garantido
- **Controller:** Sistema centralizado com redundância para gestão unificada

**Controle de Acesso à Rede (NAC):**
- Verificação de conformidade do dispositivo antes do acesso
- Atribuição automática de VLAN baseada no usuário/dispositivo
- Quarentena automática para dispositivos não conformes
- Integração completa com Active Directory para autenticação

### 1.4. Serviços Locais Críticos (Mantidos Localmente)
| Serviço | Justificativa Técnica | Especificação Detalhada |
|---------|----------------------|-------------------------|
| **Active Directory** | Sincronização com Azure AD + fonte de identidade local | 2x Windows Server 2022, Azure AD Connect, FSMO distribuídas |
| **DNS/DHCP** | Resolução local rápida + performance otimizada | Sistema redundante com encaminhamento para 8.8.8.8 |
| **FileServer** | Armazenamento corporativo + compliance local | DFS-R para replicação, RAID 10, backup estratégia 3-2-1 |
| **PrintServer** | Gestão centralizada impressoras (crítico para armazém) | Filas centralizadas, drivers automáticos, controle de cotas |
| **NPS/RADIUS** | Autenticação 802.1X para rede e Wi-Fi | Integrado com AD, logs completos de auditoria |
| **Proxy/Cache** | Controle de navegação + otimização de banda | Squid/BlueCoat com filtro de conteúdo empresarial |

---

## 2. NUVEM PÚBLICA - ARQUITETURA DE SERVIÇOS

### 2.1. Arquitetura AWS (Implementação de Referência)

#### 2.1.1. Conectividade Híbrida
- **Direct Connect:** Conexão dedicada de 1GB com roteamento BGP e SLA de 99.9%
- **VPN Backup:** Túneis IPSec duplos com failover automático para redundância
- **Transit Gateway:** Hub central para múltiplas VPCs e conectividade híbrida
- **Configuração BGP:** AS65000 (local) → AS64512 (AWS) para roteamento otimizado

#### 2.1.2. Aplicação E-commerce Multi-AZ (.NET 8 + Node.js)
```
Internet (Clientes) → CloudFront → WAF → ALB (Multi-AZ)
                                           ↓
Frontend Node.js (3 AZs) ←→ Backend .NET 8 (3 AZs)
                                           ↓
                              MySQL Aurora (Multi-AZ)

Backoffice (IPs Internos) → VPN/Direct Connect → ALB Interno
                                                    ↓
                                    Backend .NET 8 (Admin APIs)
```

**Arquitetura de Containers:**
- **Frontend Node.js:** ECS Fargate com 2-6 tasks por AZ (auto-scaling)
- **Backend .NET 8:** ECS Fargate com 3-9 tasks por AZ (auto-scaling)
- **Load Balancing:** ALB público para frontend, ALB interno para backoffice
- **Session Management:** Redis ElastiCache para sessões compartilhadas

**Configuração de Auto Scaling:**
- **Target Tracking:** CPU 70%, Memória 80% para escalabilidade automática
- **Scheduled Scaling:** Redução de 50% dos recursos fora do horário comercial
- **Health Checks:** Verificação de saúde via ALB + ECS service
- **Rolling Updates:** Deploy blue-green com zero downtime

#### 2.1.3. Camada de Banco de Dados
```
Aurora MySQL Multi-AZ (Primário)
    ├── Writer Instance: db.r6g.xlarge (us-east-1a) - Escritas
    ├── Reader Instance: db.r6g.large (us-east-1b) - Leituras
    └── Cross-Region Read Replica: db.r6g.large (us-west-2) - DR
```

**Estratégia de Backup:**
- **Backups Automáticos:** Diários com retenção de 30 dias
- **Snapshots Manuais:** Semanais com retenção de 1 ano
- **Point-in-Time Recovery:** Recuperação com granularidade de 5 minutos
- **Replicação Cross-Region:** RPO (Recovery Point Objective) menor que 5 minutos

---

## 3. FLUXOS DE CONEXÃO E INTEGRAÇÃO

### 3.1. Fluxo SSO (Clientes Externos)
```
Cliente E-commerce
    ↓
CloudFront (CDN)
    ↓
WAF (Web Application Firewall)
    ↓
Application Load Balancer
    ↓
Frontend Node.js 
    ↓
Amazon Cognito (SSO)
    ↓ (JWT Token)
Backend .NET 8 API (ECS Fargate)
    ↓
Aurora MySQL (Multi-AZ)
```

**Detalhes do Fluxo:**
- **Passo 1:** Cliente acessa loja de pneus via browser
- **Passo 2:** CloudFront serve conteúdo estático (imagens de pneus, CSS, JS)
- **Passo 3:** WAF filtra ataques maliciosos (SQL injection, XSS)
- **Passo 4:** ALB distribui carga entre instâncias Node.js
- **Passo 5:** Frontend apresenta tela de login/cadastro
- **Passo 6:** Cognito autentica usuário (email/senha, social login)
- **Passo 7:** Token JWT é gerado e enviado ao browser
- **Passo 8:** Backend .NET 8 valida token e processa pedidos
- **Passo 9:** Dados são consultados/gravados no MySQL

### 3.2. Fluxo Backoffice (Apenas IPs Internos)
```
Funcionário Interno (IP 192.168.x.x)
    ↓
Active Directory Local (Autenticação)
    ↓
VPN/Direct Connect (Túnel Criptografado)
    ↓
Security Group AWS (IP Filtering)
    ↓
Internal Load Balancer (Apenas IPs internos)
    ↓
Backend .NET 8 - Admin APIs (ECS Fargate)
    ↓
Aurora MySQL (Admin Queries)
```

**Detalhes do Fluxo:**
- **Passo 1:** Funcionário faz login no AD local (Windows)
- **Passo 2:** Acessa backoffice via intranet (192.168.x.x)
- **Passo 3:** Tráfego passa por Direct Connect (dedicado) ou VPN
- **Passo 4:** Security Group valida se IP é da faixa interna
- **Passo 5:** Load Balancer interno roteia para APIs administrativas
- **Passo 6:** Backend processa operações admin (relatórios, estoque)
- **Passo 7:** Dados sensíveis são acessados com permissões elevadas

### 3.3. Fluxo Sincronização AD ↔ Azure AD
```
Active Directory Local
    ↓
Azure AD Connect (Sync Agent)
    ↓
Azure Active Directory
    ↓
SAML/OIDC Integration
    ↓
AWS Cognito (Identity Federation)
    ↓
Backend .NET 8 (User Context)
```

**Detalhes do Fluxo:**
- **Passo 1:** Funcionário é criado no AD local
- **Passo 2:** Azure AD Connect sincroniza (delta sync a cada 30min)
- **Passo 3:** Usuário fica disponível no Azure AD
- **Passo 4:** Se funcionário acessar backoffice, usa identidade federada
- **Passo 5:** JWT token inclui roles/groups do AD
- **Passo 6:** Backend .NET 8 aplica autorização baseada em grupos

### 3.4. Fluxo Wi-Fi Corporativo (500 Usuários)
```
Dispositivo do Usuário
    ↓
Wi-Fi 6E (WPA3-Enterprise)
    ↓
RADIUS/NPS Server (802.1X)
    ↓
Active Directory (Validação)
    ↓
VLAN Assignment Dinâmica
    ├── VLAN 30: Administrativo (230 usuários)
    └── VLAN 40: Armazém (270 usuários)
    ↓
Switch Core
    ↓
Internet/Intranet Access
```

**Detalhes do Fluxo:**
- **Passo 1:** Usuário conecta dispositivo ao Wi-Fi corporativo
- **Passo 2:** WPA3-Enterprise solicita credenciais de domínio
- **Passo 3:** RADIUS consulta AD para validar usuário/senha
- **Passo 4:** Baseado no grupo AD, usuário é colocado na VLAN correta
- **Passo 5:** VLAN 30 (administrativo) tem acesso completo
- **Passo 6:** VLAN 40 (armazém) tem acesso restrito a sistemas específicos

---

## 4. SEGURANÇA HÍBRIDA (MODELO ZERO TRUST)

### 4.1. Gestão de Identidade e Acesso (SSO + Backoffice)
```
Clientes E-commerce → Cognito/Auth0 (SSO) → JWT → Frontend Node.js
                                                       ↓
Funcionários → AD Local → Azure AD Sync → SAML → Backend .NET 8

Backoffice Admin → IP Validation → VPN/Direct Connect → ALB Interno
                                                            ↓
                                              Backend .NET 8 (Admin APIs)
```

**Controle de Acesso Backoffice:**
- **IP Whitelisting:** Security Groups AWS restringindo acesso por CIDR interno
- **VPN Obrigatória:** Conexão via Direct Connect ou VPN Site-to-Site
- **MFA Corporativo:** Autenticação dupla via Azure AD para administradores
- **Session Timeout:** Timeout agressivo de 30 minutos para sessões admin

### 4.2. Camadas de Segurança de Rede
1. **Perímetro:** Criptografia SD-WAN + proteção contra DDoS
2. **DMZ:** Regras de firewall + inspeção IPS/IDS em tempo real
3. **Edge da Nuvem:** WAF + proteção via CDN contra ataques web
4. **Aplicação:** Security groups + NACLs restritivas por microsserviço
5. **Dados:** Criptografia em repouso + em trânsito com chaves rotacionadas

### 4.3. Conformidade e Governança
**Conformidade com Padrões:**
- **PCI DSS:** Nível 1 para processamento seguro de pagamentos
- **LGPD:** Atendimento completo à lei brasileira de proteção de dados
- **ISO 27001:** Gestão de segurança da informação certificada
- **SOC 2 Type II:** Certificações dos provedores de nuvem validadas

**Governança de Dados:**
- **Classificação:** Público, Interno, Confidencial, Restrito
- **Prevenção de Vazamentos:** CASB na nuvem + DLP local
- **Criptografia:** AES-256 em repouso, TLS 1.3 em trânsito
- **Gestão de Chaves:** HSM com rotação automática de chaves

---

## 5. OBSERVABILIDADE E MONITORAMENTO

### 5.1. Estratégia de Observabilidade Integrada
```
Aplicações → OpenTelemetry → Monitoramento Nativo da Nuvem
     ↓                              ↓
Infraestrutura → Prometheus → Dashboards Grafana
     ↓                              ↓
Logs → Logging Centralizado → Análise SIEM
```

### 5.2. Fluxo Observabilidade e Alertas
```
Aplicação (.NET 8 + Node.js)
    ↓
OpenTelemetry (Traces + Metrics)
    ↓
CloudWatch/Prometheus
    ↓
Grafana Dashboards
    ↓
Alert Manager
    ↓ (Thresholds atingidos)
PagerDuty/Slack/Email
    ↓
Equipe de Plantão
```

**Detalhes do Fluxo:**
- **Passo 1:** Aplicação gera métricas (tempo resposta, erros, CPU)
- **Passo 2:** OpenTelemetry coleta e envia para monitoring
- **Passo 3:** Dashboards mostram health em tempo real
- **Passo 4:** Alertas são disparados quando SLOs são violados
- **Passo 5:** Notificações são enviadas por canal apropriado (P0=chamada, P1=SMS)
- **Passo 6:** Equipe recebe e responde conforme runbook

### 5.3. Objetivos de Nível de Serviço (SLOs) - E-commerce Crítico
| Serviço | Indicador (SLI) | Objetivo (SLO) | Impacto no Negócio |
|---------|-----------------|----------------|-------------------|
| **Frontend Node.js** | Carregamento página P95 | < 1.5s | Conversão direta em vendas |
| **Backend .NET 8** | Tempo resposta API P99 | < 300ms | UX e performance checkout |
| **MySQL Aurora** | Tempo consulta P95 | < 50ms | Catálogo e carrinho responsivos |
| **SSO Cognito** | Taxa de sucesso login | > 99.9% | Acesso de clientes garantido |
| **Backoffice** | Disponibilidade | > 99.5% | Operações internas críticas |

### 5.4. Estratégia de Alertas Escalonados
**Níveis de Severidade de Alertas:**
- **P0 (Crítico):** Indisponibilidade completa do serviço → PagerDuty + chamada
- **P1 (Alto):** Performance degradada → Slack + SMS para equipe
- **P2 (Médio):** Violação de SLO → Email + abertura de ticket
- **P3 (Baixo):** Condições de aviso → Apenas dashboard

### 5.5. Dashboards de Business Intelligence
- **Dashboard Executivo:** Receita, pedidos, taxas de conversão em tempo real
- **Dashboard Operacional:** Saúde da infraestrutura, métricas de performance
- **Dashboard de Segurança:** Ameaças detectadas, status de conformidade
- **Dashboard de Custos:** Acompanhamento de gastos, oportunidades de otimização

---

## 6. RECUPERAÇÃO DE DESASTRES E CONTINUIDADE DO NEGÓCIO

### 6.1. Objetivos de Recuperação Definidos
- **RTO (Tempo de Recuperação):** 2 horas para restauração completa do serviço
- **RPO (Ponto de Recuperação):** Máximo de 5 minutos de perda de dados
- **Meta de Disponibilidade:** 99.9% (8.7 horas de downtime por ano)

### 6.2. Fluxo Disaster Recovery
```
Região Primária (us-east-1)
    ↓ (Replicação Contínua)
Região Secundária (us-west-2)
    ↓ (Em caso de falha total)
Route 53 Health Check
    ↓ (Detecta falha)
DNS Failover Automático
    ↓
Usuários redirecionados para DR
    ↓
RDS Read Replica → Promoted to Master
    ↓
Aplicação ativa em região secundária
```

**Detalhes do Fluxo:**
- **Passo 1:** Replicação cross-region contínua (RPO < 5min)
- **Passo 2:** Health checks monitoram região primária
- **Passo 3:** Em caso de falha, Route 53 altera DNS automaticamente
- **Passo 4:** Read replica é promovida para master
- **Passo 5:** Auto Scaling inicia instâncias na região DR
- **Passo 6:** RTO de 2 horas para restauração completa

### 6.3. Estratégia de Backup (Regra 3-2-1-1)
- **3 cópias** de todos os dados críticos
- **2 tipos diferentes** de armazenamento (local + nuvem)
- **1 backup offsite** em região geográfica diferente
- **1 backup offline** mensal em fita magnética para compliance

### 6.4. Cronograma de Testes de DR
- **Mensal:** Testes de failover de componentes individuais
- **Trimestral:** Ativação completa do ambiente de DR
- **Semestral:** Simulação de continuidade do negócio com equipes
- **Anual:** Drill completo de recuperação de desastres com stakeholders

---

## 7. GESTÃO DE CUSTOS E OTIMIZAÇÃO FINANCEIRA

### 7.1. Estratégias de Otimização Específicas para E-commerce
- **Reserved Instances:** Baseline para backend .NET 8 (economia 40-60%)
- **Spot Instances:** Ambientes de desenvolvimento e testes (economia 70-90%)
- **Auto-scaling Inteligente:** Scale-out durante picos de vendas, scale-in fora do horário
- **CloudFront Caching:** Cache agressivo para catálogo de produtos (redução de 80% nas requests)
- **RDS Aurora Serverless:** Para ambientes de desenvolvimento (pay-per-use)

### 7.2. Fluxo Gestão de Custos (FinOps)
```
CloudWatch Metrics
    ↓
Cost Explorer APIs
    ↓
Custom Dashboard (Custo vs Receita)
    ↓
Lambda Function (Cost Analysis)
    ↓ (Se custo/receita > 3%)
Slack Alert para FinOps Team
    ↓
Automated Right-sizing
    ↓
Reserved Instance Recommendations
```

**Detalhes do Fluxo:**
- **Passo 1:** Métricas de uso são coletadas continuamente
- **Passo 2:** APIs de billing calculam custos em tempo real
- **Passo 3:** Dashboard mostra custo por transação de e-commerce
- **Passo 4:** Lambda analisa se ratio custo/receita está acima do target
- **Passo 5:** Alertas são enviados para equipe FinOps
- **Passo 6:** Ações automáticas de otimização são executadas

### 7.3. Métricas de Custo vs. Receita (ROI)
```
Custo Infraestrutura vs. Receita E-commerce
     ↓                           ↓
  Monitoramento Real-time → Alertas de Custo/Receita
     ↓                           ↓
  Auto-scaling baseado em vendas → Otimização dinâmica
```

**KPIs Financeiros:**
- **Custo por Transação:** < R$ 0,50 por pedido processado
- **Revenue per Instance Hour:** Mínimo R$ 100/hora durante picos
- **Infrastructure Cost Ratio:** < 3% da receita bruta do e-commerce
- **Availability Cost:** Custo de downtime = R$ 10.000/hora (80% do lucro)

### 7.4. Implementação FinOps
```
Visibilidade de Custos → Otimização → Controle Financeiro
     ↓                    ↓                ↓
  Tagueamento         Reserved Instances    Orçamentos
  Relatórios         Auto-scaling         Alertas
  Análises           Right-sizing         Políticas
```

---

## 8. AUTOMAÇÃO E DEVOPS

### 8.1. Fluxo CI/CD (DevOps)
```
Developer Commit (GitHub)
    ↓
GitHub Actions (CI Pipeline)
    ↓
Build & Test (.NET 8 + Node.js)
    ↓
Push to ECR (Container Registry)
    ↓
Deploy to ECS Staging
    ↓ (Testes automáticos OK)
Blue-Green Deploy to Production
    ↓
Health Check Validation
    ↓ (Se OK, traffic switch)
Old version shutdown
```

**Detalhes do Fluxo:**
- **Passo 1:** Desenvolvedor faz commit no repositório
- **Passo 2:** Pipeline CI/CD é ativado automaticamente
- **Passo 3:** Código é compilado e testado (unit + integration tests)
- **Passo 4:** Container images são criadas e enviadas para ECR
- **Passo 5:** Deploy primeiro em ambiente de staging
- **Passo 6:** Se testes passam, deploy blue-green em produção
- **Passo 7:** Traffic é gradualmente movido para nova versão
- **Passo 8:** Versão antiga é mantida por 24h para rollback rápido

### 8.2. Infrastructure as Code
- **Terraform:** Provisionamento completo da infraestrutura AWS
- **Ansible:** Configuração e gestão de servidores locais
- **GitOps:** Versionamento e controle de mudanças
- **Automated Testing:** Validação contínua da infraestrutura

---

## 9. COMPARAÇÃO DE SERVIÇOS POR PROVEDOR DE NUVEM

| Função | AWS (Referência) | Azure (Equivalente) | Oracle OCI (Equivalente) |
|--------|------------------|---------------------|--------------------------|
| **Conectividade Dedicada** | Direct Connect | ExpressRoute | FastConnect |
| **VPN Site-to-Site** | VPN Gateway | VPN Gateway | VPN Connect |
| **Hub de Conectividade** | Transit Gateway | Virtual WAN Hub | Dynamic Routing Gateway |
| **Rede Virtual** | VPC | Virtual Network | Virtual Cloud Network (VCN) |
| **Balanceador de Carga** | Application Load Balancer | Application Gateway | Load Balancer |
| **Plataforma de Containers** | ECS Fargate | Container Instances | Container Instances |
| **Kubernetes Gerenciado** | EKS | AKS | Container Engine (OKE) |
| **Banco MySQL Gerenciado** | Aurora MySQL | Azure Database MySQL | MySQL Database Service |
| **Cache Redis** | ElastiCache | Azure Cache for Redis | OCI Cache |
| **Armazenamento de Objetos** | S3 | Blob Storage | Object Storage |
| **CDN** | CloudFront | Azure CDN | Oracle CDN |
| **Web Application Firewall** | AWS WAF | Application Gateway WAF | Web Application Firewall |
| **Identidade Externa** | Cognito | Azure AD B2C | Identity Cloud Service (IDCS) |
| **Gerenciamento de Secrets** | Secrets Manager | Key Vault | OCI Vault |
| **Gerenciamento de Chaves** | KMS | Key Vault | OCI Vault (HSM) |
| **Fila de Mensagens** | SQS | Service Bus Queue | Queue Service |
| **Monitoramento** | CloudWatch | Azure Monitor | Application Performance Monitoring |
| **Rastreamento Distribuído** | X-Ray | Application Insights | APM Trace Explorer |
| **Prometheus Gerenciado** | Amazon Managed Prometheus | Azure Monitor for Prometheus | N/A (terceiros) |
| **Grafana Gerenciado** | Amazon Managed Grafana | Azure Managed Grafana | N/A (terceiros) |
| **DNS Gerenciado** | Route 53 | Azure DNS | DNS |
| **Gestão de Certificados** | ACM | Key Vault Certificates | Certificates Service |
| **Centro de Segurança** | Security Hub | Defender for Cloud | Cloud Guard |
| **Conformidade e Políticas** | Config | Policy | Cloud Guard |
| **Gestão de Custos** | Cost Explorer | Cost Management | Cost Analysis |
| **Serviço de Backup** | AWS Backup | Azure Backup | Database Backup |

---

## 10. MELHORIAS E INOVAÇÕES PROPOSTAS

### 10.1. Otimizações para Negócio de Pneus
- **CDN Geográfico:** Cache regional para catálogo de pneus por região/cidade
- **Inventory Real-time:** Integração API com estoque físico do armazém
- **Recommendation Engine:** ML para sugestão de pneus baseado em veículo/região
- **Mobile-First:** PWA para vendedores externos e aplicativo mobile

### 10.2. Analytics e Business Intelligence
- **Data Lake:** S3 + Athena para análise de vendas e comportamento
- **Real-time Dashboards:** Métricas de vendas, estoque e performance em tempo real
- **Predictive Analytics:** ML para previsão de demanda sazonal de pneus
- **Customer Journey:** Tracking completo desde acesso até conversão

### 10.3. Sustentabilidade e ESG
- **Green Computing:** Uso de instâncias Graviton (ARM) para menor consumo
- **Carbon Footprint:** Monitoramento pegada de carbono da infraestrutura
- **Efficient Scaling:** Auto-scaling agressivo para reduzir desperdício de recursos
- **Renewable Energy:** Preferência por regiões AWS com energia renovável

---

## 11. CONCLUSÃO

Esta arquitetura híbrida foi especificamente desenhada para atender aos requisitos críticos de uma **empresa de pneus com e-commerce responsável por 80% do lucro**, oferecendo:

**Resiliência de Dados:**
- MySQL Aurora Multi-AZ com RPO < 5 minutos
- Backup automatizado 3-2-1-1 com retenção adequada
- Cross-region replication para DR geográfico

**Medidas de Segurança:**
- Zero Trust com segmentação completa
- SSO para clientes + IP whitelisting para backoffice
- WAF + DDoS protection para proteção da aplicação
- AD local sincronizado com Azure AD para identidade híbrida

**Observabilidade Completa:**
- Monitoramento end-to-end da stack (.NET 8 + Node.js + MySQL)
- SLOs específicos para cada componente crítico
- Alertas escalonados baseados no impacto no negócio
- Dashboards executivos com métricas de receita vs. infraestrutura

**Gestão de Custos Eficiente:**
- Auto-scaling baseado em demanda real de vendas
- Reserved Instances para baseline + Spot para dev/test
- Métricas de ROI (custo por transação, revenue per instance)
- Otimização contínua com FinOps implementado

A arquitetura garante que o ambiente crítico de e-commerce permaneça disponível, performático e seguro, enquanto mantém os custos operacionais sob controle e alinhados com a receita gerada.