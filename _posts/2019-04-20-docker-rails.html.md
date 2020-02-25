---
:title: 用 Docker 部署 Rails，原來是這樣！？
:date: 2019-04-20 00:00:00 UTC
:slug: deploying-your-Docker-rails-app
:author: Johnson
:tags: 5xruby, 心得分享, 五倍紅寶石, Docker,
:category: techpost
:image:
  :original: post/image/2019-04-20-docker-rails/cover.jpg
  :preview: post/image/2019-04-20-docker-rails/cover.jpg
  :thumb: post/image/2019-04-20-docker-rails/cover.jpg
---
> 圖片作者：Julius Silver，
> [圖片來源連結](https://pixabay.com/zh/photos/%E6%B1%89%E5%A0%A1-%E6%B1%89%E5%A0%A1%E6%B8%AF-%E9%9B%86%E8%A3%85%E7%AE%B1%E8%88%B9-%E5%BE%B7%E5%9B%BD-3021820/)

如果你曾經看過 Docker 相關的文章，對 Docker 有點認識但又不是太熟，需要找個主題來實作看看，那這篇文章就是為你準備的！我們將透過 Docker 部署一個 Rails 應用到 DigitalOcean 平台上，本文會一步一步介紹部署的過程。

---
### 目錄

* Docker 基本介紹
  * Docker 三大元素
  * 設定檔
  * 常用指令
* 實戰演練
  * 步驟0: 簡單開一個 Rails 專案
  * 步驟1: 開始加入 Docker
  * 步驟2: Nginx 搭配 Puma
  * 步驟3: 部署到雲端上吧！
  * 步驟4: HTTPS 認證
  * 步驟5: Docker image 瘦身
* 結論

---

## Docker 基本介紹

在程式開發的過程中，常常會發生環境不同或是系統設定有誤的情況，這樣的情況如果在本機重現不了 bug 真的很讓人頭痛，如果有一種技術可以讓我們本機的環境跟測試站、正式站一樣或是只要更接近一點，一定可以省下不少時間。於是 Docker 出現了，Docker 是現在當紅的虛擬化技術，它非常的輕量，啟動、停止是可以在幾秒內完成的那種，同時 Docker 對系統資源需求很少，甚至一台主機可以同時啟動數百甚至數千個 Docker 容器。為了要學習和體驗一下 Docker 的威力，所以我試著用 Docker 來部署我的 Rails 應用程式，這是個很好的機會讓我對不管是 Docker 技術還是環境部署都有更深一層的認識！

> 礙於篇幅這邊只會簡要說明實作部分可能用到的名詞，更詳細的資料可能需要再 google 一下了


### Docker 三大元素

- Docker Image

```
Docker 映像檔，有點像虛擬機的映像檔，是一個能產生 Docker Container 的模板
之後文章中都簡寫成 Image
```

- Docker Container

```
Docker 容器，就是用它來運行和隔離環境，它有點像簡易版的 Linux 環境
我們可以用 Docker Image 來建立 Docker Container，也能將它停止刪除等等
之後文章中都簡寫成 Container
```


- Docker Registry

```
Docker 倉庫註冊伺服器，可以把我們的 Image 放上去
也能在其他機器把這些 Docker Image 下載下來
Docker Hub 就是 Docker 倉庫註冊伺服器的代表
```

### 設定檔

- Dockerfile

```
可以設定自己客製化的 Docker Image
```

- docker-compose.yml

```
docker-compose.yml 是個可以同時管理，啟動多個 Container 的工具
```


### 常用指令

- docker images

```
列出目前機器上有的 Image
```

- docker ps

```
列出目前機器上有運作中的 Container
```


- docker exec -it `<Container ID>` bash

```
進入指定的 Container
-i 參數是本機電腦的鍵盤接到 Container 的 stdin
-t 參數是 Container 的 stdout 接到本機電腦的螢幕
```


- docker run -it `<Image ID>` bash

```
用 Image 創建一個 Container 然後運行
```


- docker rmi `<Image ID>`

```
刪除指定的 Image
```

- docker rm `<Container ID>`

```
刪除指定的 Container
```

- docker build .

```
依據 Dockerfile 的設定建立 Image
```

- docker push `NAME[:TAG]`

```
上傳 Image 到 Docker Registry
```

- docker pull `NAME[:TAG]`

```
從 Docker Registry 下載 Image 到本機
```

- docker-compose up

```
啟動 docker-compose.yml 的設定，有 Container 就會啟動
沒 Container 也會從指定的 Image 生成
```

---


## 實戰演練


看完 Docker 基本介紹可能有種似懂非懂的感覺，所以我們就來操作一下吧！這單元除了會用到上述 Docker 相關的知識外，也會需要一些部署的觀念，可能可以參考[這篇文章](https://5xruby.tw/posts/rails-deploy/)


### 步驟 0: 簡單開一個 Rails 專案

從終端機輸入指令建立專案吧！

```
rails new docker_study
```

接著新增 `gem 'pg'` 以及更新對應的 `config/database.yml`

完成後再用 scaffold 產生基本框架

```
rails generate scaffold post title:string body:text
```

如果想要畫面好看一點點可以加上 bootstrap
請參考： [https://github.com/twbs/bootstrap-sass](https://github.com/twbs/bootstrap-sass)

就先這樣，完成一個最簡單只有 CRUD 網站

### 步驟 1: 開始加入 Docker

> 目標：
> 在本機使用 docker-compose up 啟動多個 Container
> 瀏覽器輸入 http://127.0.0.1:3000 可以開啟專案

我們先試著在本機的環境使用 Dokcer，新增 Dockerfile 如下

```=
FROM ruby:2.5.1
MAINTAINER johnson <johnson@5xruby.tw>
RUN apt-get update && apt-get install -y build-essential libpq-dev nodejs vim postgis imagemagick
RUN mkdir /app
WORKDIR /app
COPY Gemfile /app/Gemfile
COPY Gemfile.lock /app/Gemfile.lock
ENV RAILS_ENV production
RUN bundle install
COPY . /app
CMD rake db:migrate assets:precompile && puma -C config/puma.rb
```

這一個 Dockerfile 產生的 Image 就是我們 Rails App 的 Image，它是從一個有 Ruby 的 Linux 環境做出來的 `FROM ruby:2.5.1`，同時它安裝了許多環境中需要的套件 `RUN apt-get ...`，最後用 puma 啟動它。


看到這邊可能會有個疑問，為什麼在第 6、7、10 行都要 COPY，而不是直接在第六行寫一個 `COPY . /app` 就好了？因為這樣子的話我們的 Rails 專案每次有修改，都會重新跑一次 `bundle install` 。這是什麼意思！？ Dockerfile 在建立 Image 的時候是一層一層的包上去的，也就是 Dockerfile 中每一行設定都是會記錄下來的，如果這次專案沒有修改任何東西，當我們執行 `docker build .` 後，就不會重新產生 Image ，如果有做修改，就是從有修改的那個步驟再往下走，譬如第十行才修改，那就只會重新跑第十行後的設定。


目前操作的時候用 ruby 2.4.1 版會遇到問題，這篇文章先用 Ruby 2.5.1 版來操作，所以在Gemfile 要記得設定
```
ruby '2.5.1'
```

>如果用 ruby 2.4.1 會遇到什麼問題呢？在用 Docker 跑 ruby 2.4.1 的時候用 apt-get update 指令會遇到下面這問題
>
>```
>W: Failed to fetch http://deb.debian.org/debian/dists/jessie-updates/InRelease  Unable to find expected entry 'main/binary-amd64/Packages' in Release file (Wrong sources.list entry or malformed file)
>```
>會有這問題是因為 `/etc/apt/sources.list` 檔案，也就是 APT 的套件來源清單沒更新，所以更新一下就能繼續用 ruby 2.4.1 囉，但在這範例我就直接用 ruby 2.5.1 給它操作下去！


docker-compose.yml


```=
version: '3'

services:
  db:
    image: postgres:9.6
    environment:
      - "POSTGRES_USER=postgres"
      - "POSTGRES_PASSWORD=postgres"
    networks:
      - johnson
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgres://postgres:postgres@db/postgres
      - SECRET_KEY_BASE=8527f4cd4df56c3f41d8dc27251c09a55ed872eb299d230da758a05ee399b20ed655c33cf3039eccd59cf787cfb05ebe2b3e5a0b1811ad9d14163d0ee48f5f39
      - RAILS_SERVE_STATIC_FILES=true
    networks:
      - johnson
    depends_on:
      - db

networks:
  johnson:

```
現在我們同時啟動兩個 Container ，一個是我們的 Rails App，另一個則是 Postgres 資料庫，其中資料庫的部分我們是直接使用現成的 Image 來建立，也就是`image: postgres:9.6`，而 Rails App 則是從當前目錄 `(備註1)` 的 Dockerfile 產生的 Image 來產生也就是`build: .`。

> 備註1
>  放 docker-compose.yml 的目錄

在 Rails App 的 Container 啟動的最後階段會執行 `rake db:migrate` (Dockerfile CMD 設定的）也就是說必須先讓資料庫的 Container 啟動後，才能啟動 Rails App 的 Container ，因此可以在 docker-compose.yml 中看到以下的設定。

```
app:
  depends_on:
     - db
```

在專案中新增完這兩個檔案後，就執行吧
```
docker-compose up
```
沒意外的話到瀏覽器輸入 http://127.0.0.1:3000 應該能看到專案跑起來了！希望你可以看到類似這樣的畫面XD

![](https://i.imgur.com/kzzMYjS.png)


同時也可以用 `docker ps` 來看看這些跑起來的 Container

```
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
09f91d4b32aa        docker_app_app      "/bin/sh -c 'rake db…"   41 seconds ago      Up 11 seconds       0.0.0.0:3000->3000/tcp   docker_app_app_1
6297180618b4        postgres:9.6        "docker-entrypoint.s…"   42 seconds ago      Up 12 seconds       5432/tcp                 docker_app_db_1
```


### 步驟 2: Nginx 搭配 Puma

> 目標:
> 在 docker-compose.yml 中加入 Nginx
> 客製化 Nginx 的相關設定，讓 Nginx 能搭配 Puma 處理流量

誒！？什麼是 Nginx 跟 Puma？為什麼需要這些東西？剛剛不是已經把專案跑起來了嗎？

其實一個完整的網站服務，除了有我們的 Rails App 之外，還會有專門處理 http request 的 reverse proxy server 像是 Nginx 或是 Apache ，還有在 reverse proxy server 跟 Rails App 中間的 App Server，例如 Puma 或 Passenger。

> 如果想了解更多 Nginx 或是部署相關的說明，可以參考[這篇文章](https://5xruby.tw/posts/rails-deploy/)


由於 Puma 是 Rails 內建的 App Server，所以他會和我們的 Rails App 一起啟動，Puma 的設定也在 `config/puma.rb` 調整即可，而 Nginx 的部分則是我們自己透過 Dockerfile 來客製化的。

為了要讓 Puma 可以搭配 Nginx，我們需要讓 Puma bind 在 Unix domain socket 上，而非一般常見的  TCP/IP network socket，修改 `config/puma.rb` 如下

```
# port        ENV.fetch("PORT") { 3000 }
bind "unix:///app/tmp/sockets/puma.sock"
```


再來我們要用 Docker 的 Nginx 來作為我們的 reverse proxy server，在專案目錄中再開一個 Dockerfile，路徑可以是 `docker/nginx/Dockerfile`，這個 Dockerfile 就是官方的 Nginx 加上我們修改過的 `default.conf` 檔案，這檔案就是 Nginx 設定檔，我們要在裡面設定 Nginx 處理 request 的規則。新增 `docker/nginx/Dockerfile` 如下

```
FROM nginx
COPY default.conf /etc/nginx/conf.d/default.conf
```

我們要用下面這個設定方式，來執行我們的 Nginx，修改 `docker/nginx/default.conf` 如下

```=
server {
    listen       80;
    server_name  localhost;

     location / {
        proxy_pass http://unix:/app/tmp/sockets/puma.sock;
        proxy_set_header X-Forwarded-Host localhost;
    }
}
```
上面的設定中，比較需要注意的是第六行的部分，要讓 Nginx 把接收到的 request 傳給 Puma

修改 `docker-compose.yml` 如下

```
app:
  volumes:
      - /tmp/sockets:/app/tmp/sockets
nginx:
  build: docker/nginx/.
  ports:
    - "80:80"
  networks:
    - johnson
  volumes:
    - /tmp/sockets:/app/tmp/sockets
  depends_on:
    - app
```
在 `docker-compose.yml` 也加上 Nginx，由於我們要讓 Nginx 跟 Web 這兩個 Container 可以共用 `/app/tmp/sockets/puma.sock` 這個檔案，所以我們用 volumes 來設定，讓 App 跟 Nginx 的 Container 的 `/app/tmp/sockets` 都會對應到本機的 `/tmp/sockets`。由於這次我們更新到 `config/puma.rb` 所以我們需要執行一次 `docker-compose build` 來讓 Dockerfile 重新檢查並產生新版本的 Image，更新完後再跑一次 `docker-compose up`


這一次 Nginx 可以搭配 Puma 一起運作！可以直接在瀏覽器輸入 `127.0.0.1` 來看我們的網站，由於 Nginx 是設定 port 80 ，因此這次不用在網址後面加上 `:3000`，然後再試著用 `docker ps` 看看我們啟動的 Container 吧！

```
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                NAMES
c54bb26356b1        docker_app_nginx    "nginx -g 'daemon of…"   12 minutes ago      Up 13 seconds       0.0.0.0:80->80/tcp   docker_app_nginx_1
a65a5c532bb2        docker_app_app      "/bin/sh -c 'rake db…"   12 minutes ago      Up 13 seconds                            docker_app_app_1
6297180618b4        postgres:9.6        "docker-entrypoint.s…"   About an hour ago   Up 14 seconds       5432/tcp             docker_app_db_1
```


### 步驟 3: 部署到雲端上吧！
> 目標:
> 將我們的 Docker 應用部署到 DigitalOcean 平台上
> 並用申請好的 Domain Name 連線成功

首先到 DigitalOcean 上申請一台機器吧！DigitalOcean 有提供已經裝好 Docker 的機器選擇，其實這就是一台裝好 Docker 的 Ubuntu 機器，我自己是選每月 10 美元的方案!

![](https://i.imgur.com/6h29f72.png)

![](https://i.imgur.com/SoUVvtq.png)

DigitalOcean 的機器申請完後，可以在 `Droplets` 列表找到一組代表這台機器的 IP，有 IP 後就要來申請個 Domain Name，這樣才有部署網站的樣子嘛！可以網路上找個免費的服務來申請，像是 [freenom](https://my.freenom.com) 就是個不錯的選擇。這練習中我註冊的 Domain Name 是 `dockerapp.ga`

所以要記得把 `docker/nginx/default.conf` 修改如下

```
server {
    listen       80;
    server_name  dockerapp.ga;

    location / {
        proxy_pass http://unix:/app/tmp/sockets/puma.sock;
        proxy_set_header X-Forwarded-Host localhost;
    }
}
```

從這步驟起，我們要把所有的 Image 都放到 [Docker Hub](https://hub.docker.com/) 上面，這樣就可以直接從 Docker Hub 把 Image pull 到機器上跑，這樣讓我們的環境更乾淨了一些。

首先把 `docker-compose.yml` 這兩個地方修改一下，這樣我們的 `docker-compose up` 用的 Image 就會是來自 Docker Hub ，而非當下目錄 build 出來的

```
  app:
    image: johnsonzhan121/docker_app:latest

  nginx:
    image: johnsonzhan121/docker_nginx:latest
```

> johnsonzhan121 是我在 Docker Hub 上的帳號，我的 Image 都會放在這個帳號之下

修改完後，可以執行 `docker-compose up` 看看會發生什麼事？

```
Pulling app (johnsonzhan121/docker_app:latest)...
ERROR: manifest for johnsonzhan121/docker_app:latest not found
```
在 Docker Hub 並沒有我的 Image，我現在需要把 Image 一個一個在本地 build 完，然後 psuh 到 Docker Hub 上面，也就是執行以下指令

```
docker build . -t johnsonzhan121/docker_app
docker build docker/nginx/. -t johnsonzhan121/docker_nginx
docker push johnsonzhan121/docker_app
docker push johnsonzhan121/docker_nginx
```

事情還沒完成，我們要在 DigitalOcean 平台上執行 `docker-compose up`，還需要把 Docker Hub 上的 Image pull 到 DigitalOcean 上等等，要做的事太多，直接寫成腳本自動執行吧！以後只要執行 `ruby auto_deploy.rb` 就能把部署的事情做完啦！自己新增一個 `auto_deploy.rb` 檔案吧


```=
#!/usr/bin/env ruby

p 'build docker_app and docker_nginx'
`docker build . -t johnsonzhan121/docker_app`
`docker build docker/nginx/. -t johnsonzhan121/docker_nginx`

p 'push docker_app and docker_nginx'
`docker push johnsonzhan121/docker_app`
`docker push johnsonzhan121/docker_nginx`

p 'update docker-compose.yml'
`scp docker-compose.yml root@178.128.214.6:/home/docker`

p 'pull Image from Docker Hub'
`ssh root@178.128.214.6  'docker pull johnsonzhan121/docker_app'`
`ssh root@178.128.214.6  'docker pull johnsonzhan121/docker_nginx'`

p 'remote docker-compose up'
`ssh root@178.128.214.6  'cd /home/docker && docker-compose up'`
```
> 178.128.214.6 是我 DigitalOcean 的 IP，你們的腳本要換上自己的 IP 喔，當然也可以用你申請到的 Domain Name


在執行 `ruby auto_deploy.rb` 之前記得要到伺服器上新增對應的目錄喔，像是這個 `/home/docker` 正常執行後終端機會長這樣

```
"build docker_app and docker_nginx"
"push docker_app and docker_nginx"
"update docker-compose.yml"
"pull Image from Docker Hub"
"remote docker-compose up"
Starting docker_db_1 ... done
Recreating docker_app_1 ... done
Recreating docker_nginx_1 ... done
```
然後就能用剛剛申請的 Domain Name 來試試囉！也可以用 ssh 登入 DigitalOcean 然後輸入指令 `docker images` 或 `docker ps` 查看，我的網站已經可以用 `http://dockerapp.ga/` 連線上囉！雖然目前只能做 get 的操作還不能用 post 的操作。

### 步驟 4: HTTPS 認證

> 目標:
> 透過 certbot 來申請 SSL 憑證
> 可以用在瀏覽器輸入 https://dockerapp.ga 然後連線成功

一般正式的網站都會採用 HTTPS，來讓網站更安全更專業，我們要透過 Cerbot 自動化工具來申請免費的 Let’s Encrypt 的 SSL 憑證

在 `docker-compose.yml` 加入 certbot 以及調整 Nginx 如下

```
  nginx:
    image: johnsonzhan121/docker_nginx:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /tmp/sockets:/app/tmp/sockets
      - ./data/certbot/conf:/etc/letsencrypt
      - ./data/certbot/www:/var/www/certbot

  certbot:
    image: certbot/certbot
    volumes:
      - ./data/certbot/conf:/etc/letsencrypt
      - ./data/certbot/www:/var/www/certbot
```
我們要讓 Nginx 跟 certbot 可以共用 `/etc/letsencrypt` 、`/var/www/certbot` 這兩個目錄，前者是放我們 SSL 認證的資料，後者是拿來申請 SSL 憑證用的。再來就是修改 Nginx 設定檔的部分，我們要用兩個 Sever 的 block 來控制不同 port 進來的 request，第一個 server block 是接收 80 port 的 request，接收到後把 request 轉到第二個 server block 也就是 443 port 的。這邊可以注意到我們新增了 `proxy_set_header X-Forwarded-Ssl on;`，有了它後我們就能對網站做 post request 囉，不然原本會遇到 Rails 的 `ActionController::InvalidAuthenticityToken ` 問題

```=
server {
    listen       80;
    server_name  dockerapp.ga;
    return 301 https://dockerapp.ga$request_uri;
}

server {
    listen       443 ssl;
    server_name dockerapp.ga;

    ssl on;
    # 剛剛說的 SSL 認證資料
    ssl_certificate      /etc/letsencrypt/live/dockerapp.ga/fullchain.pem;
    ssl_certificate_key  /etc/letsencrypt/live/dockerapp.ga/privkey.pem;

    # 跟 Puma 搭配用
    location / {
        proxy_pass http://unix:/app/tmp/sockets/puma.sock;
        proxy_set_header X-Forwarded-Ssl on;
        proxy_set_header X-Forwarded-Host dockerapp.ga;
    }

    # 申請 SSL 憑證用的
    location ~ /.well-known {
        root /var/www/certbot;
    }
}
```

完成囉！執行看看 `ruby auto_deploy.rb` 執行完後好像開始不能連上網站了XD，我們需要登入到機器裡面看看，登入機器後用 `docker ps`

```
CONTAINER ID        IMAGE                              COMMAND                  CREATED              STATUS              PORTS               NAMES
8dac143fc834        johnsonzhan121/docker_app:latest   "/bin/sh -c 'rake db…"   About a minute ago   Up About a minute                       docker_app_1
2887c373e865        postgres:9.6                       "docker-entrypoint.s…"   18 minutes ago       Up 13 minutes       5432/tcp            docker_db_1
```

看來 Nginx 沒有跑起來，所以到 `/home/docker` 跑跑看 `docker-compose up` ，跑完後大概看到這樣的東西

```
docker_nginx_1 exited with code 1
certbot_1  | Saving debug log to /var/log/letsencrypt/letsencrypt.log
certbot_1  | Certbot doesn't know how to automatically configure the web server on this system. However, it can still get a certificate for you. Please run "certbot certonly" to do so. You'll need to manually configure your web server to use the resulting certificate.
docker_certbot_1 exited with code 1
```
我們遇到一個問題，我們要用 Nginx 來申請 SSL 憑證，但是 Nginx 沒有申請 SSL 憑證的情況下無法啟動 Container，要處理這問題很麻煩，我們需要先做出一個假的憑證讓 Nginx 可以正常啟動，然後刪除掉這假憑證，接著正式申請 Let’s Encrypt 的 SSL 憑證。看到這真是麻煩到讓人想放棄，好險有人已經針對這塊寫好了腳本讓大家一鍵完成以上工作，請參考[這篇文章的教學](https://medium.com/@pentacent/nginx-and-lets-encrypt-with-docker-in-less-than-5-minutes-b4b8a60d3a71) 看完教學後，我們需要做的事情就是下載腳本，修改權限

```
curl -L https://raw.githubusercontent.com/wmnnd/nginx-certbot/master/init-letsencrypt.sh > init-letsencrypt.sh

chmod +x init-letsencrypt.sh
```
然後修改檔案裡面的 domains 等參數，然後再執行 `sudo ./init-letsencrypt.sh` 即可完成，試試看用 HTTPS 來拜訪網站吧！


### 步驟 5: Docker image 瘦身

> 目標:
> 用 alpine 的 Ruby Image 來幫我們的 web image 瘦身
> 從原本的 1.72GB 簡化到 503MB

Docker Image 的大小是衡量我們容器技術的重要指標，小的 Docker Image 可以讓部署更有效率而且也更安全！用 `docker iamges` 來看看我們的 web Image 到底多大吧，居然有 1.72GB，我們有很大的空間來幫它瘦身。

```
REPOSITORY                    TAG                 IMAGE ID            CREATED             SIZE
johnsonzhan121/docker_app     latest              fab76a5630cb        45 minutes ago      1.72GB
ruby                          2.5.1               3c8181e703d2        5 months ago        869MB
```

要幫 Image 瘦身有很多方法，包含使用小的 Base Image 、用 Multi-Stage 的方式寫 Dockerfile、移除不用進入 Image 的檔案等等，在這次實作我們將要用縮小 Base Image 來瘦身，因為在 1.72 GB 的 Image 中  Base Image 就佔了 896MB 呀！


其實 Ruby 有更小 Image 可以使用，那就是 Alpine 版本的 Ruby Image 啦！什麼是 Alpine？ Alpine 是一個輕型的 Linux 發行版本，目前大部分 Docker 官方映像檔都已經有支援 Alpine 作為基礎的映像檔，當然 Ruby 也不例外，下面是 Alpine 版本的 Ruby 比原版小的多，只有 45.4MB

```
REPOSITORY                    TAG                 IMAGE ID            CREATED             SIZE
ruby                          2.5.1-alpine        d29267791323        6 months ago        45.3MB
```
來改寫我們的 Dockerfile 吧！

把原本的

```=
FROM ruby:2.5.1
RUN apt-get update && apt-get install -y build-essential libpq-dev nodejs vim postgis imagemagick
```

改成

```=
FROM ruby:2.5.1-alpine
RUN apk update && apk upgrade && apk add --update --no-cache build-base nodejs imagemagick postgresql-dev
```

第一行好理解，就是改用 Alpine 版本的 Ruby。第二行不是用 Ubuntu 常見的 `apt-get`，而是 `apk add`， apk 就是 Alpine 的套件管理工具，那我要怎麼知道每個原本用 `apt-get` 套件在 apk 有沒有支援呢？只能試試看了？或是到 apk 提供的[文件](https://pkgs.alpinelinux.org/packages)去看囉。完成後再部署一次吧，執行 `ruby auto_deploy.rb`

執行後會看到類似這樣的錯誤訊息

```
web_1      | rake aborted!
web_1      | TZInfo::DataSourceNotFound: tzinfo-data is not present. Please add gem 'tzinfo-data' to your Gemfile and run bundle install
```

看起來是少裝了 `tzinfo-data`，這個 gem 到底是什麼呢？稍微看一下我們這個預設的 Gemfile 裡面其實有寫到

```
# Windows does not include zoneinfo files, so bundle the tzinfo-data gem
gem 'tzinfo-data', platforms: [:mingw, :mswin, :x64_mingw, :jruby]
```
看上面的意思就是說，如果在某些環境執行 Rails 就會需要裝這個 gem，剛好我們從原來的 ruby:2.5.1 `Debian` 系統，換成現在 ruby:2.5.1-alpine 就遇到了這問題，沒意外的話，只要加上這個 gem 就真的能順利運作了吧！？於是我們加上這個 gem 重跑一次試試看！沒錯成功啦 ><

再一次用 `docker iamges` 指令來確認，我們成功瘦身了多少？
我們從原本的 1.72GB 瘦身成 503MB 啦！還有一些進步空間，那就之後再說吧～

```
REPOSITORY                    TAG                 IMAGE ID            CREATED             SIZE
johnsonzhan121/docker_app     latest              6a7f0235885f        About a minute ago   503MB
```



## 結論

終於成功用 Docker 部署一個 Rails 應用到 DigitalOcean 平台上了，雖然很陽春但還是很開心可以完成。[範例程式碼](https://github.com/shes50103/docker_app)都放到 GitHub 上了，需要就拿去吧，每個步驟都有用 git commit 分出來。目前手邊的專案還沒有使用到 Docker 來部署，但在未來還是會想再撥一些時間研究 Docker，目前為止感覺它是相對有趣的技術，希望未來可以對 Docker 越來越熟悉，甚至在工作中使用到。
