# Relatório Técnico - Trabalho Prático 1 (TP1): DCC091A

**Autores:** João Victor & Leandro  
**Disciplina:** DCC091A - Tópicos em Sistemas Distribuídos

---

## 1. Introdução e Visão Geral da Arquitetura
O presente relatório documenta a concepção, automação e validação do ambiente de infraestrutura desenvolvido para o TP1. O projeto emprega uma topologia baseada em **4 Máquinas Virtuais (VMs)** provisionadas via Vagrant, separando as responsabilidades de controle, processamento web e persistência de dados.

A estrutura do cluster é composta por:
* **Bastion (`192.168.56.10`)**: Atua como nó de gerenciamento e controle, além de funcionar como **Proxy Reverso e Balanceador de Carga** para o tráfego web.
* **Web1 (`192.168.56.11`) & Web2 (`192.168.56.12`)**: Nós de servidores web executando Nginx, responsáveis por responder às requisições HTTP direcionadas pelo proxy.
* **DB (`192.168.56.20`)**: Nó de banco de dados configurado com diretórios sincronizados para garantir a persistência física dos dados.

---

## 2. Automação e Gerenciamento de Configuração (Ansible)
Para eliminar a configuração manual e garantir a idempotência do ambiente, todo o gerenciamento de configuração foi automatizado utilizando o **Ansible**, estruturado em um inventário estático (`inventory.ini`) e um playbook central (`site.yml`).

### 2.1. Hardening de Segurança (SSH)

* **Desativação de Autenticação por Senha**: O parâmetro `PasswordAuthentication no` foi aplicado no arquivo `/etc/ssh/sshd_config`, obrigando o uso exclusivo de chaves SSH (ED25519) distribuídas a partir do nó Bastion.
* **Bloqueio de Login Direto do Root**: O parâmetro `PermitRootLogin no` foi configurado para impedir acessos administrativos diretos ao usuário `root`, mitigando vetores de ataque por força bruta.
* **Handlers Automatizados**: Alterações nas configurações de segurança disparam um *handler* que reinicia de forma segura o serviço SSH (`ssh`).

### 2.2. Firewall e Controle de Acesso (UFW)
Para garantir o isolamento da rede e o controle rigoroso do tráfego, foi implementado o Uncomplicated Firewall (UFW):

* **Política de Bloqueio Padrão**: Ativação do UFW em todos os nós do cluster com a política de entrada padrão definida para `deny`.
* **Exceções Críticas de Rede**: Criação de regras de exceção (`allow`) aplicadas de forma estritamente sequencial *antes* da ativação final do firewall. Isso garantiu a disponibilidade contínua das portas **22 (SSH)** para gerência via Ansible, além das portas **80 (HTTP)** e **443 (HTTPS)** para o tráfego web.

---

## 3. Alta Disponibilidade e Balanceamento de Carga (Proxy Reverso)
O roteamento de tráfego foi centralizado no nó **Bastion**, que atua como ponto de entrada único e seguro para a infraestrutura HTTP.

* **Distribuição de Carga (`upstream`)**: Utilizando a diretiva `upstream` do Nginx no Bastion, as requisições externas que chegam à porta 80 são distribuídas dinamicamente e de forma balanceada entre os servidores web (`web1` no IP 192.168.56.11 e `web2` no IP 192.168.56.12).
* **Identificação de Nós e Validação**: Para fins de auditoria, cada nó web possui uma página HTML personalizada que renderiza o seu respectivo `inventory_hostname`. A arquitetura foi testada exaustivamente via comando `curl` direcionado ao Bastion, confirmando a alternância visual das respostas entre `web1` e `web2` sem a exposição direta das portas dos *webservers* ao meio externo.

---

## 4. Persistência de Dados e Tolerância a Falhas
A camada de dados do projeto foi projetada para garantir que informações críticas sobrevivam a pane sistêmicas ou destruições acidentais de instâncias virtuais.

* **Diretórios Sincronizados (Shared Folders)**: O arquivo `Vagrantfile` mapeia uma pasta física do host Windows para o diretório `/var/lib/dados_persistentes` na máquina virtual `db`.
* **Teste de Sobrevivência (Survival Test)**: O procedimento de validação consistiu em:
  1. Gravar dados simulados de banco de dados no diretório persistente.
  2. Destruir completamente a VM de banco de dados utilizando o comando `vagrant destroy -f db`.
  3. Reconstruir a infraestrutura com `vagrant up db`.
  4. Constatar que os arquivos gravados anteriormente permanecem intactos devido à retenção no sistema de arquivos do host.

---

## 5. Conclusão
A combinação de Vagrant, Ansible, Nginx e UFW proporcionou um fluxo de entrega contínua, seguro, auditável e altamente resiliente a falhas de infraestrutura, consolidando os conceitos de Infraestrutura como Código (IaC) na prática.