# Deploy em Produção - AWS

Este guia fornece um passo a passo para fazer o deploy do seu sistema de estacionamento em produção na AWS, usando uma **instância EC2** com **Debian 12** e **Docker Compose**. Além disso, este guia presume que você já possui uma chave SSH configurada em sua máquina local.

## Pré-requisitos

- Chave SSH configurada em sua máquina local (não obrigatório, mas recomendado).
- Chave SSH salva na AWS.
- Conta na AWS configurada.

## Etapas para o Deploy

### 1. Criar VPC e Security Group

1. No painel da AWS, acesse **VPC** e crie uma **VPC padrão** (via "VPC e muito mais").
2. Crie um **Security Group** com as seguintes permissões:
   - Porta **22** (SSH) para acesso do desenvolvedor.
   - Porta **80** (HTTP) para que o Let's Encrypt possa verificar o site e domínio.
   - Porta **443** (HTTPS) para acesso criptografado.

### 2. Criar Instância EC2

1. No painel **EC2**, crie uma nova instância.
2. Utilize as seguintes configurações:
   - **Imagem do SO**: Debian 12, arquitetura 64 bits.
   - **Tipo da instância**: t2.micro.
   - **Par de chaves**: Selecione sua chave SSH salva na AWS.
   - **Configurações de rede**:
     - Escolha a **VPC** que você criou.
     - Defina a **subnet** como pública.
     - Habilite a opção de **atribuir IP público automaticamente**.
     - Selecione o **Security Group** criado anteriormente (com as portas SSH, HTTP, e HTTPS).
   - **Armazenamento**: 16 GB.
3. Clique em **Executar instância**.

### 3. Conectar-se à Instância

Para conectar-se à sua instância **Debian 12** na AWS via SSH, utilize o seguinte comando (substitua `<ip-da-instancia>` pelo IP da sua instância):

```bash
ssh -i ~/.ssh/id_ed25519 admin@ec2-<ip-da-instancia>.us-east-2.compute.amazonaws.com
```

### 4. Instalar Docker e Docker Compose

Dentro da instância EC2, siga os passos abaixo para instalar o **Docker** e o **Docker Compose**.

#### Atualizar os pacotes e instalar dependências:

```bash
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

#### Adicionar o repositório Docker:

```bash
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

#### Instalar Docker:

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

#### Adicionar seu usuário ao grupo Docker:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

#### Verificar se o Docker foi instalado corretamente:

```bash
docker ps
```

### 5. Transferir Imagens e Docker Compose para a Instância

Compacte as imagens do projeto localmente e envie-as para a instância:

```bash
docker save -o front.tar park-front-prod:1.0
docker save -o back.tar park-back-prod:1.0
```

Depois, use **SCP** para transferir os arquivos:

```bash
scp -i ~/.ssh/id_ed25519 front.tar admin@ec2-<ip-da-instancia>.us-east-2.compute.amazonaws.com:~
scp -i ~/.ssh/id_ed25519 back.tar admin@ec2-<ip-da-instancia>.us-east-2.compute.amazonaws.com:~
scp -i ~/.ssh/id_ed25519 /<caminho-para-docker-compose.yml>/docker-compose.yml admin@ec2-<ip-da-instancia>.us-east-2.compute.amazonaws.com:~
```

### 6. Carregar as Imagens Docker na Instância

Na instância, carregue as imagens Docker:

```bash
docker load -i back.tar
docker load -i front.tar
```

### 7. Subir a Aplicação com Docker Compose

Agora, execute o **Docker Compose**:

```bash
docker compose up
```

**Nota**: Um erro pode ocorrer neste estágio, pois o **nginx.conf** precisa do certificado SSL para rodar o site em HTTPS.

### 8. Instalar o Certificado SSL com Let's Encrypt

Em um terminal separado, instale o **Certbot** para configurar o certificado SSL:

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d www.seu-dominio.com
```

Durante este processo, será solicitado um e-mail para notificações do Let's Encrypt.

Após a instalação do certificado, pare a aplicação com o comando:

```bash
docker compose down
```

### 9. Verificar e Liberar a Porta 80

Se o Nginx estiver ocupando a porta 80, pare-o:

```bash
sudo lsof -i :80
sudo systemctl stop nginx
```

Se necessário, force a parada com:

```bash
sudo killall nginx
```

### 10. Rodar a Aplicação com Docker Compose

Finalmente, inicie a aplicação novamente:

```bash
docker compose up
```

Agora a aplicação estará rodando em **HTTPS** com o certificado SSL configurado.

---

Este guia explica detalhadamente o processo de deploy na AWS para um ambiente de produção com **Docker**, **Nginx** e **Certbot**, garantindo que a aplicação rode de forma segura com **HTTPS**.
