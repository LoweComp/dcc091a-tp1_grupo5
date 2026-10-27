# TP2 - DCC091A: Orquestração de Containers (K3s) e Automação de Infraestrutura

Repositório dedicado ao Trabalho Prático 2 (TP2) da disciplina **DCC091A** por João Victor Dias e Leandro Alvares.

---

## 🏗️ Arquitetura da Infraestrutura

O ambiente evoluiu para uma topologia de **5 Máquinas Virtuais (VMs)** provisionadas via Vagrant, com separação estrita de responsabilidades, autenticação baseada em chaves SSH isoladas e orquestração de contentores:

1. **Bastion (`192.168.56.10`)**: Nó de controle (executa o Ansible) e atua como *Jump Server* blindado para acesso às restantes máquinas.
2. **Master (`192.168.56.20`)**: Nó de controle do cluster Kubernetes, executando o *Control Plane* do K3s.
3. **Worker1 (`192.168.56.21`)**: Nó de processamento do cluster (K3s Agent).
4. **Worker2 (`192.168.56.22`)**: Nó de processamento do cluster (K3s Agent).
5. **DB (`192.168.56.30`)**: Servidor de base de dados (MySQL) completamente isolado da camada de computação.

---

## 📁 Estrutura do Repositório
```text
dcc091a-tp1/
├── vagrant/
│   └── Vagrantfile             # Definição e mapeamento de rede das 5 VMs
├── ansible/
│   ├── hosts                   # Inventário com apontamento de chaves SSH dinâmicas
│   ├── playbook_tp2.yml        # Playbook de orquestração do cluster K3s
│   └── playbook_db.yml         # Playbook de instalação do MySQL Server
├── docs/
│   ├── TP1/                    # Arquivo de documentação do Trabalho Prático 1
│   └── TP2/                    # Relatório técnico e prints de evidência do TP2
└── README.md                   # Guia mestre de implantação e uso
```

---

## ⚙️ Guia de Instalação e Execução

### 1. Subir a Infraestrutura Base
No terminal da sua máquina hospedeira, navegue até à pasta do Vagrant e inicie as VMs:
```bash
cd dcc091a-tp1/vagrant
vagrant up
```

### 2. Aceder ao Bastion e Blindar Chaves SSH
Aceda ao nó de controle e prepare o ambiente copiando e protegendo as chaves geradas pelo Vagrant:
```bash
vagrant ssh bastion
```
Dentro do Bastion, execute o bloco abaixo para isolar as credenciais:
```bash
mkdir -p ~/.ssh/keys
cp /vagrant/.vagrant/machines/db/virtualbox/private_key ~/.ssh/keys/db_key
cp /vagrant/.vagrant/machines/master/virtualbox/private_key ~/.ssh/keys/master_key
cp /vagrant/.vagrant/machines/worker1/virtualbox/private_key ~/.ssh/keys/worker1_key
cp /vagrant/.vagrant/machines/worker2/virtualbox/private_key ~/.ssh/keys/worker2_key
chmod 600 ~/.ssh/keys/*
```

### 3. Executar a Automação via Ansible
Navegue para o diretório partilhado do Ansible e dispare os *playbooks*:
```bash
cd /ansible

# Orquestrar o cluster Kubernetes (Master + Workers)
ansible-playbook -i hosts playbook_tp2.yml

# Configurar o servidor de Base de Dados (MySQL)
ansible-playbook -i hosts playbook_db.yml
```

---

## 🧪 Testes de Validação

### 1. Validar Cluster K3s (Nó Master)

Para garantir que o Kubernetes está a funcionar corretamente e que os *Workers* estão ligados ao *Master*:

```bash
# A partir do terminal do Bastion, aceda ao Master:
ssh -i ~/.ssh/keys/master_key vagrant@192.168.56.20

# Liste o estado dos nós do cluster:
sudo kubectl get nodes
```

*Resultado esperado:* Os três nós (master, worker1, worker2) devem apresentar o estado **Ready**.

### 2. Validar Base de Dados (Nó DB)

Para confirmar que o servidor MySQL está ativo e pronto a receber ligações:

```bash
# A partir do terminal do Bastion, aceda ao DB:
ssh -i ~/.ssh/keys/db_key vagrant@192.168.56.30

# Verifique o estado do serviço:
sudo systemctl status mysql
```

*Resultado esperado:* O serviço deve estar assinalado como **active (running)**.
