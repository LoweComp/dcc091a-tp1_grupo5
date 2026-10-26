# Relatório Técnico - Trabalho Prático 2 (TP2): DCC091A

**Autores:** João Victor & Leandro  
**Disciplina:** DCC091A - Tópicos em Sistemas Distribuídos

---

## 1. Introdução e Visão Geral da Arquitetura
Dando continuidade ao projeto anterior, o ambiente foi expandido para uma topologia baseada em **5 Máquinas Virtuais (VMs)** provisionadas via Vagrant, separando as responsabilidades de controle administrativo, orquestração de containers (Kubernetes) e persistência de dados.

A estrutura do cluster é composta por:
* **Bastion (`192.168.56.10`)**: Atua como o nó central de gerenciamento (*Jump Server* e *Control Node* do Ansible), blindado com chaves SSH assimétricas segregadas para toda a frota.
* **Master (`192.168.56.20`)**: Nó de controle do cluster Kubernetes, executando o *Control Plane* do K3s.
* **Worker1 (`192.168.56.21`) & Worker2 (`192.168.56.22`)**: Nós de processamento do cluster, registrados dinamicamente ao *Master* através de tokens de segurança.
* **DB (`192.168.56.30`)**: Nó dedicado ao servidor de base de dados (MySQL), isolado da camada de computação para garantir integridade e segurança.

---

## 2. Automação e Gestão de Configuração (Ansible)
Para garantir a idempotência e a eliminação de processos manuais, a infraestrutura foi inteiramente provisionada através do **Ansible**, utilizando um inventário estático (`hosts`) e *playbooks* modulares (`playbook_tp2.yml` e `playbook_db.yml`).

### 2.1. Segurança e Isolamento de Acessos (Chaves SSH)
* **Arquitetura de Chaves Criptográficas (ED25519)**: O acesso direto por palavra-passe foi desativado nas instâncias base do Vagrant. O nó *Bastion* foi configurado para gerar e gerir chaves privadas individuais (`~/.ssh/keys/`) para cada máquina do ambiente.
* **Permissões Rigorosas**: Aplicação de restrições de leitura (`chmod 600`) nos ficheiros de chave privada, assegurando conformidade com os requisitos de segurança do protocolo OpenSSH.

### 2.2. Orquestração do Cluster K3s (Playbook TP2)
O *playbook* principal automatizou o ciclo de vida do Kubernetes de forma sequencial:
1. **Control Plane**: Instalação automatizada do K3s no nó *Master* e extração em tempo de execução do *token* de segurança (`/var/lib/rancher/k3s/server/node-token`).
2. **Workers**: Registo automatizado dos nós secundários (`worker1` e `worker2`) apontando diretamente para o IP do *Master* com o respectivo *token* injetado via variáveis de inventário (*hostvars*).

---

## 3. Validação do Cluster Kubernetes
A comunicação e a saúde do cluster foram validadas diretamente a partir do ecossistema Kubernetes.

* **Estado dos Nós (`kubectl get nodes`)**: O comando de validação executado no nó *Master* confirmou a integração bem-sucedida de todos os componentes com o estado **Ready** para o `master`, `worker1` e `worker2`, atestando a estabilidade da rede interna e a sincronização do *Control Plane*.
* **Resiliência e Escalabilidade**: A infraestrutura demonstrou capacidade de expansão modular, permitindo o agendamento de *pods* e serviços distribuídos pelos nós de trabalho.

---

## 4. Camada de Persistência e Base de Dados (MySQL)
A separação da camada de dados foi planejada para isolar cargas de trabalho em containers.

* **Provisionamento Automatizado**: O *playbook dedicado (`playbook_db.yml`)* executou a atualização de pacotes do sistema na máquina virtual `db`, instalando o motor **MySQL Server** em conjunto com as dependências de integração em Python (`python3-pymysql`).
* **Validação do Serviço**: A verificação do estado do serviço através de `systemctl status mysql` atestou que o motor de base de dados se encontra ativo, em execução permanente e preparado para receber conexões seguras oriundas das aplicações alojadas no cluster K3s.

---

## 5. Conclusão
A transição para a arquitetura do TP2 consolidou a aplicação prática de conceitos avançados de Infraestrutura como Código (IaC), orquestração de containers com K3s e automação via Ansible. O isolamento de serviços entre o *Bastion*, o *Control Plane*, os *Workers* e a Base de Dados resultou num ambiente altamente coeso, seguro, auditável e preparado para cenários de produção distribuídos.