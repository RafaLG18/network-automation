# Zabbix - Configuração e Integração com NetBox

## 🎯 Pré-requisitos: NetBox como Fonte da Verdade

### **Nível de Maturidade Mínimo do NetBox**

Para que o NetBox seja considerado a **fonte única da verdade** e possa alimentar automaticamente o Zabbix, deve atender aos seguintes critérios:

#### **1. Completude dos Dados (95% dos ativos)**
- **Todos os dispositivos ativos** cadastrados com informações básicas
- **IPs de gerência** configurados e testados
- **Localização física** completa (Site → Rack → Posição)
- **Tipo de dispositivo** e fabricante definidos
- **Status operacional** atualizado

#### **2. Qualidade da Informação**
- **Dados validados** e testados (ping, SNMP walk)
- **Nomenclatura padronizada** seguindo convenções estabelecidas
- **Informações consistentes** sem duplicatas ou conflitos
- **Metadados organizacionais** (tags, custom fields) preenchidos
- **Relacionamentos** entre dispositivos mapeados

#### **3. Automação e Integridade**
- **API NetBox** funcional e acessível
- **Sincronização automática** com descoberta de rede
- **Validação de dados** implementada (scripts de verificação)
- **Versionamento** de mudanças com auditoria
- **Backup automatizado** dos dados

#### **4. Governança de Dados**
- **Responsáveis definidos** para manutenção dos dados
- **Processos estabelecidos** para inclusão/alteração de ativos
- **SLA de atualização** definido (ex: 24h para novos dispositivos)
- **Métricas de qualidade** implementadas e monitoradas

### **Checklist de Validação NetBox**
```
□ 95%+ dispositivos cadastrados com IP de gerência
□ 100% dispositivos críticos documentados
□ Ping test passa em 90%+ dos IPs de gerência  
□ SNMP walk funciona em 85%+ dos dispositivos
□ Dados sincronizados há menos de 24h
□ Zero duplicatas de dispositivos/IPs
□ Taxonomia de tags implementada
□ API responde em <2 segundos
□ Backup diário funcionando
□ Processo de change management ativo
```

---

## 🏗️ Requisitos de Infraestrutura do Zabbix

### **Servidor Zabbix**
- **CPU**: 4+ cores (8+ para >1000 dispositivos)
- **RAM**: 8GB mínimo (16GB+ recomendado)
- **Storage**: 
  - Sistema: 20GB SSD
  - Dados/History: 100GB+ SSD (crescimento 1GB/mês por 100 devices)
  - Logs: 50GB adicional
- **OS**: Ubuntu 20.04+ LTS, CentOS 8+, ou RHEL 8+

### **Banco de Dados**
- **MySQL 8.0+** ou **PostgreSQL 13+** (recomendado)
- **Servidor dedicado** para >500 dispositivos
- **InnoDB** engine com otimizações específicas
- **Particionamento** de tabelas history/trends

### **Zabbix Proxy (Para Sites Remotos)**
- **CPU**: 2+ cores por site remoto
- **RAM**: 2GB mínimo
- **Conectividade**: Túnel VPN estável para servidor central
- **Função**: Coleta local + envio batch para servidor

### **Componentes Adicionais**
- **Zabbix Agent 2** nos servidores monitorados
- **SNMP Daemon** para discovery automático
- **Grafana** (opcional) para dashboards avançados
- **Elasticsearch** (opcional) para logs centralizados

---

## 🔧 Configuração do Zabbix via Docker

### **1. Pré-requisitos do Sistema**

#### **Docker Installation**
```bash
# Ubuntu/Debian
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
sudo usermod -aG docker $USER

# Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.21.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

#### **Sistema Host**
```bash
# Criar estrutura de diretórios
mkdir -p /opt/zabbix/{mysql,grafana,scripts}
cd /opt/zabbix

# Ajustar limites do sistema
echo "vm.max_map_count=262144" >> /etc/sysctl.conf
sysctl -p
```

### **2. Docker Compose Stack**

#### **Arquivo docker-compose.yml**
```yaml
version: '3.8'

services:
  # MySQL Database
  mysql-server:
    image: mysql:8.0-oracle
    container_name: zabbix-mysql
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: zabbix_password
      MYSQL_ROOT_PASSWORD: root_password
    command:
      - mysqld
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_bin
      - --skip-character-set-client-handshake
      - --default-authentication-plugin=mysql_native_password
      - --innodb-buffer-pool-size=2G
      - --max-connections=1000
    volumes:
      - mysql_data:/var/lib/mysql
      - ./mysql/conf.d:/etc/mysql/conf.d:ro
    networks:
      - zabbix-net
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      timeout: 20s
      retries: 10

  # Zabbix Server
  zabbix-server:
    image: zabbix/zabbix-server-mysql:alpine-6.4-latest
    container_name: zabbix-server
    restart: unless-stopped
    environment:
      DB_SERVER_HOST: mysql-server
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: zabbix_password
      MYSQL_ROOT_PASSWORD: root_password
      ZBX_STARTPOLLERS: 30
      ZBX_STARTPOLLERSUNREACHABLE: 5
      ZBX_STARTTRAPPERS: 10
      ZBX_STARTPINGERS: 10
      ZBX_STARTDISCOVERERS: 5
      ZBX_CACHESIZE: 128M
      ZBX_HISTORYCACHESIZE: 64M
      ZBX_HISTORYINDEXCACHESIZE: 32M
      ZBX_TRENDCACHESIZE: 32M
      ZBX_VALUECACHESIZE: 256M
      ZBX_TIMEOUT: 10
    ports:
      - "10051:10051"
    volumes:
      - zabbix_data:/var/lib/zabbix
      - ./scripts:/usr/lib/zabbix/externalscripts:ro
    networks:
      - zabbix-net
    depends_on:
      mysql-server:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "zabbix_server", "-R", "config_cache_reload"]
      timeout: 10s
      retries: 5

  # Zabbix Web Frontend
  zabbix-web:
    image: zabbix/zabbix-web-nginx-mysql:alpine-6.4-latest
    container_name: zabbix-web
    restart: unless-stopped
    environment:
      ZBX_SERVER_HOST: zabbix-server
      DB_SERVER_HOST: mysql-server
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: zabbix_password
      MYSQL_ROOT_PASSWORD: root_password
      PHP_TZ: America/Sao_Paulo
      ZBX_SERVER_NAME: "Zabbix Network Monitor"
    ports:
      - "80:8080"
      - "443:8443"
    volumes:
      - zabbix_web:/etc/ssl/nginx
    networks:
      - zabbix-net
    depends_on:
      - mysql-server
      - zabbix-server
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/"]
      timeout: 10s
      retries: 5

  # Zabbix Agent (para monitorar o próprio servidor)
  zabbix-agent:
    image: zabbix/zabbix-agent2:alpine-6.4-latest
    container_name: zabbix-agent
    restart: unless-stopped
    environment:
      ZBX_HOSTNAME: "Zabbix-Server"
      ZBX_SERVER_HOST: zabbix-server
      ZBX_SERVER_PORT: 10051
      ZBX_ACTIVE_ALLOW: true
    ports:
      - "10050:10050"
    networks:
      - zabbix-net
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /dev:/host/dev:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    privileged: true
    pid: host

  # Grafana (Dashboard Avançado)
  grafana:
    image: grafana/grafana:10.1.0
    container_name: zabbix-grafana
    restart: unless-stopped
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin_password
      GF_INSTALL_PLUGINS: alexanderzobnin-zabbix-app
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources
    networks:
      - zabbix-net

volumes:
  mysql_data:
    driver: local
  zabbix_data:
    driver: local
  zabbix_web:
    driver: local
  grafana_data:
    driver: local

networks:
  zabbix-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

### **3. Configurações Personalizadas**

#### **MySQL Configuration (mysql/conf.d/custom.cnf)**
```ini
[mysqld]
# Performance Tuning
innodb_buffer_pool_size = 2G
innodb_log_file_size = 256M
innodb_log_buffer_size = 16M
innodb_flush_log_at_trx_commit = 2
innodb_flush_method = O_DIRECT
max_connections = 1000
query_cache_size = 0
query_cache_type = 0

# Zabbix Specific
sql_mode = ''
```

#### **Grafana DataSource (grafana/datasources/zabbix.yml)**
```yaml
apiVersion: 1
datasources:
  - name: Zabbix
    type: alexanderzobnin-zabbix-datasource
    access: proxy
    url: http://zabbix-web:8080/api_jsonrpc.php
    jsonData:
      username: Admin
      password: zabbix
      trends: true
      trendsFrom: '7d'
      trendsRange: '4d'
      cacheTTL: '1h'
    isDefault: true
```

### **4. Deploy e Inicialização**

#### **Comando de Deploy**
```bash
# Deploy da stack completa
cd /opt/zabbix
docker-compose up -d

# Verificar status dos containers
docker-compose ps

# Acompanhar logs
docker-compose logs -f zabbix-server
```

#### **Configuração Inicial**
```bash
# Aguardar inicialização completa (2-3 minutos)
docker-compose logs zabbix-server | grep "server #0 started"

# Acesso via web: http://servidor:80
# Login padrão: Admin / zabbix
```

### **4. Integração com NetBox**

#### **Script de Descoberta Automática**
```python
#!/usr/bin/env python3
"""
Zabbix Auto-Discovery via NetBox API
Sincroniza dispositivos do NetBox para Zabbix
"""

import requests
import json
from pyzabbix import ZabbixAPI

# Configurações
NETBOX_URL = "https://netbox.empresa.com"
NETBOX_TOKEN = "your_netbox_api_token"
ZABBIX_URL = "https://zabbix.empresa.com"
ZABBIX_USER = "admin"
ZABBIX_PASS = "zabbix_password"

def get_netbox_devices():
    """Busca dispositivos ativos no NetBox"""
    headers = {
        'Authorization': f'Token {NETBOX_TOKEN}',
        'Content-Type': 'application/json'
    }
    
    response = requests.get(
        f"{NETBOX_URL}/api/dcim/devices/",
        headers=headers,
        params={'status': 'active', 'limit': 1000}
    )
    
    return response.json()['results']

def sync_to_zabbix(devices):
    """Sincroniza dispositivos para Zabbix"""
    zapi = ZabbixAPI(ZABBIX_URL)
    zapi.login(ZABBIX_USER, ZABBIX_PASS)
    
    for device in devices:
        if device['primary_ip']:
            ip_address = device['primary_ip']['address'].split('/')[0]
            
            # Criar/atualizar host no Zabbix
            host_data = {
                'host': device['name'],
                'name': device['display_name'],
                'interfaces': [{
                    'type': 2,  # SNMP
                    'main': 1,
                    'useip': 1,
                    'ip': ip_address,
                    'dns': '',
                    'port': '161'
                }],
                'groups': [{'groupid': '2'}],  # Linux servers
                'templates': [{'templateid': '10001'}],  # Template Net Generic Device SNMPv2
                'inventory_mode': 1,
                'inventory': {
                    'location': f"{device['site']['name']} - {device['location']['name']}",
                    'model': device['device_type']['model'],
                    'vendor': device['device_type']['manufacturer']['name'],
                    'serialno_a': device['serial']
                }
            }
            
            try:
                zapi.host.create(host_data)
                print(f"Host {device['name']} criado com sucesso")
            except Exception as e:
                if "already exists" in str(e):
                    # Atualizar host existente
                    zapi.host.update({
                        'hostid': zapi.host.get(filter={'host': device['name']})[0]['hostid'],
                        **host_data
                    })
                    print(f"Host {device['name']} atualizado")
                else:
                    print(f"Erro ao criar {device['name']}: {e}")

if __name__ == "__main__":
    devices = get_netbox_devices()
    sync_to_zabbix(devices)
```

### **5. Templates de Monitoramento**

#### **Template por Tipo de Dispositivo**
- **Network Generic SNMP**: Switches/Roteadores básicos
- **Cisco Catalyst SNMP**: Switches Cisco específicos  
- **Linux Server**: Servidores com Zabbix Agent
- **Windows Server**: Servidores Windows
- **UPS SNMP**: No-breaks com interface SNMP
- **Firewall SNMP**: Firewalls e appliances de segurança

#### **Métricas Essenciais por Template**
```
Network Devices:
- Interface utilization (in/out)
- CPU usage
- Memory usage  
- Temperature sensors
- Power supply status
- Port status (up/down)

Servers:
- CPU load average
- Memory utilization
- Disk space usage
- Network interfaces
- Process monitoring
- Log file monitoring
```

---

## 📊 Configuração de Monitoramento

### **1. Host Groups Organizacionais**
```
Infraestrutura/
├── Core Network/
├── Distribution/
├── Access Switches/
├── Firewalls/
├── Servidores/
│   ├── Produção/
│   ├── Desenvolvimento/
│   └── Backup/
└── Sites Remotos/
    ├── Filial-SP/
    ├── Filial-RJ/
    └── Filial-BH/
```

### **2. Discovery Rules Automáticas**
- **Network Discovery**: Range de IPs por subnet
- **SNMP Discovery**: Auto-detecção de interfaces
- **File System Discovery**: Partições de disco
- **Process Discovery**: Serviços críticos

### **3. Triggers e Alertas**

#### **Criticidade dos Alertas**
- **Disaster**: Falha total de serviço crítico
- **High**: Degradação severa de performance  
- **Average**: Problemas que requerem atenção
- **Warning**: Situações de monitoramento preventivo
- **Information**: Eventos informativos

#### **Escalation Matrix**
```
Disaster → SMS + Email → NOC (imediato)
High → Email → Técnico responsável (5min)
Average → Email → Equipe (15min)
Warning → Dashboard → Revisão diária
```

---

## 🔗 Automação e Integração

### **1. Synchronization Schedule**
```bash
# Crontab para sincronização NetBox → Zabbix
# Executa a cada 4 horas
0 */4 * * * /opt/scripts/netbox-zabbix-sync.py

# Limpeza de hosts inativos (semanal)
0 2 * * 0 /opt/scripts/cleanup-inactive-hosts.py
```

### **2. API Integration**
- **NetBox API**: Source of truth para metadados
- **Zabbix API**: Criação/atualização automática de hosts
- **GitLab API**: Trigger de sync após mudanças no NetBox
- **Webhook**: Notificações em tempo real de mudanças

### **3. Dashboards Personalizados**
```
Executive Dashboard:
- Network availability overview
- Critical alerts summary  
- Capacity utilization trends
- Site comparison metrics

Technical Dashboard:
- Real-time performance graphs
- Alert correlation analysis
- Historical trend analysis
- Troubleshooting views
```

---

## 📋 Procedimentos Operacionais

### **1. Processo de Onboarding de Novos Dispositivos**
1. **NetBox**: Cadastrar dispositivo com metadados completos
2. **Validação**: Ping/SNMP test via script automático
3. **Zabbix Sync**: Execução do script de sincronização
4. **Template**: Associação automática baseada no device type
5. **Verificação**: Confirmação de coleta de dados funcionando

### **2. Manutenção e Troubleshooting**
- **Health Check Diário**: Script de validação da integridade
- **Performance Tuning**: Otimização baseada em métricas
- **Log Analysis**: Monitoramento de erros e warnings
- **Capacity Planning**: Análise de crescimento e dimensionamento

### **3. Disaster Recovery**
- **Backup Zabbix**: Database + configuration files
- **Backup NetBox**: Database + media files  
- **Documentação**: Procedimentos de restore
- **Testes**: Simulação de recovery mensalmente

---

## 🎯 Métricas de Sucesso

### **KPIs de Implementação**
- **Cobertura**: 95%+ dispositivos monitorados
- **Disponibilidade**: 99.5%+ uptime do Zabbix  
- **Performance**: <10s response time para dashboards
- **Alertas**: <5% false positives
- **Sincronização**: 100% devices do NetBox no Zabbix

### **Benefícios Esperados**
- **Detecção Proativa**: Redução 80% tempo de identificação de problemas
- **MTTR**: Diminuição 50% tempo médio de resolução
- **Disponibilidade**: Aumento 99.9%+ uptime da infraestrutura
- **Eficiência**: Redução 60% trabalho manual de monitoramento