# 🔒 Laboratório Prático de Firewalld (com Vagrant e Ansible)

Este repositório contém o ambiente de laboratório (IaC) para o artigo: **"Para além do iptables: um guia prático de segurança para sysadmins emocionados. Parte: 2 - Firewalld"**.

O objetivo é provisionar uma VM AlmaLinux 10 usando `Vagrant` e `Ansible`, fornecendo um ambiente seguro para testar e dominar a arquitetura de **Zonas** do `firewalld`.

---

## 🚀 Como Executar o Laboratório

Estes passos irão clonar o projeto e subir a máquina virtual de testes.

### Pré-requisitos

* [Git](https://git-scm.com/downloads)
* [Vagrant](https://www.vagrantup.com/downloads)
* [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) (precisa estar instalado no seu WSL ou host)
* Um provedor de virtualização (ex: [Hyper-V](https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/quick-start/enable-hyper-v) ou [VirtualBox](https://www.virtualbox.org/wiki/Downloads))

### Passos de Execução

1.  **Clone o Repositório**
    ```bash
    git clone [https://github.com/seu-usuario/seu-repo-firewalld.git](https://github.com/seu-usuario/seu-repo-firewalld.git)
    cd seu-repo-firewalld
    ```

2.  **Suba a VM**
    O `Vagrantfile` está configurado para usar o `ansible` como provisionador. O playbook `playbook.yml` será executado automaticamente.

    * **Para Hyper-V** (em PowerShell como Admin):
        ```powershell
        vagrant up --provider=hyperv
        ```
    * **Para VirtualBox**:
        ```bash
        vagrant up
        ```

3.  **Acesse a VM**
    Uma vez que o `vagrant up` for concluído, o Ansible já terá instalado e iniciado o `firewalld`.
    ```bash
    vagrant ssh
    ```

4.  **Verifique a Instalação**
    Dentro da VM, confirme que o `firewalld` está ativo:
    ```bash
    sudo firewall-cmd --state
    # Deve retornar: running
    ```

---

## 🛡️ O Modelo de Segurança do Firewalld: Zonas

Diferente do `iptables` (uma lista única) ou `ufw` (simples), o `firewalld` usa **Zonas**. Pense em Zonas como "perfis de segurança" pré-definidos.

* O `firewalld` decide qual Zona aplicar a um pacote com base em duas regras (nesta ordem):
    1.  **Source IP:** O IP de origem pertence a uma Zona específica? (Ex: `10.0.1.0/23` -> zona `observabilidade`).
    2.  **Interface:** A interface de rede por onde o pacote chegou pertence a uma Zona? (Ex: `eth0` -> zona `public`).

* Todo o tráfego que não se encaixa em uma `source` explícita cai na Zona padrão da interface (`public`, por padrão).

### Conceito-Chave: Runtime vs. Permanent

Esta é a feature mais poderosa para um SRE/DevOps. O `firewalld` tem duas "camadas" de configuração:

1.  **Runtime (Tempo de Execução):**
    * Qualquer comando executado *sem* a flag `--permanent`.
    * **O efeito é imediato.**
    * Perfeito para testar uma regra. Se você se trancar para fora, basta reiniciar o serviço (`systemctl restart firewalld`) e as regras runtime desaparecem.

2.  **Permanent (Persistente):**
    * Qualquer comando executado *com* a flag `--permanent`.
    * **O efeito NÃO é imediato.** A regra é salva nos arquivos de configuração.
    * Para aplicar as regras permanentes ao estado runtime, você deve rodar:
        ```bash
        sudo firewall-cmd --reload
        ```
    * O `--reload` é seguro e **não derruba conexões ativas**.

---

## 📚 Guia Prático: Resumo dos Passos do Artigo

Estes são os passos executados no laboratório para construir uma política de segurança robusta.

### Passo 1: Aplicar a "Regra de Ouro" (Isolar a Zona `public`)

Por padrão, a zona `public` é muito permissiva. O primeiro passo é trancá-la, permitindo acesso SSH apenas para seu IP.

1.  **Encontre seu IP de Gerência** (o IP do seu host que acessa a VM):
    ```bash
    ip route | grep default
    # Anote o IP em "default via ..."
    ```

2.  **Adicione a Exceção de SSH (Rich Rule)**
    Substitua `SEU_IP_DEFAULT_AQUI` pelo IP que você anotou.
    ```bash
    sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="SEU_IP_DEFAULT_AQUI" service name="ssh" accept'
    ```

3.  **Remova os Serviços Padrão**
    ```bash
    sudo firewall-cmd --permanent --zone=public --remove-service=cockpit --remove-service=ssh
    ```

4.  **Defina o Alvo (Target) como DROP**
    Este é o comando que implementa a política "Negue tudo por padrão". `DROP` descarta pacotes silenciosamente.
    ```bash
    sudo firewall-cmd --permanent --zone=public --set-target=DROP
    ```

5.  **Recarregue o Firewall**
    Agora suas regras permanentes estão ativas.
    ```bash
    sudo firewall-cmd --reload
    ```

### Passo 2: Caso de Uso 1 - Servidor Web (na Zona `public`)

Como a zona `public` está ligada à interface `eth0`, ela é ideal para serviços públicos.

```bash
# Permite tráfego HTTP (porta 80)
sudo firewall-cmd --permanent --zone=public --add-service=http

# Permite tráfego HTTPS (porta 443)
sudo firewall-cmd --permanent --zone=public --add-service=https
Passo 3: Caso de Uso 2 - Zona de Observabilidade (Zabbix)
Este é um exemplo perfeito de uma Zona baseada em source.


# 1. Criar a nova zona
sudo firewall-cmd --permanent --new-zone=observabilidade

# 2. Definir a política de "negar por padrão" para esta zona
sudo firewall-cmd --permanent --zone=observabilidade --set-target=DROP

# 3. Vincular a rede do NOC/Zabbix a esta zona
sudo firewall-cmd --permanent --zone=observabilidade --add-source=10.0.1.0/23

# 4. Permitir o serviço (APENAS para esta zona/source)
sudo firewall-cmd --permanent --zone=observabilidade --add-service=zabbix-agent
Não se esqueça de rodar sudo firewall-cmd --reload para aplicar.
```

### Passo 4: Caso de Uso 3 - Zona de Administração (DevOps/SysAdmin)
Uma zona para IPs de confiança que precisam de mais acesso.

```bash

# Criar a zona
sudo firewall-cmd --permanent --new-zone=admin

# Definir o target como DROP
sudo firewall-cmd --permanent --zone=admin --set-target=DROP

# Adicionar os IPs de origem (seu IP e a rede de dev)
sudo firewall-cmd --permanent --zone=admin --add-source=172.20.160.1/32
sudo firewall-cmd --permanent --zone=admin --add-source=10.0.2.0/23

# Adicionar os serviços permitidos para eles
sudo firewall-cmd --permanent --zone=admin --add-service=ssh
sudo firewall-cmd --permanent --zone=admin --add-service=git
```

### Passo 5: Refatoração (Princípio DRY)
Agora que a zona admin cuida do seu IP, a regra de SSH na zona public é redundante. Vamos removê-la.
  ```bash
sudo firewall-cmd --permanent --zone=public --remove-rich-rule='rule family="ipv4" source address="SEU_IP_DEFAULT_AQUI" service name="ssh" accept'
  ```

  ## 🔧 Referência Rápida: "Rich Rules"

Para regras complexas, as "rich rules" são sua melhor ferramenta.

Sintaxe Geral: rule [family="..."] [source] [destination] [element] [log] [audit] [action]

Exemplos de Alto Valor:

  ```bash
# Limitar conexões (Rate Limiting) - Proteção contra Brute Force
# Permite 10 conexões por minuto para o serviço HTTPS
sudo firewall-cmd --permanent --add-rich-rule='rule service name="https" limit value="10/m" accept'

# Redirecionamento de Porta (Port Forwarding)
# Redireciona a porta externa 2222 para a porta 22 do IP 10.0.1.10
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" forward-port port="2222" protocol="tcp" to-port="22" to-addr="10.0.1.10"'

# Logar Regras Específicas
# Permite SSH da rede de dev e loga com um prefixo
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.2.0/23" service name="ssh" log prefix="SSH_Dev" level="info" accept'

# Bloquear um IP Malicioso
# Descarta silenciosamente (DROP) todo o tráfego de um IP
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="123.123.123.123" drop'
  ```

## 🚨 Modo Pânico (O "Kill Switch")

Se você suspeitar de uma intrusão, este comando bloqueia TODO o tráfego de rede (exceto localhost), derrubando conexões existentes.
 
  ```bash
# Ativa o bloqueio total
sudo firewall-cmd --panic-on

# Desativa o bloqueio e restaura as regras normais
sudo firewall-cmd --panic-off

# Verifica se o modo pânico está ativo
sudo firewall-cmd --query-panic
  ```

---

## Autor do Artigo

* **Valdemir Bezerra de Souza Júnior**
* Analista Infraestrutura | Devops | SRE | Cloud | Oracle Cloud | Linux | Docker | Kubernets | Python | Go | Rust | Lua | N8N | No Code
* [Link para o Artigo Original]([https://www.linkedin.com/pulse/para-al%C3%A9m-do-iptables-um-guia-pr%C3%A1tico-de-seguran%C3%A7a-1-valdemir-jhudf/?trackingId=PYbnZYC4Q8yfZIA5wpxmdA%3D%3D](https://www.linkedin.com/pulse/para-al%25C3%25A9m-do-iptables-um-guia-pr%25C3%25A1tico-de-seguran%25C3%25A7a-2-valdemir-lfrgf/))


