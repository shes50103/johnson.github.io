---
:title: Rails 部署工具，原來是這樣
:date: 2018-08-17 09:34 UTC
:slug: rails-deploy
:author: Johnson
:tags: Rails, Ruby on Rails, 5xruby, 心得分享, 五倍紅寶石
:category: techpost
:image:
  :original: post/image/157/photo-1526639673778-1ed7ff390a4a.jpeg
  :preview: post/image/157/preview_photo-1526639673778-1ed7ff390a4a.jpeg
  :thumb: post/image/157/thumb_photo-1526639673778-1ed7ff390a4a.jpeg

---

## 部署 Rails 你可能會需要用到...

在學習 Rails 部署的路上，你一定聽過 Nginx、 Passenger、 Capistrano 這幾個東西吧？這篇文章會依序介紹這幾個工具特色，以及常見功能！

> 學習 Rails 部署建議可以參考蒼時的 [系列教學文章](https://blog.frost.tw/tags/DevOps/)
它會詳細帶著讀者一步一步在 DigitalOcean 的機器上部署 Rails 專案

部署大概像什麼樣子？大概像下面這張圖吧
![](https://i.imgur.com/TeWMZTB.png)

遠端伺服器可能是 GCP、AWS、DigitalOcean 這些機器，程式碼倉庫則是 GitHub、 Gitlab、Bitbucket 這些服務，部署過程就是讓遠端伺服器可以取得程式碼倉庫的程式碼，然後順利的把專案執行起來。其中我們會需要在遠端伺服器安裝 Passenger、Nginx 這兩個軟體，同時得用 Capistrano 讓部署自動化、更有效率！

---
### 目錄

* Nginx & Passenger
  * 為什麼是 Nginx 搭配 Passenger ?
  * Passenger 擅長什麼呢？
  * Passenger 常用指令：
  * Nginx 設定檔說明
  * 自己設定 Nginx

* Capistrano
  * Capistrano 功能介紹
  * 部署架構
  * 設定檔
  * 常用指令
* 結論

---

## Nginx & Passenger

Nginx 是一個 Web Server，它的功能是接受 http request 並回覆 http response ， Nginx 本身和 Ruby 沒有關係，通常使用 Nginx 時我們還會搭配 Passenger 這一個 App Server，它可以管理 Rails 的 process 數量，同時 App Server 符合 Rack 的規範所以可以跟符合 Rack 規範的 Web Application (例如 Rails 溝通)

> 在接下來的文章中，會使用三個名詞

> Web Server 就是 Nginx

> App Server 就是 Passenger

> Web Application 就是 Rails App

> 關於 Passenger 和 Nginx 的安裝可以參考 [這篇文章](https://blog.frost.tw/posts/2018/04/10/Getting-started-deploy-your-Ruby-on-Rails-Part-4/#more)


### 為什麼是 Nginx 搭配 Passenger ?

這是我一開始學習部署時一直弄不懂的地方，我原本的想像是伺服器上面除了有我的 Web Application 外，再來就是某個軟體來處理其他的事情。到底為什麼需要 Nginx 和 Passenger 兩樣工具的搭配才能完成這些工作呢？

Nginx 和 Apache 都是 Web Server，它們專攻處理靜態檔案還有 http request。一般來說 Web Server 會放在 App Server 還有 Web Application 之前，它可以直接接觸 Client 端送來的請求，但在它後面的 App Server 和 Web Application 卻不行，因為這樣的架構讓這些軟體可以更有效率的分工，也讓我們的伺服器更安全。

此外，Web Server 還能處理效能上的議題，譬如 slow client 問題，如果 Client 端處理 http request 的速度超慢，Server 這邊就可能得花許多時間在等待這個 Client ，這樣的等待是很浪費資源的，為了解決這類型效能問題，又不讓其他開發者浪費時間重複造輪子，最好的方式就是讓 Web Server 處理這些問題，讓 App Server 和 Web Application 處理他們擅長的，這就是為什麼我們會用到 Nginx 和 Passenger 這兩樣工具了！


### Passenger 擅長什麼呢？

Passenger 會根據流量來判斷現在應該開啟多少個 Web Application，同時它也像是 Web Application 還有 Web Server之間的橋樑，它可以用 Rack 跟 Web Application 溝通，也能處理 Web Server 傳來的 http request，另外 Passenger 在架構上相當彈性也穩定，下圖是 Passenger 的架構圖

![](https://i.imgur.com/XevEwx1.png)

圖片來源：www.phusionpassenger.com

Passenger 是由多個模組所組成的，包含在編譯 Nginx 時一起編譯的 Phusion Passenger module，以及負責管理、重啟其它模組的 Watchdog 等等，因為這樣的結構可以讓 Passenger 更加有彈性，只要 Watchdog 沒有壞掉它就能幫忙重啟其他模組。

由於這些模組都是由許多的 process 在運行，因此彼此之間的溝通，需要透過一個 instance directory 的目錄，來存放這些 process 之間彼此溝通需要的檔案，所以在 nginx.conf 裡面我們會設定 `passenger_instance_registry_dir` 告訴 Passenger 它的 instance directory 在哪裡，我們在使用 `passenger-status` 指令的時候也是透過這資料夾內的資料取得資訊的。

> 之前在用 Passenger 的時候常常會跑不起來卻不知道為什麼？
> 原來就是少了 `instance directory` 這個目錄！

關於這架構圖更詳細介紹可看[官方文件](https://www.phusionpassenger.com/documentation/Design%20and%20Architecture.html#_web_server_module)


### Passenger 常用指令：

要使用 passenger 的指令必須先將 passenger 的 bin 目錄放到路徑之下，我們可以看到 passenger 的 bin 目錄下有這些執行檔
```
passenger  passenger-config  passenger-install-apache2-module  passenger-install-nginx-module  passenger-memory-stats  passenger-status
```

> 這些執行檔也就是我們可以使用的指令唷！

其中我們常用的可能有這兩個

```
# 查詢 passenger 的狀態
passenger-status

# 透過 passenger 重新啟動 Rails
passenger-config restart-app
```

咦！那要在哪邊做 Passenger 的相關設定呢？還記得前面說到的跟 Nginx 一起編譯的  Phusion Passenger module 模組嗎？因為這樣的關係，我們在寫 Nginx 設定檔時也可以一起設定一些 Passenger 需要的東西，接下來可以來看看 Nginx 的設定檔

### Nginx 設定檔說明

Nginx 安裝好後會有 `nginx.conf` 這設定檔，我們可以直接在裡面做修改，或是用 `include site-enabled/*.conf;` 之類的方法把設定檔寫在其他地方，在這個範例我們會將設定檔寫在其他地方，所以在 `nginx.conf` 加上一行 `include site-enabled/*.conf;` 即可，同時也得在有 `nginx.conf` 檔案的目錄下新增 site-enabled 資料夾來放我們要的設定檔。

如果把 nginx.conf 的註解都拿掉的話，你的設定檔可能長這樣

```
# 註解0
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    passenger_root /usr/local/passenger/passenger-5.3.3;
    passenger_ruby /usr/local/ruby-2.4.3/bin/ruby;
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;

    server {
        listen       80;
        server_name  localhost;
        location / {
            root   html;
            index  index.html index.htm;
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }

    # 這行是後來加上去的
    include site-enabled/*.conf;
}

```

在 nginx.conf 中可以看到 `http { ... }` 包起來的設定，在這裡面還會看到 ` server {...} ` 的結構，而我們的`include site-enabled/*.conf;` 會寫在 http 這個 block 內然後 server 這個 block 外，因為其實我們要寫的設定檔就是好幾個 `server {...}` ，這些層級之間的規定可以在 [Nginx 的文件](http://nginx.org/en/docs/beginners_guide.html
)中找到~

> 註解0

> 設定 Nginx 的 worker_processes 數量，Nginx 啟動方式是一個 master process 和多個 worker processes，master process 負責根據設定檔的設定來操作 worker processes，所以 worker processes 是真正在處理 http request 的 process！

### 自己設定 Nginx

在看完 nginx.config 後我們使用以下指令來建立我們自己的設定檔

```
$ mkdir site-enabled
$ cd site-enabled
$ vim xxx.conf (xxx可換為你的設定檔名稱，通常會用網域名稱好區分)
```

完成後可以在 `xxx.conf` 中這樣寫，其中 `#` 為註解符號

```
#設定 Passenger 的 instance directory
passenger_instance_registry_dir /var/run/passenger-instreg;

#第一個 server 區塊
server {

    #監聽 port 80 （ http request ）
    listen 80;

    #註解1，設定 server name
    server_name dodeploy.tk;

    #註解2，將 http request 都轉給第二個 server 區塊
    return 301 https://dodeploy.tk$request_uri;
}

#第二個 server 區塊
server {

    #監聽 port 443 （ https request ）
    listen       443 ssl;

    server_name dodeploy.tk;

    #SSL 憑證相關設定
    ssl on;
    ssl_certificate      /etc/letsencrypt/live/dodeploy.tk/fullchain.pem;
    ssl_certificate_key  /etc/letsencrypt/live/dodeploy.tk/privkey.pem;

    #註解3
    location ~ /.well-known {
        allow all;
        root /opt/nginx/html;
    }

    #提供靜態檔案的放置目錄
    root /home/deploy/do_deploy.com/current/public;

    #啟動 Passenger
    passenger_enabled on;
}
```

> 註解1

> 如果有需要，我們也可以讓多個 domain 都對應到這台機器，首先申請多個 domain name 都對應到我們機器的 IP，然後我們就可以用多個 server 的區塊分別接收到帶有不同網域名稱的 http 請求。


> 註解2

> 我們的範例會將 HTTP 轉換成 HTTPS ，要完成這功能要設定多個 server 的區塊，第一個區塊監聽 port 80，也就是只接受 HTTP 請求，在接收到 HTTP 請求後再來就是 return 將請求交由監聽 https 的第二個區塊來處理。

> 註解3

> 在第二個區塊我們做了 SSL 的憑證的設定，另外 location 那一塊是為了讓遇到 ~/.well-known 開頭的頁面，都使用 /opt/nginx/html 這個目錄，如此一來就能夠讓 Certbot 把驗證用的檔案放在這裡面，讓 Let’s Encrypt 可以驗證到。

完成以上設定後，可以用以下指令檢查設定檔有沒有問題
```
sudo /opt/nginx/sbin/nginx -t
```

如果沒問題可以用以下指令重新啟動 Nginx ( 使用這指令需要設定 systemd 功能 )
```
sudo systemctl restart nginx
```

## Capistrano

### Capistrano 功能介紹

Capistrano 是自動化部屬的工具，如果少了 Capistrano 我們要部署遠端伺服器就得每個動作都自己來，如果有多台機器多個環境，就更會需要 Capistrano 來幫忙處理各種繁瑣的任務了！

Capistrano 支援 Multiple stages，stage 指的是環境( staging, production 之類的)，我們只需要定義一次部署的方式，就能將這樣的部署方式套用在多個環境，也就是將 Web Application 部署到不同環境卻不用重複寫相同的設定檔，只需要設定這些環境所不同的地方譬如 IP 位置就可以了！


Capistrano 也支援 Server roles，在一個環境下我們可能會定義多個 role，因為 Web Application 可能是由多個服務一起完成的，有跑 Web Application 、database 和 CronJob server 等等，每個 role 在部署時會有不同的流程，這些也可以透過定義什麼 role 需要執行甚麼來完成。


### 部署架構

Capistrano 部署到機器後結構大概會如下

```
├── current -> /home/deploy/do_deploy.com/releases/20150120114500/
├── releases
│   ├── 20150080072500
│   ├── 20150090083000
│   ├── 20150100093500
│   ├── 20150110104000
│   └── 20150120114500
├── repo
│   └── <VCS related data>
├── revisions.log
└── shared
    └── <linked_files and linked_dirs>
```

Capistrano 在每次部署時都會產生一個新的資料夾在 releases 目錄下，新的資料夾裡面就是新版本的 code 而資料夾名稱也是用當時時間命名。所以 releases 目錄下就會有每次部署時程式碼。current 資料夾是目前最新版本的程式碼資料夾，他的來源其實就是用 symlink 指到 releases 資料夾中最新版本的程式資料夾！

在我們的專案中有一些資料我們可能希望他能在不同的 release 中都能使用同樣的資料，就譬如 `20150080072500`、`20150090083000` 等資料夾中的 log 目錄想用同一個，這時候我們會把 log 目錄放在 shared 資料夾中，同時一些敏感資料如 database.yml 或是 settings.yml 這些不會放在版本控制的檔案，也會放在 shared 資料夾中，要怎麼放進去呢？那就要在設定檔中設定啦！

### 設定檔

Capistrano 的相關檔案在專案中的位置如下：

```
├── Capfile
├── config
│   ├── deploy
│   │   ├── production.rb
│   │   └── staging.rb
│   └── deploy.rb
└── lib
    └── capistrano
            └── tasks
```
>設定檔部分會依序介紹 Capfile、deploy.rb、prdouction.rb 這三份檔案

Capfile 的設定會決定你的 Capistrano 可以使用哪些功能，這些功能也就是你能使用的 Capistrano 指令有哪些，以及在跑這些指令的時候會做什麼事。

將部分註解都拿掉，一份全新的 Capfile 設定檔可能長這樣

```
# Load DSL and set up stages
require "capistrano/setup"

# Include default deployment tasks
require "capistrano/deploy"

require "capistrano/scm/git"
install_plugin Capistrano::SCM::Git

# require "capistrano/bundler"
# require "capistrano/rails/assets"
# require "capistrano/rails/migrations"
# require "capistrano/passenger"

# Load custom tasks from `lib/capistrano/tasks` if you have any defined
Dir.glob("lib/capistrano/tasks/*.rake").each { |r| import r }
```
為了部署 Rails App 同時讓我們管理設定檔方便，我會在 Capfile 加上

```

require "capistrano/bundler"
require "capistrano/rails/assets"
require "capistrano/rails/migrations"
require "capistrano/passenger"
require 'capistrano/upload-config'
```

這樣我們就有了專門針對 Rails 部署設計的功能，至於到底會有什麼差別呢？我們可以在使用 cap production deploy 的時候知道，詳細情況會在常用指令介紹


---

接下來 deploy.rb 還有 production.rb 分別代表 global 的設定還有 stage specific 的設定，這是什麼意思呢？我們部署時可能有很多種環境，staging、production 等等之類的，如果有些設定是所有環境都套用的就是 global 的設定，如果是專門放在某個環境的就是 stage specific 的設定。其中 deploy.rb 裡面就是 global 的設定，而 config/deploy/ 這目錄下的 xxx.rb 就是 stage specific 的設定唷！

config/deploy.rb
> global 設定

```
lock "~> 3.10.1"

# 設定專案名稱
set :application, "do_deploy"

# 讓機器知道要到哪裡找我們的程式碼
set :repo_url, "git@github.com:shes50103/do_deploy.git"

# 放在 shared 中，那些不在版本控制中的檔案
append :linked_files, "config/database.yml", "config/secrets.yml", "config/settings.yml"

# 放在 shared 中，不同 release 之間共享的目錄
append :linked_dirs, "log", "tmp/pids", "tmp/cache", "tmp/sockets", "public/system"
```

config/deploy/production.rb
> stage specific  設定

```
# 設定部署 Rails 的模式
set :rails_env, :production

# 決定要部署版本控制庫的哪一個 branch
set :branch, 'master'

# 決定部署到遠端伺服器哪個地方，從絕對路徑寫起
set :deploy_to, '/home/deploy/do_deploy.com'

# 見下方說明
role :web, %w{deploy@178.128.xxx.xxx}
role :db, %w{deploy@178.128.xxx.xxx}
```
>其實如果你喜歡的話，把所有 deploy.rb 裡面的東西都丟到 production.rb 裡面也是跑得動的，問題就會是假如到時候想要部署 staging 的時候， staging.rb 也得寫一大堆東西

整個部署流程，我們需要在本地電腦用 ssh 連線到遠端伺服器，要連線上去需要有遠端伺服器的 IP 以及我們要連線上去的身份，在這裡我們使用一個叫做 `deploy` 的使用者連線上去，意思就是在遠端伺服器裡面有這麼一個使用者，同時他的`~/.ssh/authorized_keys` 裡面有我本地電腦 ssh 公鑰，這樣我才能連線上去！

此外，當我以 deploy 使用者連線上遠端伺服器後，我還需要在遠端伺服器中再連線到程式碼倉庫，所以我也必須把 deploy 使用者的公鑰放到程式碼倉庫才能完成部署。記得前面提到的 Capistrano 的特性 Server roles 嗎？這邊看到的 `role :web` 還有 `role :db` 就是我們在部署時指定的 role 唷！


誒！？
怎麼感覺只設定了一點點東西，其實設定檔還有很多其他參數可以調整，但都有預設值了，譬如

```
# Releases 目錄下最多保留五份程式碼
set :keep_releases, 5

# 預設版本控制是使用 Git
set :scm, :git
```

因為已經有預設值了，所以可以少設定很多東西，我們就特別對我們要設定參數調整即可！
更多詳細資料可以看[官方文件](https://capistranorb.com/documentation/getting-started/configuration/)


### 常用指令

完成設定檔後，我們總需要在終端機下指令來部署或是操作些什麼，要怎麼知道我們現在的設定能使用哪些指令呢？那就是下面這個

```
bundle exec cap -T
```

可以查詢目前 Capistrano 中能使用的指令，能使用的指令有哪些就取決於 Capfile 中 require 了多少東西，可以試試看把一些 require 的設定註解掉後再輸入一次 `bundle exec cap -T` 能使用的指令會變少

---

```
bundle exec cap production config:pull
bundle exec cap production config:push
```

要使用這兩個指令必須安裝 `gem 'capistrano-upload-config'` 以及在 Capfile 中加入 `require 'capistrano/upload-config'` ，這兩個指令就是把遠端伺服器中的 linked_files 檔案抓下來以及上傳上去

---



```
bundle exec cap production deploy:check
```

這個指令會幫忙檢查要部署的專案目錄( `deploy_to設定的那個` )、 linked_dirs 、 linked_files 這些是否存在，如果不存在它會幫我們把目錄的部分做出來，所以在部署專案的一開始都會使用這指令來幫我們把 shared 目錄建立好，這樣才能用 `cap production config:push` 將設定檔上傳上去！

---

接下來這個就是部署的指令啦！

```
bundle exec cap production deploy
```

到底會做什麼呢？取決於設定檔的寫法，如果 Capfile 有了這兩行

```
require "capistrano/setup"
require "capistrano/deploy"
```

在 deploy 的時候就會跑過以下步驟

```
git:wrapper
git:check
deploy:check:directories
deploy:check:linked_dirs
deploy:check:make_linked_dirs
git:clone
git:update
git:create_release
deploy:set_current_revision
deploy:symlink:linked_files
deploy:symlink:linked_dirs
deploy:symlink:release
deploy:cleanup
deploy:log_revision
```

如果要部署 Rails 這可能還不夠，我們會需要在 Capfile 內加上

```
require "capistrano/bundler"
require "capistrano/rails/assets"
require "capistrano/rails/migrations"
```

這樣在跑 deploy 的時候就能增加這幾個步驟

```
bundler:install
deploy:assets:precompile
deploy:assets:backup_manifest
deploy:migrate
deploy:migrating
```

> 這五個步驟會在 `deploy:symlink:linked_dirs` 和 `deploy:symlink:release` 之間

在使用 Capistrano 比較常卡關的問題可能就是有些步驟是只有特定的 role 才能執行，這時候就要另外設定，又或者是有些東西的安裝我們可能需要額外指定明確路徑，不然就會安裝在錯的地方。更詳細的部署流程可以參考[這裡](https://capistranorb.com/documentation/getting-started/flow/)

## 結論

在學習部署的過程中，儘管已經完整走完了部署的流程讓服務可以正常執行，但是對這些部署工具的掌握還是十分有限，有很多東西只知道出問題了要這樣做、要那樣試可能可以解決問題，但往往不是很清楚自己到底在幹嘛XD，所以趁這個機會整理了一下這些工具的使用方法。
