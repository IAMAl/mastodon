# Mastodonの設定
ConoHaのVPSを対象としたMastodonサーバーの立ち上げ方法を説明する。

ConoHaサイト：
https://www.conoha.jp/


作業前に最低１回手順内容を確認することをお勧めする。
DNS設定がうまくできずその後のドメイン認証で躓くかもしれないのでこの辺を注意すること。
また、全体で最悪4日必要なので余裕をもって作業を行うこと。


## 1. ドメイン契約

ドメイン名を契約する。
ConoHaで購入可能。他で購入したものも可能。
ネームサーバーをメモする。

＊他で購入したドメインを使用する場合、後でドメイン管理側サイトでネームサーバーを登録する。

＊契約したドメイン名の登録に１～３日要する。

＊後で行うMastodon(VPS)サーバーとこのドメイン名の紐づけやドメイン名の認証は登録後に行う。


## 2. VPSサーバー契約

VPS構成を選択する際、余裕を持った構成を選択すること。

＊性能が低いと起動後にBad Gatewayエラーになる。

- OS:ubuntu 20.04 or 22.04
- アプリケーション: Mastodon
- rootパスワード設定して契約する。
- IPアドレスをメモする。


## 3. メールサーバー契約（外部サーバー契約も可能）

サブドメインを設定して契約する。

＊Mastodonに使用するドメインとすると運用上誤解がない。

ドメインを追加してメールアドレスを（アカウント名とパスワードの設定）追加する。

SMTPサーバーをメモする。

接続先サーバー情報についてTXTRecordの名称と値をメモする。


## 4. DNS設定
ドメイン名を追加して、
- タイプ「A」を選択、名称「＠」、メモした値「IPアドレス」を追加。
- タイプ「MX」を選択、値としてSMTPサーバーを追加。
- タイプ「TXT」を選択、メモしたTXTRecordの名称とその値を追加。

＊反映までに10数分～数時間要する。

### DNS設定の確認方法

#dig 契約したドメイン名

DNS設定で追加したタイプ「A」が登録されているか確認する。
登録されていれば、ANSWERの値が1以上になり、ANSWERのセクションにIPアドレスが表示される。

ANSWERの値が0の場合
- ドメインがまだ登録されていない
- DNSがまだ反映されていない
- DNS設定に誤りがある

のいずれかが原因。


## 5. VPSサーバーへログインし設定

アカウント名「root」でVPS契約時のrootパスワードでログイン。


### nginx設定

ディレクトリ```/etc/nginx/sites-available/```に以下内容のファイルをファイル名「契約したドメイン名.conf」で作成し、「契約したドメイン名」の部分四か所を修正する。
```
#cd /etc/nginx/sites-available/
```
```
sitemap $http_upgrade $connection_upgrade {
  default upgrade;
  ''      close;
}

server {
  listen 80;
  listen [::]:80;
  server_name 契約したドメイン名;
  root /home/mastodon/live/public;
  # Useful for Let's Encrypt
  location /.well-known/acme-challenge/ { allow all; }
  location / { return 301 https://$host$request_uri; }
}

server {
  listen 443 ssl http2;
  listen [::]:443 ssl http2;
  server_name 契約したドメイン名;

  ssl_protocols TLSv1.2;
  ssl_ciphers HIGH:!MEDIUM:!LOW:!aNULL:!NULL:!SHA;
  ssl_prefer_server_ciphers on;
  ssl_session_cache shared:SSL:10m;

  ssl_certificate     /etc/letsencrypt/live/契約したドメイン名/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/契約したドメイン名/privkey.pem;s-available/Mastodon
map $http_upgrade $connection_upgrade {
  default upgrade;
  ''      close;
}

  root /home/mastodon/live/public;

  gzip on;
  gzip_disable "msie6";
  gzip_vary on;
  gzip_proxied any;
  gzip_comp_level 6;
  gzip_buffers 16 8k;
  gzip_http_version 1.1;
  gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

  add_header Strict-Transport-Security "max-age=31536000";

  location / {
    try_files $uri @proxy;
  }

  location ~ ^/(emoji|packs|system/accounts/avatars|system/media_attachments/files) {
    add_header Cache-Control "public, max-age=31536000, immutable";
    try_files $uri @proxy;
  }

  location /sw.js {
    add_header Cache-Control "public, max-age=0";
    try_files $uri @proxy;
  }

  location @proxy {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header Proxy "";
    proxy_pass_header Server;

    proxy_pass http://127.0.0.1:3000;
    proxy_buffering off;
    proxy_redirect off;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;

    tcp_nodelay on;
  }

  location /api/v1/streaming {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header Proxy "";

    proxy_pass http://127.0.0.1:4000;
    proxy_buffering off;
    proxy_redirect off;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;

    tcp_nodelay on;
  }

  error_page 500 501 502 503 504 /500.html;
}
```

ファイルを参照できるようにシンボリックリンクで置いておく。
```
#cd /etc/nginx/sites-enabled
ln -s ../sites-available/契約したドメイン名.conf
```

このファイルはセミコロンが抜けていたりでエラーになりやすいのであらかじめテストしてエラーを修正しておく。
```
#nginx -t
```

## 6. ドメイン名の認証手続き（チャレンジ）

チャレンジ前に認証プログラム（nginx）を停止させる。
```
#systemctl stop nginx
```

### 認証手続き

認証手続きを実行する。
```
#certbot certonly --standalone -d 契約したドメイン名
```
チャレンジが成功すれば認証が完了。

不成功の場合、その理由が記載されているメッセージを確認する。
タイプ「A」がない、と記載されている場合、

- ドメイン名の登録がまだされていない
- DNS設定がまだ反映されていない
- DNS設定に誤りがある

のいずれかが原因。

＊１時間以内に合計５回チャレンジ不成功が許されている。

＊それ以上一時間以内に試行すると１っ週間認証手続きができなくなるので注意すること。

＊それ以上のチャレンジを行う場合は数時間空けてから行うこと。


成功したらnginxを起動する。
```
#systemctl start nginx
```

認証時に参照できるようにドメイン上のルートディレクトリを指定する。
```
#certbot certonly --webroot -d 契約したドメイン名 -w /home/mastodon/live/public/
```


### TLS証明書の自動更新設定
```
#cd /etc/cron.daily
```
このディレクトリにファイル名「letsencrypt-renew」を以下の内容で作成する。
```
#!/usr/bin/env bash
certbot renew
systemctl reload nginx
```

定期的に証明を実行させる。
```
#chmod +x /etc/cron.daily/letsencrypt-renew
#systemctl restart cron
```


## 7. Mastodonの設定

アカウント「mastodon」としてログイン。
```
#sudo su – mastodon
#bash
#cd ~/live
```
Mastodonの設定を開始する。
```
#RAILS_ENV=production ~/.rbenv/versions/3.0.4/bin/bundle exec rake mastodon:setup
```
ここでPostgreSQL、Redis、SMTPの設定を行う。

＊テストメール送信で失敗しても後でDNS設定で修正できるのでメール送信に関しては問題なし。

＊Dockerを使ってインストールしないのでこれについて聞かれたら「N」。

＊クラウドへ画像や動画をアップロードする設定をする場合、追加で認証設定をここで行う。


ログアウト。
```
#exit
#exit
```


## 8. Mastodonデーモンの設定（ファイルが既にある場合は不要）
```
#cd /etc/systemd/system/
```
ファイル名「mastodon-web.service」で下記内容のテキストを作成する。
```
[Unit] Description=mastodon-web
After=network.target
[Service] Type=simple
User=mastodon
WorkingDirectory=/home/mastodon/live
Environment=”RAILS_ENV=production”
Environment=”PORT=3000″
ExecStart=/home/mastodon/.rbenv/versions/3.0.4/bin/bundle exec puma -C config/puma.rb
ExecReload=/bin/kill -SIGUSR1 $MAINPID
TimeoutSec=15
Restart=always
[Install] WantedBy=multi-user.target
```

同じ
ディレクトリで、
ファイル名「mastodon-sidekiq.service」で下記内容のテキストを作成する。
```
[Unit] Description=mastodon-sidekiq
After=network.target
[Service] Type=simple
User=mastodon
WorkingDirectory=/home/mastodon/live
Environment=”RAILS_ENV=production”
Environment=”DB_POOL=5″
ExecStart=/home/mastodon/.rbenv/versions/3.0.4/bin/bundle exec sidekiq -c 5 -q default -q push -q mailers -q pull
TimeoutSec=15
Restart=always
[Install] WantedBy=multi-user.target
```

同じディレクトリで、
ファイル名「mastodon-streaming.service」で下記内容のテキストを作成する。
```
[Unit] Description=mastodon-streaming
After=network.target
[Service] Type=simple
User=mastodon
WorkingDirectory=/home/mastodon/live
Environment=”NODE_ENV=production”
Environment=”PORT=4000″
ExecStart=/usr/bin/npm run start
TimeoutSec=15
Restart=always
[Install] WantedBy=multi-user.target
```


## 9. Mastodon（デーモン）の起動
```
#systemctl enable /etc/systemd/system/mastodon-*.service
#systemctl enable nginx
```
＊停止したい時は「enable」ではなく「stop」を指定して実行。


## 10. WebでMastodonの起動を確認

URLに契約したドメイン名を入力し起動を確認する。
```
"We're sorry, but something went wrong on our end."
```
とサイトに表示された場合、サーバーを再起動する。
