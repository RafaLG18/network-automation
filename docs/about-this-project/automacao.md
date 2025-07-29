## Introdução
Esse projeto apresenta uma solução para automação da configuração e monitoramento de equipamentos de redes, integrando algumas ferramentas como Python, Ansible, Netbox, Zabbix, e também com pipelines CI/CD, visando:
- Reduzir erros operacionais;
- Aumentar escalabilidade da infraestrutura.
- Oferecer saúde da rede em tempo real 

## Problema
O problema a ser solucionado consiste nas dificuldades encontradas em  manter, gerenciar, e implementar ativos de rede dentro de uma infraestrutura robusta, tais dificuldades consistem :
- Erros nas configurações dos equipamentos por ser gerenciamento manual.
- A escalabilidade dos equipamentos limitadas por processos manuais
- Falta de padronização nas configurações, e consequentemente, também de rastreabilidade;
- Monitoramento reativo da rede.
## Objetivo
O objetivo do projeto consiste utilizar as ferramentas descritas, para:
- Centralizar as informações de infraestrutura, com Netbox;
- Aplicar de maneira padronizada as configurações dos ativos, com Python e Ansible;
- Executar os scripts automaticamente com pipeline CI/CD, com Gitlab CI/CD;
- Monitoramento dos ativos, através de dashboards e alertas, com o Zabbix. 

## Arquitetura
![[../images/Pasted-image-20250715133028.png]]

## Funcionamento
1) Administrador registra as informações do ativo no Netbox;
2) Pipeline CI/CD executa scritps Python, e Ansible;
3) Script consulta Netbox, e gera configuração, e o Ansible aplica.
4) Zabbix consulta no netbox as informações do novo host, começa a monitorá-lo;
5) Dashboard de monitoramento fica disponível com informações do ativo
## Mapa de riscos e Ações mitigadoras
| **Risco**                                                   | **Impacto** | **Probabilidade** | **Ação Mitigadora**                                                            |
| ----------------------------------------------------------- | ----------- | ----------------- | ------------------------------------------------------------------------------ |
| Scripts aplicando configurações incorretas                  | Alto        | Média             | Ambiente de teste para validação; simulação com `--check` no Ansible           |
| Falha no pipeline GitLab CI/CD interrompendo deploy         | Médio       | Alta              | Pipeline com etapas atomizadas e rollback automático                           |
| NetBox com dados desatualizados ou incorretos               | Alto        | Média             | Validações automáticas e sincronização periódica                               |
| Zabbix sobrecarregado com muitos dispositivos monitorados   | Médio       | Média             | Balanceamento de carga (Zabbix Proxy) e tuning de performance                  |
| Falta de rastreabilidade de mudanças                        | Médio       | Alta              | Controle de versão no Git e auditoria via Ansible logs                         |
| Acesso indevido ou alteração não autorizada em dispositivos | Alto        | Média             | Autenticação forte (ex: LDAP/RADIUS), controle RBAC e logging de comandos      |
| Alertas excessivos (alert fatigue) no Zabbix                | Médio       | Alta              | Regras de correlação, thresholds ajustados e tuning de triggers                |
| Dispositivos legados sem suporte SNMP ou integração         | Baixo       | Alta              | Monitoramento alternativo por ICMP, syslog ou agentes intermediários (scripts) |
## Resultados 
O que se espera como resultado desse projeto:
- Eliminação de erros humanos
- Redução do tempo de provisionamento;
- Centralização das informações da infraestrutura.
- Monitoramento centralizado da saúde da rede. 
- Escalabilidade da infraestrutura com equipes reduzidas. 


