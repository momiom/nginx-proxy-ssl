# dockerコンテナをSSL化
docker + docker-compose.yml + nginx-proxy + letsencrypt-nginx-proxy-companionで他のdockerコンテナをSSL化する。
- nginx-proxy ... 他のコンテナに数行設定を書くとそこにプロキシしてくれるコンテナ
- letsencrypt-nginx-proxy-companion ... nginx-proxy用に作られた、Let's Encryptの自動登録と自動更新をしてくれるコンテナ

## 構成
docker-composeで全部自動で作成されるので、`docker-comopse.yml`と`.env`を書くだけ。

```
.
├── docker-compose.yml
├── .env
├── certs（自動生成）
├── conf.d（自動生成）
├── html（自動生成）
└── vhost.d（自動生成）
```

## 0からの構築
1. dockerのインストール
2. docker-composeのインストール
3. docker-compose.ymlの作成
4. 実行と確認

### dockerのインストール
必要なライブラリをインストール。
```
$ sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```

dockerのリポジトリを追加。
```
$ sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

yumのパッケージインデックスを更新。（DockerのINSTALLやUPGRADEの前に一回実行することが推奨されている。）
```
$ sudo yum makecache fast
```

インストール可能バージョンを調べる。
```
$ yum list docker-ce.x86_64 --showduplicates | sort -r
```

インストールする。
```
バージョン指定
$ sudo yum install docker-ce-17.06.0.ce-1.el7.centos

最新バージョン
$ sudo yum install docker-ce
```

自動起動の設定、起動
```
$ sudo systemctl enable docker
$ sudo systemctl start docker
```

確認
```
$ sudo docker run hello-world
```

sudoを書かなくてもようにする
```
現在のユーザーをdockerグループに追加
$ sudo usermod -aG docker $USER

グループが作成されていない場合は作成
$ sudo groupadd docker
```

再ログインまたは再起動後に確認
```
sudo不要で起動すればOK
$ docker run hello-world
```

### docker-composeのインストール
[公式サイト](https://github.com/docker/compose/releases)でインストールしたいバージョンに記載されたコマンドを実行
```
ダウンロードしてインストール
$ curl -L https://github.com/docker/compose/releases/download/1.24.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose

バイナリに実行権限を付与
$ chmod +x /usr/local/bin/docker-compose
```

確認
```
$ docker-compose --version
```

### docker-compose.ymlを作成
ディレクトリを作成し、このリポジトリをクローンする。
今回はnginx-proxyで作成。
```
$ mkdir nginx-proxy
$ git clone https://github.com/K-Kazuki/nginx-proxy-ssl.git
```

`.env`を作成し環境別の変数を記述。
```.env
DEFAULT_HOST=k12i.space
NETWORK=nginx-ssl
```

### 実行と確認
今回作成するコンテナとSSL化するコンテナはすべて同じネットワークにいる必要がある。
今回は`nginx-ssl`という名前のネットワークを作成する。SSL化するコンテナはこのネットワークに接続する。
```
nginx-sslという名前のネットワークを作成
$ docker network create nginx-ssl
```

Python + Flask + UWSGIのコンテナの設定例
```yml
version: "2"
services:

  uwsgi:
    build: ./app
    volumes:
      - ./app/:/var/www/
      - ./logs:/var/logs/
    ports:
      - "3031:3031"
    environment:
      TZ: "Asia/Tokyo"
    networks:
      - flask_uwsgi_nginx

  nginx:
    build: ./nginx
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/logs:/var/log/nginx
    networks:
      - flask_uwsgi_nginx
      
      # nginx-proxyのネットワークに接続する
      - nginx-ssl
    ports:
      - "60000:80"
    environment:
      TZ: "Asia/Tokyo"

      # proxyのvhostの設定
      VIRTUAL_HOST: example.com
      VIRTUAL_PORT: 60000

      # Let's Encryptの設定
      LETSENCRYPT_HOST: example.com
      LETSENCRYPT_EMAIL: mailaddress@example.com

networks:
  flask_uwsgi_nginx:
    driver: bridge

  # 作成済のnginx-sslに接続する
  nginx-ssl:
    external: true

```
*TODO: 確認用の簡単なコンテナを作成*
