---

# Dockerとn8nのセットアップガイド

このガイドは、EC2インスタンスのUbuntu24.04上でDockerとn8nをセットアップし、再起動後に自動的に起動するように設定するためのステップバイステップの手順を示しています。

## 前提条件

- EC2インスタンスでUbuntu24.04を開始していてClIを操作できる状態であること
- EC2上でSSH,80,443,5678のTCPポートを0.0.0.0/0（任意の場所）からアクセスできるようにセキュリティグループを設定しておくこと
- ドメインを取得すること
- EC2のパブリック IPv4 DNS　（ec2-xx-xx-xxx-xxx.ap-northeast-1.compute.amazonaws.com#こんな感じ）をサブドメイン（一般的にはn8n.xxxx.com）にCNAMEで紐づけておくこと


## ステップ1: 古いDocker関連パッケージの削除

以下のコマンドを実行して、古いDocker関連パッケージを削除します。

```sh
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove -y $pkg; done
```

## ステップ2: Dockerリポジトリの設定とインストール

1. 必要なパッケージのインストール

```sh
sudo apt-get update
sudo apt-get install -y ca-certificates curl
```

2. DockerのGPGキーを追加

```sh
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

3. Dockerリポジトリの追加

```sh
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

4. Dockerと関連パッケージのインストール

最新バージョンのインストールコマンドを入れます。
```sh
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

5. Dockerの動作確認

```sh
sudo docker run hello-world
```

####ここまでの参考
https://docs.docker.com/engine/install/ubuntu/


## ステップ3: ユーザーをDockerグループに追加

```sh
sudo usermod -aG docker ${USER}
```

この変更を反映するために、再ログインするか次のコマンドを実行します：

```sh
su - ${USER}　#なんか知らんがこれ動かないし、動かなくても動いた。
```

## ステップ4: n8nとTraefikのセットアップ

1. ディレクトリとボリュームの作成

```sh
mkdir ~/n8n #　n8nのdocker設定などを保管する場所として作成。
cd ~/n8n
sudo docker volume create n8n_data
sudo docker volume create traefik_data
```

2. `docker-compose.yml`ファイルの作成

```yaml
version: "3.7"

services:
  traefik:
    image: "traefik"
    restart: always
    command:
      - "--api=true"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.mytlschallenge.acme.tlschallenge=true"
      - "--certificatesresolvers.mytlschallenge.acme.email=${SSL_EMAIL}"
      - "--certificatesresolvers.mytlschallenge.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - traefik_data:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock:ro

  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    ports:
      - "127.0.0.1:5678:5678"
    labels:
      - traefik.enable=true
      - traefik.http.routers.n8n.rule=Host(`${SUBDOMAIN}.${DOMAIN_NAME}`)
      - traefik.http.routers.n8n.tls=true
      - traefik.http.routers.n8n.entrypoints=web,websecure
      - traefik.http.routers.n8n.tls.certresolver=mytlschallenge
      - traefik.http.middlewares.n8n.headers.SSLRedirect=true
      - traefik.http.middlewares.n8n.headers.STSSeconds=315360000
      - traefik.http.middlewares.n8n.headers.browserXSSFilter=true
      - traefik.http.middlewares.n8n.headers.contentTypeNosniff=true
      - traefik.http.middlewares.n8n.headers.forceSTSHeader=true
      - traefik.http.middlewares.n8n.headers.SSLHost=${DOMAIN_NAME}
      - traefik.http.middlewares.n8n.headers.STSIncludeSubdomains=true
      - traefik.http.middlewares.n8n.headers.STSPreload=true
      - traefik.http.routers.n8n.middlewares=n8n@docker
    environment:
      - N8N_HOST=${SUBDOMAIN}.${DOMAIN_NAME}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - NODE_ENV=production
      - WEBHOOK_URL=https://${SUBDOMAIN}.${DOMAIN_NAME}/
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  traefik_data:
    external: true
  n8n_data:
    external: true
```

3. `.env`ファイルの作成
所得したドメイン名とそのサブドメイン。Let’s EncryptのACME証明書取得に使用するメールアドレスを好きなEmailアドレスに書き換えて実行してください。

```env
# The top level domain to serve from
DOMAIN_NAME=revol-one.com #ここは都度書き換えましょう。事前に設定しておかないと設定時に名前解決できずSSLエラーになります

# The subdomain to serve from
SUBDOMAIN=n8n　 #ここは都度書き換えましょう

# DOMAIN_NAME and SUBDOMAIN combined decide where n8n will be reachable from
# above example would result in: https://n8n.example.com

# Optional timezone to set which gets used by Cron-Node by default
# If not set New York time will be used
GENERIC_TIMEZONE=Asia/Tokyo

# The email address to use for the SSL certificate creation
SSL_EMAIL=hori@revol.co.jp #ここは都度書き換えましょう。。Let’s Encryptで利用します。
```

4. Docker Composeを使用してn8nとTraefikを起動
docker-compose.ymlと.envのある場所でコマンドを実行すること。

```sh
sudo docker compose up -d
```

## ステップ5: 再起動後の自動起動設定

1. 既存のコンテナにrestart policyを設定

```sh
sudo docker update --restart always n8n-n8n-1
sudo docker update --restart always n8n-traefik-1
```

2. Dockerサービスをシステムの再起動時に自動的に起動するように設定

```sh
sudo systemctl enable docker
```

これで、EC2インスタンスの再起動時にDockerが自動的に起動し、n8nとTraefikのコンテナが再起動されます。

---

このガイドに従って設定を行えば、Dockerとn8nの環境が正常に動作し、システムの再起動後も自動的に起動します。
