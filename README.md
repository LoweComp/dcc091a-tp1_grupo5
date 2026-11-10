# TP2 - DCC091A: Orquestração K3s, Redundância de Proxy (HA) e Persistência de Estado Não Destrutiva

GRUPO: JOÃO VICTOR DIAS & LEANDRO ALVARES

---

## 🏗️ 1. Topologia do Laboratório e Arquitetura de Rede

O ambiente é composto por 4 Máquinas Virtuais gerenciadas via **Vagrant** e interconectadas por uma rede Host-Only privada (`192.168.56.0/24`), com um Endereço IP Virtual (VIP) flutuante gerido por VRRP.

| Máquina Virtual | Papel no Cluster | IP Estático | Recursos (vCPU / RAM) |
| :--- | :--- | :--- | :--- |
| **VM 1 (bastion)** | Nó Ansible, Keepalived MASTER & Nginx Proxy Primário | `192.168.56.10` | 1 vCPU / 1024 MB |
| **VM 2 (k3s-server)** | K3s Control Plane, etcd & Local-Path Storage | `192.168.56.20` | 2 vCPU / 2048 MB |
| **VM 3 (k3s-worker)** | K3s Worker Node (Hospeda workloads da aplicação) | `192.168.56.30` | 2 vCPU / 2048 MB |
| **VM 4 (auxiliary)** | Keepalived BACKUP, Nginx Proxy Secundário & Backup Sync | `192.168.56.40` | 1 vCPU / 1024 MB |

* **IP Virtual Flutuante (VIP):** `192.168.56.254` (Gerido via Keepalived / VRRP).

---

## 🛠️ 2. Fases de Implementação e Tecnologias

### Fase I: Vagrant e Ciclo de Vida Não Destrutivo
* Definição declarativa multi-VM no `Vagrantfile` com recursos isolados e pastas sincronizadas (`./data_backup` -> `/var/lib/app_data`).
* Suporte completo ao ciclo de vida sem destruição (`vagrant suspend` / `vagrant resume` e `vagrant halt` / `vagrant up`), garantindo a preservação dos dados do banco e estado do cluster em menos de 60 segundos.

### Fase II: Automação com Ansible Roles Idempotentes
Organização modular em roles estruturadas:
1. `common`: Atualização, pacotes essenciais (`curl`, `dnsutils`), fuso horário e mapeamento `/etc/hosts`.
2. `proxy_ha`: Instalação do Nginx e Keepalived com templates Jinja2 e scripts de *health check*.
3. `k3s_server`: Bootstrap do control-plane K3s e extração do token de junção.
4. `k3s_agent`: Registo automatizado do worker node ao cluster.
5. `app_deploy`: Aplicação idempotente dos manifestos via `kubectl apply`.

### Fase III: Camada de Proxy Reverso Redundante (Keepalived + Nginx)
* Camada de borda com balanceamento de carga (`upstream`) distribuindo o tráfego entre a VM 2 e VM 3 na porta `30080`.
* Monitorização ativa de integridade via `check_nginx.sh`, permitindo a transição instantânea do VIP (`192.168.56.254`) para a VM de backup em caso de falha.

### Fase IV: Orquestração K3s e Aplicação Multicamadas
* **Banco de Dados (MySQL 8.0):** Implementado com Secrets para credenciais seguras e um `PersistentVolumeClaim` (PVC) de 2Gi utilizando o `local-path-provisioner` nativo do K3s.
* **Aplicação Web (Nginx Alpine):** Deployment distribuído com exposição via `NodePort` (`30080`).

---

## 🚀 3. Como Executar o Projeto

1. **Subir a infraestrutura completa:**
   ```bash
   vagrant up
   ```

2. **Validar o estado dos nós do K3s (no Control Plane):**
   ```bash
   vagrant ssh k3s-server
   sudo k3s kubectl get nodes
   ```

3. **Aplicar os Manifestos de Deploy (MySQL e Aplicação Web):**
   ```bash
   sudo k3s kubectl apply -f mysql-deployment.yaml
   sudo k3s kubectl apply -f app-deployment.yaml
   ```

---

## 🧪 4. Roteiro Prático de Verificação e Testes (Apresentação)

Estes são os testes oficiais executados para validação técnica do sistema:

### Teste A: Validação do Balanceamento e Acesso via VIP Flutuante
1. Executar consultas HTTP direcionadas ao IP Virtual do proxy:
   ```bash
   curl -I http://192.168.56.254
   ```
2. **Resultado Esperado:** Resposta `HTTP/1.1 200 OK` processada com balanceamento dinâmico entre os nós do cluster K3s.

### Teste B: Transição de Tráfego do Proxy Primário (Alta Disponibilidade)
1. Simular uma falha no Nginx do nó primário (`bastion` - VM 1):
   ```bash
   vagrant ssh bastion
   sudo systemctl stop nginx
   ```
2. **Resultado Esperado:** O Keepalived na VM 4 (`auxiliary`) deteta a falha via script de saúde e assume imediatamente o VIP `192.168.56.254`, mantendo o acesso à aplicação web ininterrupto.

### Teste C: Pausa e Retomada de Infraestrutura (`vagrant suspend` / `vagrant resume`)
1. Com dados gravados no banco persistente do cluster, suspender o ambiente:
   ```bash
   vagrant suspend
   ```
2. Retomar o ambiente:
   ```bash
   vagrant resume
   ```
3. **Resultado Esperado:** O cluster recupera o estado anterior, os Pods voltam ao estado `Running` e os dados do PVC permanecem intactos sem necessidade de reconfiguração.

### Teste D: Resiliência e Reconciliação de Pods no Cluster
1. Remover manualmente um Pod ativo da aplicação:
   ```bash
   sudo k3s kubectl delete pod <nome-do-pod-web-app>
   ```
2. **Resultado Esperado:** O controlador do Deployment no K3s recria automaticamente uma nova réplica em poucos segundos, restabelecendo a contagem desejada.

---

## 📂 Estrutura de Ficheiros do Repositório
* `Vagrantfile`: Definição declarativa das 4 VMs e pastas sincronizadas.
* `mysql-deployment.yaml`: Manifesto do Secret, PVC, Deployment e Service do MySQL.
* `app-deployment.yaml`: Manifesto do Deployment e Service (NodePort 30080) da web-app.
* `ansible/`: Playbooks e roles automatizados.
* `docs/`: Relatório técnico detalhado do projeto.