# TP1 - DCC091A: Automação de Infraestrutura, Segurança e Persistência de Dados

Repositório dedicado ao Trabalho Prático 1 (TP1) da disciplina **DCC091A** por João Victor Dias e Leandro Alvares

---

## 🏗️ Arquitetura da Infraestrutura

O ambiente é composto por **4 Máquinas Virtuais (VMs)** provisionadas via Vagrant e gerenciadas por um nó de controle central:

1. **Bastion (`192.168.56.10`)**: Nó de controle (executa o Ansible) e atua como Proxy Reverso / Balanceador de Carga (Nginx) para o cluster web.
2. **Web1 (`192.168.56.11`)**: Servidor web executando Nginx com identificação customizada do nó.
3. **Web2 (`192.168.56.12`)**: Servidor web executando Nginx com identificação customizada do nó.
4. **DB (`192.168.56.20`)**: Servidor de banco de dados estruturado com pasta compartilhada para garantir persistência a nível de host.

---

## 📁 Estrutura do Repositório
```
dcc091a-tp1/
├── vagrant/
│   ├── Vagrantfile             # Configuração das 4 VMs e diretórios sincronizados
│   └── ...
├── ansible/
│   ├── inventory.ini           # Inventário estático dos hosts do cluster
│   └── site.yml                # Playbook central (Hardening, Nginx e Proxy Reverso)
├── docs/
│   └── relatorio_tecnico.md    # Relatório detalhado das decisões de projeto
├── scripts/
│   └── ...                     # Utilitários de validação (opcional)
└── README.md                   # Guia mestre de implantação e uso
```
## 🚀 Pré-requisitos

Antes de iniciar, certifique-se de ter instalado em sua máquina host:
* [VirtualBox](https://www.virtualbox.org/)
* [Vagrant](https://www.vagrantup.com/)
* [Git](https://git-scm.com/)

---

## ⚙️ Guia de Instalação e Execução

1. **Entrar na Pasta Correta:**
   ```bash
   cd dcc091a-tp1/vagrant

2. **Acessar Nó de Controle:**
   ```bash
   vagrant up

3. **Executar os Playbooks do Ansible:**
   ```bash
   ansible-playbook -i /home/vagrant/ansible/inventory.ini /home/vagrant/ansible/site.yml


---

## 🧪 Testes de Validação

1. **Teste de Conectividade (Ansible Ping):**
   ```bash
   ansible all -i /home/vagrant/ansible/inventory.ini -m ping

2. **Teste de Balanceamento de Carga (Proxy Reverso):**
   ```bash
   curl [http://192.168.56.10](http://192.168.56.10)

3. **Teste de Sobrevivência (Persistência de Dados):**
   ```text
   Acesse a VM db e crie um arquivo dentro da pasta de dados persistentes.
   Destrua e recrie a máquina (vagrant destroy -f db seguido de vagrant up db).
   Verifique que os dados continuam intactos.
