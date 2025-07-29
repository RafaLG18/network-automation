# Ambiente de Laboratório - GNS3 com RouterOS e Docker

## 🎯 Visão Geral do Laboratório

Este ambiente simula uma infraestrutura de rede empresarial usando **GNS3** como plataforma de virtualização, **RouterOS (MikroTik)** para equipamentos de rede e **containers Docker** para simular computadores e servidores.

### **Objetivos do Laboratório**
- Validar automação de configuração via Ansible
- Testar integração NetBox → Zabbix
- Simular cenários reais de troubleshooting
- Demonstrar pipelines CI/CD em ambiente controlado
- Treinar equipe em ferramentas de automação

---

## 🏗️ Arquitetura do Laboratório

### **Topologia de Rede**
```
                    [Internet Cloud]
                           |
                    [RouterOS-GW]
                     (10.0.0.1/24)
                           |
           ┌─────────────────────────────────┐
           │                                 │
    [RouterOS-SW1]                  [RouterOS-SW2]
   (192.168.10.1/24)               (192.168.20.1/24)
           │                                 │
    ┌──────┴──────┐                  ┌──────┴──────┐
    │             │                  │             │
[Docker-PC1] [Docker-PC2]      [Docker-SRV1] [Docker-SRV2]
.10.10/24    .10.20/24         .20.10/24     .20.20/24

                    [Management Network]
                        (172.16.0.0/24)
                             │
                 ┌───────────┼───────────┐
                 │           │           │
            [NetBox]    [Zabbix]   [GitLab-CI]
           .16.10/24   .16.20/24   .16.30/24
```

### **Componentes do Ambiente**
- **3x RouterOS**: Gateway + 2 Switches L3
- **4x Docker Containers**: Simular PCs e Servidores
- **3x Serviços**: NetBox, Zabbix, GitLab-CI
- **1x Cloud**: Acesso à internet (NAT)

---

## 🖥️ Configuração do GNS3

### **Requisitos do Sistema Host**
- **CPU**: 8+ cores (Intel VT-x/AMD-V habilitado)
- **RAM**: 16GB+ (32GB recomendado)
- **Storage**: 50GB+ SSD disponível
- **OS**: Ubuntu 20.04+ ou Windows 10/11
- **Virtualização**: KVM (Linux) ou VMware/VirtualBox (Windows)

### **Instalação GNS3**

#### **Ubuntu/Linux**
```bash
# Adicionar repositório GNS3
sudo add-apt-repository ppa:gns3/ppa
sudo apt update

# Instalar GNS3 GUI e Server
sudo apt install gns3-gui gns3-server

# Configurar usuário para usar GNS3
sudo usermod -aG gns3 $USER
sudo usermod -aG docker $USER
sudo usermod -aG libvirt $USER

# Instalar dependências adicionais
sudo apt install qemu-kvm qemu-utils libvirt-daemon-system libvirt-clients bridge-utils virt-manager
```

#### **Configuração Inicial**
```bash
# Iniciar serviços
sudo systemctl enable libvirtd
sudo systemctl start libvirtd

# Configurar bridge para networking
sudo virsh net-start default
sudo virsh net-autostart default
```

### **Configuração do GNS3 Server**
```json
{
    "version": "2.2.0",
    "type": "topology",
    "revision": 1,
    "auto_close": true,
    "auto_open": false,
    "auto_start": false,
    "drawing_grid_size": 25,
    "grid_size": 75,
    "show_grid": false,
    "show_interface_labels": true,
    "show_layers": false,
    "snap_to_grid": false,
    "supplier": null,
    "variables": null
}
```

---

## 🔧 RouterOS (MikroTik) no GNS3

### **Download e Preparação**
- **Imagem**: RouterOS CHR (Cloud Hosted Router)
- **Versão**: 7.x Long-term (stable)
- **Formato**: VMDK ou QCOW2
- **Tamanho**: ~50MB compactado

### **Configuração no GNS3**

#### **Template RouterOS**
```
Nome: MikroTik RouterOS 7.x
Tipo: QEMU VM
Categoria: Router
Symbol: :/symbols/router.svg

Configurações de VM:
- RAM: 512MB (mínimo), 1GB (recomendado)
- Disk: 1GB (suficiente para configurações)
- Network adapters: 4x Intel e1000
- Console: telnet (porta automática)
- Boot priority: HDD
```

#### **Configuração Base RouterOS**
```bash
# Configuração inicial via console
/user set 0 password=admin123
/ip service set telnet disabled=no
/ip service set ssh disabled=no

# Configurar interface de management
/ip address add address=172.16.0.1/24 interface=ether1
/ip route add gateway=172.16.0.254

# Habilitar SNMP para Zabbix
/snmp set enabled=yes
/snmp community set 0 name=public

# Configurar NTP
/system ntp client set enabled=yes servers=pool.ntp.org
```

### **Templates por Função**

#### **RouterOS-Gateway**
```bash
# Configuração WAN/LAN
/ip address add address=10.0.0.1/24 interface=ether1 comment="WAN"
/ip address add address=192.168.1.1/24 interface=ether2 comment="LAN"

# NAT para internet
/ip firewall nat add chain=srcnat action=masquerade out-interface=ether1

# DHCP Server
/ip pool add name=dhcp-pool ranges=192.168.1.100-192.168.1.200
/ip dhcp-server add name=dhcp1 interface=ether2 address-pool=dhcp-pool
/ip dhcp-server network add address=192.168.1.0/24 gateway=192.168.1.1 dns-server=8.8.8.8

# Firewall básico
/ip firewall filter add chain=input action=accept connection-state=established,related
/ip firewall filter add chain=input action=accept protocol=icmp
/ip firewall filter add chain=input action=drop in-interface=ether1
```

#### **RouterOS-Switch**
```bash
# Configuração L3 Switch
/interface bridge add name=bridge1
/interface bridge port add bridge=bridge1 interface=ether2
/interface bridge port add bridge=bridge1 interface=ether3
/interface bridge port add bridge=bridge1 interface=ether4

# VLANs
/interface vlan add name=vlan10 vlan-id=10 interface=bridge1
/interface vlan add name=vlan20 vlan-id=20 interface=bridge1

# Endereçamento por VLAN
/ip address add address=192.168.10.1/24 interface=vlan10
/ip address add address=192.168.20.1/24 interface=vlan20

# Inter-VLAN routing habilitado por padrão
```

---

## 🐳 Docker Containers como Computadores

### **Imagens Docker Personalizadas**

#### **Dockerfile - Ubuntu Desktop Simulado**
```dockerfile
FROM ubuntu:22.04

# Instalar ferramentas de rede e utilitários
RUN apt-get update && apt-get install -y \
    iputils-ping \
    traceroute \
    curl \
    wget \
    net-tools \
    openssh-server \
    snmp \
    snmp-mibs-downloader \
    python3 \
    python3-pip \
    nano \
    htop \
    iperf3 \
    tcpdump \
    && rm -rf /var/lib/apt/lists/*

# Configurar SSH
RUN mkdir /var/run/sshd
RUN echo 'root:admin123' | chpasswd
RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config

# Instalar agente Zabbix
RUN wget https://repo.zabbix.com/zabbix/6.4/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.4-1+ubuntu22.04_all.deb
RUN dpkg -i zabbix-release_6.4-1+ubuntu22.04_all.deb
RUN apt-get update && apt-get install -y zabbix-agent2

# Script de inicialização
COPY startup.sh /startup.sh
RUN chmod +x /startup.sh

EXPOSE 22 10050
CMD ["/startup.sh"]
```

#### **Script startup.sh**
```bash
#!/bin/bash

# Configurar hostname baseado na variável de ambiente
if [ ! -z "$HOSTNAME" ]; then
    echo "$HOSTNAME" > /etc/hostname
    hostname "$HOSTNAME"
fi

# Configurar Zabbix Agent
if [ ! -z "$ZABBIX_SERVER" ]; then
    sed -i "s/Server=127.0.0.1/Server=$ZABBIX_SERVER/" /etc/zabbix/zabbix_agent2.conf
    sed -i "s/ServerActive=127.0.0.1/ServerActive=$ZABBIX_SERVER/" /etc/zabbix/zabbix_agent2.conf
    sed -i "s/Hostname=Zabbix server/Hostname=$HOSTNAME/" /etc/zabbix/zabbix_agent2.conf
fi

# Iniciar serviços
service ssh start
service zabbix-agent2 start

# Manter container rodando
tail -f /dev/null
```

### **Docker Compose para Computadores**
```yaml
version: '3.8'

services:
  # Simulação PC1 - VLAN 10
  pc1:
    build: ./docker-pc
    container_name: lab-pc1
    hostname: PC1-Finance
    environment:
      - HOSTNAME=PC1-Finance
      - ZABBIX_SERVER=172.16.0.20
    networks:
      vlan10:
        ipv4_address: 192.168.10.10
    cap_add:
      - NET_ADMIN
    privileged: true

  # Simulação PC2 - VLAN 10  
  pc2:
    build: ./docker-pc
    container_name: lab-pc2
    hostname: PC2-Finance
    environment:
      - HOSTNAME=PC2-Finance
      - ZABBIX_SERVER=172.16.0.20
    networks:
      vlan10:
        ipv4_address: 192.168.10.20

  # Simulação Server1 - VLAN 20
  srv1:
    build: ./docker-server
    container_name: lab-srv1
    hostname: SRV1-Web
    environment:
      - HOSTNAME=SRV1-Web
      - ZABBIX_SERVER=172.16.0.20
    networks:
      vlan20:
        ipv4_address: 192.168.20.10
    ports:
      - "8080:80"

  # Simulação Server2 - VLAN 20
  srv2:
    build: ./docker-server
    container_name: lab-srv2
    hostname: SRV2-DB
    environment:
      - HOSTNAME=SRV2-DB
      - ZABBIX_SERVER=172.16.0.20
    networks:
      vlan20:
        ipv4_address: 192.168.20.20
    ports:
      - "3306:3306"

networks:
  vlan10:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.10.0/24
          gateway: 192.168.10.1
  
  vlan20:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.20.0/24
          gateway: 192.168.20.1

  management:
    driver: bridge
    ipam:
      config:
        - subnet: 172.16.0.0/24
          gateway: 172.16.0.1
```

---

## 🔗 Integração GNS3 + Docker

### **Configuração de Conectividade**

#### **Conectar GNS3 ao Docker**
```bash
# Criar bridge para integração
sudo docker network create --driver bridge --subnet=192.168.100.0/24 gns3-bridge

# No GNS3, adicionar Cloud conectado à bridge
# Cloud → Configure → Show special Ethernet interfaces → docker0
```

#### **TAP Interfaces (Alternativa)**
```bash
# Criar TAP interface
sudo ip tuntap add tap0 mode tap
sudo ip link set tap0 up
sudo ip addr add 192.168.100.1/24 dev tap0

# Conectar container à TAP
docker run -d --name test-pc --net=none ubuntu:22.04
sudo docker exec test-pc ip link set eth0 up
```

### **Scripts de Automação**

#### **deploy-lab.sh**
```bash
#!/bin/bash

echo "🚀 Deploying Network Automation Lab..."

# Verificar pré-requisitos
echo "📋 Checking prerequisites..."
docker --version || { echo "Docker not found!"; exit 1; }
gns3server --version || { echo "GNS3 Server not found!"; exit 1; }

# Build das imagens Docker
echo "🐳 Building Docker images..."
docker build -t lab-pc:latest ./docker-pc/
docker build -t lab-server:latest ./docker-server/

# Deploy dos containers
echo "📦 Deploying containers..."
docker-compose -f lab-compose.yml up -d

# Aguardar inicialização
echo "⏳ Waiting for services to start..."
sleep 30

# Verificar conectividade
echo "🔍 Testing connectivity..."
docker exec lab-pc1 ping -c 3 192.168.10.1
docker exec lab-srv1 ping -c 3 192.168.20.1

echo "✅ Lab deployment completed!"
echo "🌐 Access points:"
echo "  - NetBox: http://172.16.0.10"
echo "  - Zabbix: http://172.16.0.20"
echo "  - GitLab: http://172.16.0.30"
```

---

## 📊 Monitoramento do Laboratório

### **Métricas Coletadas**
- **RouterOS**: CPU, Memory, Interface utilization, Temperature
- **Docker Containers**: CPU, Memory, Network, Process count
- **Conectividade**: Ping, Traceroute, Bandwidth tests
- **Serviços**: HTTP response time, Port availability

### **Dashboards Específicos**
```
Lab Overview Dashboard:
├── Network Topology Status
├── RouterOS Performance Metrics  
├── Container Resource Usage
├── Connectivity Matrix
└── Alert Summary

Technical Dashboard:
├── Interface Traffic Graphs
├── CPU/Memory Trends
├── Network Latency Maps
├── Service Availability
└── Historical Performance
```

---

## 🧪 Cenários de Teste

### **1. Automated Configuration Deployment**
```bash
# Scenario: Deploy new VLAN via Ansible
1. Add VLAN to NetBox
2. Trigger GitLab pipeline
3. Ansible applies RouterOS configuration
4. Zabbix auto-discovers new devices
5. Verify connectivity and monitoring
```

### **2. Network Troubleshooting Simulation**
```bash
# Scenario: Interface failure simulation
1. Shutdown interface via RouterOS
2. Monitor alert generation in Zabbix
3. Trigger automatic failover (if configured)
4. Test recovery procedures
5. Document incident response
```

### **3. Capacity Planning Validation**
```bash
# Scenario: Bandwidth testing
1. Generate traffic with iperf3 between containers
2. Monitor interface utilization
3. Test QoS policies
4. Validate bandwidth limits
5. Plan capacity upgrades
```

---

## 📋 Procedimentos Operacionais

### **Startup Sequence**
1. Iniciar GNS3 Server
2. Carregar projeto de laboratório
3. Start RouterOS instances (aguardar boot)
4. Deploy Docker containers
5. Verificar conectividade end-to-end
6. Validar serviços de monitoramento

### **Shutdown Sequence**
1. Parar containers Docker
2. Save configurations RouterOS
3. Stop GNS3 instances
4. Backup project state
5. Shutdown GNS3 Server

### **Backup e Recovery**
```bash
# Backup GNS3 project
gns3server --backup /opt/gns3-projects/network-automation-lab.gns3project

# Backup RouterOS configs
./scripts/backup-routeros-configs.py

# Backup Docker volumes
docker run --rm -v gns3_data:/data -v $(pwd):/backup alpine tar czf /backup/docker-backup.tar.gz /data
```

---

## 🎯 Resultados Esperados

### **Funcionalidades Validadas**
- ✅ Descoberta automática de dispositivos
- ✅ Configuração via Ansible playbooks
- ✅ Monitoramento proativo via Zabbix
- ✅ Pipeline CI/CD funcionando
- ✅ Alertas e escalation configurados
- ✅ Dashboards operacionais

### **Métricas de Sucesso**
- **Disponibilidade**: 99%+ uptime simulado
- **Performance**: <5ms latência interna
- **Monitoramento**: 100% dispositivos descobertos
- **Automação**: 90%+ tarefas automatizadas
- **Alertas**: <1% falsos positivos

### **Benefícios do Ambiente**
- **Treinamento** sem risco à produção
- **Validação** de mudanças antes do deploy
- **Desenvolvimento** de playbooks Ansible
- **Teste** de cenários de falha
- **Demonstração** de capacidades para stakeholders