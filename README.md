# 🚀 Guia Definitivo: Servidor TeamSpeak 6 com Docker no Ubuntu LTS (24/7)

Este documento contém o passo a passo 100% completo e detalhado para preparar uma instalação zerada do **Ubuntu Server LTS** (24.04 ou 22.04 LTS), instalar e configurar o **Docker** e colocar o servidor do **TeamSpeak 6 (TS6)** para rodar **24/7 com reinicialização automática**, incluindo configuração de firewall e roteador (Port Forwarding / NAT).

---

## 📑 Sumário
1. [Visão Geral de Portas e Arquitetura](#1-visão-geral-de-portas-e-arquitetura)
2. [Passo 1: Atualização e Preparação do Sistema Operacional](#passo-1-atualização-e-preparação-do-sistema-operacional)
3. [Passo 2: Instalação Oficial do Docker Engine e Docker Compose](#passo-2-instalação-oficial-do-docker-engine-e-docker-compose)
4. [Passo 3: Estruturação dos Diretórios e Arquivos do Servidor](#passo-3-estruturação-dos-diretórios-e-arquivos-do-servidor)
5. [Passo 4: Inicialização do Servidor e Chave de Administrador (Token)](#passo-4-inicialização-do-servidor-e-chave-de-administrador-token)
6. [Passo 5: Configuração do Firewall no Ubuntu (UFW)](#passo-5-configuração-do-firewall-no-ubuntu-ufw)
7. [Passo 6: Configuração de Rede e Roteador (Port Forwarding / NAT)](#passo-6-configuração-de-rede-e-roteador-port-forwarding--nat)
8. [Passo 7: Fixação do IP Local e Configuração de DDNS (Domínio Gratuito)](#passo-7-fixação-do-ip-local-e-configuração-de-ddns-domínio-gratuito)
9. [Passo 8: Rotinas de Manutenção, Atualização e Backup](#passo-8-rotinas-de-manutenção-atualização-e-backup)
10. [Resolução de Problemas Comuns (Troubleshooting)](#resolução-de-problemas-comuns-troubleshooting)

---

## 1. Visão Geral de Portas e Arquitetura

O TeamSpeak 6 utiliza uma arquitetura baseada em container oficial fornecida pela TeamSpeak Systems (`teamspeaksystems/teamspeak6-server`). As portas utilizadas pelo serviço são:

| Porta | Protocolo | Tipo | Descrição | Obrigatória para Voz? |
| :--- | :--- | :--- | :--- | :---: |
| **`9987`** | **UDP** | Entrada | Porta padrão de voz dos clientes TeamSpeak 6 | **SIM** |
| **`30033`** | **TCP** | Entrada | Porta de transferência de arquivos, avatares e ícones | **SIM** |
| `10080` | TCP | Entrada | WebQuery HTTP (APIs, integrações web, bots) | Opcional |
| `10022` | TCP | Entrada | ServerQuery SSH (Gerenciamento via terminal remoto) | Opcional |
| `9187` | TCP | Entrada | Prometheus Metrics (Coleta de métricas e monitoramento) | Opcional |

---

## Passo 1: Atualização e Preparação do Sistema Operacional

Conecte-se ao seu servidor Ubuntu via SSH ou diretamente pelo terminal físico e execute os passos abaixo.

### 1.1 Atualizar os pacotes do sistema
```bash
sudo apt update && sudo apt upgrade -y
```

### 1.2 Instalar utilitários essenciais
```bash
sudo apt install -y curl wget git nano ufw ca-certificates gnupg lsb-release htop
```

### 1.3 Configurar o Fuso Horário correto (Horário de Brasília)
```bash
sudo timedatectl set-timezone America/Sao_Paulo
```
*Para verificar se o horário está correto:*
```bash
timedatectl
```

---

## Passo 2: Instalação Oficial do Docker Engine e Docker Compose

Para garantir estabilidade, segurança e suporte ao `docker compose` v2 mais recente, instale a partir do repositório oficial da Docker Inc., em vez dos repositórios padrão do Ubuntu.

### 2.1 Configurar a chave GPG oficial do Docker
```bash
# Cria o diretório seguro para chaves de terceiros
sudo install -m 0755 -d /etc/apt/keyrings

# Faz o download da chave GPG oficial
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

### 2.2 Adicionar o repositório oficial à lista do APT
```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
```

### 2.3 Instalar o Docker Engine e componentes adicionais
```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 2.4 Habilitar o Docker para inicialização automática com o sistema (24/7)
```bash
sudo systemctl enable docker
sudo systemctl start docker
```

### 2.5 Configurar permissões de usuário (dispensar o uso contínuo de `sudo`)
```bash
sudo usermod -aG docker $USER
```

> **IMPORTANTE**: Após rodar o comando acima, encerre a sessão do terminal (`exit`) e faça login novamente para recarregar o grupo `docker`, ou aplique imediatamente com:
> ```bash
> newgrp docker
> ```

### 2.6 Validar a instalação
```bash
docker --version
docker compose version
```

---

## Passo 3: Estruturação dos Diretórios e Arquivos do Servidor

Criaremos uma estrutura em `/opt/teamspeak6` para isolar os dados da aplicação e facilitar backups e gerenciamento.

### 3.1 Criar as pastas
```bash
sudo mkdir -p /opt/teamspeak6/data
sudo chown -R $USER:$USER /opt/teamspeak6
cd /opt/teamspeak6
```

### 3.2 Criar o arquivo `docker-compose.yml`
Abra o editor:
```bash
nano docker-compose.yml
```

Cole a configuração abaixo:

```yaml
services:
  teamspeak6:
    image: teamspeaksystems/teamspeak6-server:latest
    container_name: ts6-server
    # restart: unless-stopped garante que o container inicie automaticamente após qualquer reboot ou falha
    restart: unless-stopped
    ports:
      - "9987:9987/udp"     # Porta Principal de Voz (UDP)
      - "30033:30033/tcp"   # Transferência de Arquivos / Ícones (TCP)
      # - "10080:10080/tcp" # Web Query HTTP (descomente se for usar APIs ou Bots)
      # - "10022:10022/tcp" # ServerQuery SSH (descomente se for usar Query via terminal)
    environment:
      # Aceitação obrigatória dos termos de licença do TeamSpeak 6
      - TSSERVER_LICENSE_ACCEPTED=accept
      - TSSERVER_LOG_TIMEZONE=local
    volumes:
      # Persistência permanente de dados, canais, permissões e banco SQLite
      - ./data:/var/tsserver
    logging:
      driver: "json-file"
      options:
        max-size: "15m"
        max-file: "3"
```

*Para salvar no nano:* Pressione `Ctrl + O`, aperte `Enter`, depois `Ctrl + X`.

---

## Passo 4: Inicialização do Servidor e Chave de Administrador (Token)

### 4.1 Iniciar o servidor em segundo plano
Na pasta `/opt/teamspeak6`, execute:
```bash
docker compose up -d
```
O Docker fará o download da imagem oficial e criará o container.

### 4.2 Obter a chave de privilégios de Administrador (Privilege Key)
Na **primeira inicialização**, o TeamSpeak cria o token de superusuário (`ServerAdmin`). Visualize os logs com:
```bash
docker compose logs -f
```

Você verá uma mensagem semelhante a:
```text
------------------------------------------------------------------
ServerAdmin privilege key created, please use the following key
token=xxxxxx-xxxxxx-xxxxxx-xxxxxx-xxxxxx
------------------------------------------------------------------
```

> ⚠️ **ATENÇÃO**: Salve esse token imediatamente em um local seguro. Ele será exigido na primeira vez que você se conectar pelo cliente do TeamSpeak para atribuir o cargo de **Server Admin** à sua conta.

Pressione `Ctrl + C` para fechar a visualização dos logs (o servidor continuará ativo em segundo plano).

---

## Passo 5: Configuração do Firewall no Ubuntu (UFW)

Para garantir a proteção do servidor e permitir apenas o tráfego essencial:

```bash
# 1. Libera o SSH para evitar perda de acesso remoto (porta padrão 22)
sudo ufw allow 22/tcp

# 2. Libera a porta de voz do TeamSpeak 6 (UDP)
sudo ufw allow 9987/udp

# 3. Libera a porta de transferência de arquivos do TeamSpeak 6 (TCP)
sudo ufw allow 30033/tcp

# 4. Ativa o firewall
sudo ufw enable
```
*Pressione `y` e dê `Enter` caso peça confirmação.*

Para verificar o status das regras ativas:
```bash
sudo ufw status verbose
```

---

## Passo 6: Configuração de Rede e Roteador (Port Forwarding / NAT)

Se o servidor estiver hospedado em uma rede local / doméstica (na sua casa ou escritório), o roteador precisa direcionar o tráfego externo da internet para o IP da sua máquina Ubuntu.

### 6.1 Descobrir o IP local da sua máquina Ubuntu
No terminal do Ubuntu, execute:
```bash
ip -4 addr show scope global | grep inet
```
*Exemplo de saída:* `inet 192.168.1.150/24 brd ...`  
Neste exemplo, o IP interno do Ubuntu é **`192.168.1.150`**.

### 6.2 Acessar a interface de administração do Roteador
1. Abra um navegador no mesmo ambiente de rede e acesse o endereço do gateway (geralmente `http://192.168.1.1` ou `http://192.168.0.1`).
2. Entre com o usuário e senha do seu roteador (comumente encontrados em uma etiqueta na parte traseira/inferior do aparelho).
3. Localize a seção chamada **Port Forwarding**, **Redirecionamento de Portas**, **Virtual Server**, **Servidores Virtuais** ou **NAT / Regras de Encaminhamento**.

### 6.3 Cadastrar as Regras de Redirecionamento

Crie duas regras apontando para o IP local do Ubuntu:

| Nome da Regra | Porta Externa (WAN) | Porta Interna (LAN) | Protocolo | IP Local de Destino |
| :--- | :---: | :---: | :---: | :--- |
| **TS6-Voice** | `9987` | `9987` | **UDP** | *IP do Ubuntu (ex: 192.168.1.150)* |
| **TS6-Files** | `30033` | `30033` | **TCP** | *IP do Ubuntu (ex: 192.168.1.150)* |

---

### 6.4 ⚠️ Verificação Crítica: CGNAT (Carrier-Grade NAT)

Muitos provedores de internet (especialmente fibra óptica) utilizam **CGNAT**, o que impede conexões diretas vindas da internet.

#### Como identificar:
1. Veja qual é o endereço de **IP da WAN (Internet IP)** na página principal de status do seu roteador.
2. Acesse um site como [meuip.com.br](https://www.meuip.com.br) e veja o seu IP público detectado.
3. Se o IP da WAN do seu roteador estiver na faixa de **`100.64.0.0` a `100.127.255.255`**, ou for diferente do IP detectado pelo site, **você está sob CGNAT**.

#### O que fazer se estiver em CGNAT:
- **Opção A (Recomendada):** Ligue para a sua operadora de internet e solicite a retirada de CGNAT. Diga: *"Preciso de um IP público dinâmico ou fixo para acesso a câmeras de segurança e servidor próprio"*. A maioria dos provedores libera gratuitamente ou por taxa simbólica.
- **Opção B:** Utilizar redes de overlay como **Tailscale**, **ZeroTier**, ou uma VPS de baixo custo com **Wireguard/FRP** fazendo proxy reverso do tráfego UDP.

---

## Passo 7: Fixação do IP Local e Configuração de DDNS (Domínio Gratuito)

### 7.1 Fixar o IP local no Roteador (DHCP Reservation)
Para que o roteador não mude o IP da máquina Ubuntu após reinicializações:
1. Na configuração do roteador, vá até **DHCP > Address Reservation** (Reserva de Endereço / Static Lease).
2. Adicione o endereço MAC da placa de rede do Ubuntu e associe-o ao IP fixo (ex: `192.168.1.150`).

*(Para descobrir o endereço MAC no Ubuntu, execute: `ip link`)*.

---

### 7.2 Configurar Domínio Dinâmico (DDNS) com DuckDNS

Como o IP público de conexões residenciais costuma mudar periodicamente, você pode usar o serviço gratuito [DuckDNS](https://www.duckdns.org/) para ter um endereço amigável (ex: `meuservidor.duckdns.org`).

#### Configuração passo a passo:
1. Acesse [https://www.duckdns.org](https://www.duckdns.org) e faça login (via Google, GitHub, etc.).
2. Crie um domínio (ex: `meuts6`).
3. Anote o seu **Token** exibido no topo da página.
4. No servidor Ubuntu, crie o script de atualização automática:

```bash
mkdir -p ~/duckdns
nano ~/duckdns/duck.sh
```

Cole o código abaixo (substitua `SEU_SUBDOMINIO` e `SEU_TOKEN` pelos seus dados reais):
```bash
#!/bin/bash
echo url="https://www.duckdns.org/update?domains=SEU_SUBDOMINIO&token=SEU_TOKEN&ip=" | curl -k -o ~/duckdns/duck.log -K -
```
Salve com `Ctrl + O`, `Enter` e saia com `Ctrl + X`.

Dê permissão de execução:
```bash
chmod +x ~/duckdns/duck.sh
```

Adicione uma tarefa no crontab para atualizar a cada 10 minutos:
```bash
(crontab -l 2>/dev/null; echo "*/10 * * * * ~/duckdns/duck.sh >/dev/null 2>&1") | crontab -
```

Agora seus amigos poderão conectar no TeamSpeak informando simplesmente:
👉 **`meuservidor.duckdns.org`**

---

## Passo 8: Rotinas de Manutenção, Atualização e Backup

### 8.1 Comandos do dia a dia
Navegue sempre para o diretório de instalação:
```bash
cd /opt/teamspeak6
```

- **Verificar se o servidor está rodando:**
  ```bash
  docker compose ps
  ```
- **Acompanhar logs ao vivo:**
  ```bash
  docker compose logs -f --tail=100
  ```
- **Reiniciar o servidor:**
  ```bash
  docker compose restart
  ```
- **Pausar o servidor:**
  ```bash
  docker compose stop
  ```
- **Iniciar o servidor parado:**
  ```bash
  docker compose start
  ```

---

### 8.2 Atualização da Imagem do TeamSpeak 6
Quando houver novas versões ou correções lançadas no repositório oficial do TeamSpeak 6:
```bash
cd /opt/teamspeak6

# Baixa a versão mais recente da imagem oficial
docker compose pull

# Recria o container mantendo os dados e configurações intactos
docker compose up -d
```

---

### 8.3 Rotina de Backup dos Dados
Todos os canais, permissões, usuários e bancos de dados SQLite ficam armazenados na pasta `./data`. Para criar um backup completo:

```bash
# Para o container para garantir integridade do banco de dados
cd /opt/teamspeak6
docker compose stop

# Cria um arquivo compactado com data e hora
tar -czvf ~/ts6_backup_$(date +%Y%m%d_%H%M%S).tar.gz -C /opt/teamspeak6 data docker-compose.yml

# Reativa o servidor
docker compose start
```

---

## Resolução de Problemas Comuns (Troubleshooting)

### Problema 1: "Failed to connect to server" pelos amigos
1. Verifique se o container está de pé com `docker compose ps`.
2. Verifique se a porta `9987` foi redirecionada como protocolo **UDP** no roteador (TeamSpeak não conecta via TCP na porta de voz).
3. Teste se você não está em **CGNAT** (veja a seção 6.4).
4. Verifique se o firewall do Ubuntu permitiu a porta: `sudo ufw status`.

### Problema 2: Não consigo baixar ícones ou arquivos de canal
1. Verifique se a porta `30033` está aberta no firewall e redirecionada no roteador como **TCP**.
2. Certifique-se de que o cliente TeamSpeak tem permissão para receber transferências de arquivos.

### Problema 3: O container reinicia constantemente (CrashLoop)
Verifique os logs com:
```bash
docker compose logs --tail=50
```
- Se a mensagem indicar falta de licença, garanta que `- TSSERVER_LICENSE_ACCEPTED=accept` esteja presente nas variáveis de ambiente do `docker-compose.yml`.
- Verifique se as permissões da pasta `./data` pertencem ao usuário atual ou root (`sudo chown -R 1000:1000 /opt/teamspeak6/data`).
