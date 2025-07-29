# NetBox - Configuração e População de Dados

## 🏢 Pré-requisitos: Infraestrutura Física de Rede

### **Infraestrutura de Rede Organizada**
Antes de implementar o NetBox, é essencial ter uma infraestrutura física bem estruturada:

#### **Cabeamento Estruturado**
- **Identificação padronizada** de cabos e pontos de rede
- **Documentação física** atualizada (plantas, esquemas)
- **Certificação** de todos os links de dados
- **Organização em racks** com patch panels identificados

#### **Equipamentos Inventariados**
- **Lista completa** de dispositivos de rede (switches, roteadores, firewalls)
- **Localização física** definida (site, rack, posição)
- **Informações básicas**: modelo, serial, IP de gerência
- **Estado operacional** conhecido

#### **Endereçamento IP Documentado**
- **Plano de endereçamento** definido e implementado
- **Subredes** organizadas por função/localização
- **VLANs** mapeadas e documentadas
- **IPs de gerência** acessíveis via SNMP/SSH

#### **Topologia de Rede Mapeada**
- **Conexões físicas** entre equipamentos conhecidas  
- **Links de uplink/downlink** identificados
- **Redundâncias** e caminhos alternativos mapeados
- **Largura de banda** de cada segmento documentada

### **Acesso aos Dispositivos**
- **SNMP v2c/v3** habilitado nos equipamentos
- **SSH/Telnet** configurado para automação
- **Credenciais** padronizadas e seguras
- **Conectividade** do servidor NetBox para todos os dispositivos

---

## 🏗️ Requisitos de Infraestrutura do Servidor NetBox

### **Servidor NetBox**
- **CPU**: Mínimo 2 cores, recomendado 4+ cores
- **RAM**: Mínimo 4GB, recomendado 8GB+ para infraestruturas médias/grandes
- **Storage**: 
  - Sistema: 20GB SSD
  - Dados: 50GB+ (dependendo do tamanho da infraestrutura)
- **OS**: Ubuntu 20.04+ LTS ou CentOS 8+

### **Banco de Dados**
- **PostgreSQL 12+** (obrigatório para NetBox)
- Pode ser no mesmo servidor ou separado
- Para ambientes de produção: servidor dedicado recomendado

### **Dependências do Sistema**
- **Redis** (cache e task queue)
- **Python 3.8+**
- **Nginx** (proxy reverso)
- **Git** (para versionamento de configurações)

### **Conectividade de Rede**
- **Acesso HTTP/HTTPS** para interface web
- **Conectividade SNMP** para dispositivos de rede
- **SSH/Telnet** para automação via Ansible
- **API REST** habilitada para integração com scripts

### **Segurança**
- **Firewall** configurado (portas 80/443, 22)
- **SSL/TLS** certificado válido
- **Backup** automatizado do banco de dados
- **Autenticação** LDAP/RADIUS (opcional)

---

## 📊 Ordem de População dos Dados no NetBox

### **1. Estrutura Organizacional**
```
Sites/Localizações → Racks → Fabricantes → Tipos de Dispositivos
```

#### **1.1 Sites (Localizações)**
- Data Centers
- Filiais/Escritórios  
- Sites remotos
- Definir hierarquia geográfica

#### **1.2 Racks**
- Localização física nos sites
- Unidades de rack (RU)
- Capacidade de energia

#### **1.3 Fabricantes**
- Cisco, Huawei, Juniper, etc.
- Informações de suporte

#### **1.4 Tipos de Dispositivos**
- Modelos específicos (ASR1000, 2960X, etc.)
- Templates de configuração
- Especificações técnicas

### **2. Infraestrutura de Rede**
```
VRFs → ASNs → Prefixos IP → VLANs → Agregados IP
```

#### **2.1 VRFs (Virtual Routing and Forwarding)**
- Segmentação de rede
- Isolamento de tráfego

#### **2.2 ASNs (Autonomous System Numbers)**
- Numeração BGP
- Roteamento inter-domínios

#### **2.3 Prefixos IP**
- Blocos de endereços
- Hierarquia de sub-redes

#### **2.4 VLANs**
- Segmentação Layer 2
- Associação com sites/grupos

### **3. Dispositivos Físicos**
```
Dispositivos → Interfaces → Cabos → Conexões
```

#### **3.1 Dispositivos**
- Switches, Roteadores, Firewalls
- Localização (site + rack + posição)
- Informações de hardware

#### **3.2 Interfaces**
- Portas físicas e lógicas
- Tipos (Ethernet, Serial, etc.)
- Configurações específicas

#### **3.3 Cabos e Conexões**
- Topologia física
- Tipos de cabo (fiber, copper)
- Documentação de conectividade

### **4. Configuração Lógica**
```
IP Addresses → Serviços → Circuitos → Provedores
```

#### **4.1 Endereços IP**
- Atribuição a interfaces
- Endereços management
- IPs virtuais (HSRP/VRRP)

#### **4.2 Serviços**
- DNS, DHCP, NTP
- Aplicações críticas
- Portas e protocolos

### **5. Dados Operacionais**
```
Contacts → Tenants → Tags → Custom Fields
```

#### **5.1 Contacts**
- Responsáveis técnicos
- Fornecedores/Integradores
- Informações de contato

#### **5.2 Tenants**
- Departamentos/Clientes
- Isolamento organizacional

#### **5.3 Tags e Custom Fields**
- Categorização personalizada
- Metadados específicos da organização

---

## 🔄 Fluxo de População Recomendado

1. **Preparação** → Levantamento da infraestrutura atual
2. **Estrutura Base** → Sites, racks, fabricantes
3. **Rede Lógica** → VLANs, prefixos IP, VRFs  
4. **Dispositivos** → Cadastro dos equipamentos
5. **Conectividade** → Interfaces e cabos
6. **Endereçamento** → IPs e serviços
7. **Validação** → Testes de integridade dos dados

## 📝 Observações Importantes

- **Dados Consistentes**: Sempre validar informações antes da inserção
- **Backup Regular**: Exportar dados após cada etapa importante
- **Documentação**: Manter registro das decisões de modelagem
- **Automação**: Usar APIs para população em massa quando possível