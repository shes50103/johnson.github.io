---
:title: 自動產生 Docker，一鍵佈署不是夢 ！？
:date: 2019-08-03 00:00:00 UTC
:slug: bankai-docker
:author: Johnson
:tags: 5xruby, 心得分享, 五倍紅寶石
:category: techpost
:image:
  :original: post/image/2019-08-03-bankai-docker/cover.jpg
  :preview: post/image/2019-08-03-bankai-docker/cover.jpg
  :thumb: post/image/2019-08-03-bankai-docker/cover.jpg
---
> 圖片作者：snoku，
> [圖片來源連結](https://pixabay.com/photos/construction-site-crane-pier-1156567/)

想要一鍵部署！？首先要一鍵產生 Docker Image！要產生 Docker Image 需要有個 Dockerfile，通常在寫 Dockerfile 時從選擇 base 的 Image 到 Container 啟動後要執行的指令都需要一行一行設定。不僅如此，如果想要優化 Docker Image 的大小，還需要使用許多的技巧...突然覺得，要做這些事也太多！有沒有現成的 gem 可以幫我們做到這些事呢？

有的！就是五倍的同事[蒼時弦也](https://blog.frost.tw/)開發的 [bankai-docker](https://github.com/5xRuby/bankai-docker) 這工具啦！其實這接下來就是要來介紹一下 bankai-docker 這個專案，從怎麼使用它到它的部分功能怎麼做出來的，都將一一為您介紹。

> 文章內容可能會需要一些 Docker 跟 Ruby DSL 相關的知識，對這些內容有興趣可以參考這兩篇文章

> [用 Docker 部署 Rails，原來是這樣！？](https://5xruby.tw/posts/deploying-your-docker-rails-app/) 和 [Ruby 探索：Blocks 深入淺出](https://5xruby.tw/posts/discover-ruby-block/)

---
### 目錄

*  bankai-docker 介紹
*  使用說明書
*  如何偵測專案下有哪些 gem ？
*  把需要的 Package 印在 Dockerfile 上吧！
*  啟動 Docker Container
*  心得

---

### bankai-docker 介紹

bankai-docker 可以讓我們用一行指令 `rake docker:build` 自動產生 Rails 專案的 Docker Image，同時它也有 DSL 可以客製化產生的 Docker Image，預設情況下 bankai-docker 產生的 Docker Image 會包含 Ruby, Node.js, Yarn 還有 Rails 專案下需要的 gem，因為有優化 Image 的大小的關係，預設情況下 Docker Image 大小會落在 180 MB 上下。

就讓我們來研究一下 bankai-docker 是如何產生 Docker Image 的吧！這篇文章的講解，我使用的環境是：

```
Rails：5.2.3
Ruby：2.6.0
資料庫：postgresql
然後額外安裝以下 gem
gem 'pg'
gem 'bankai'
gem 'bankai-docker'
```

> 目前 `bankai-docker` 還需要額外裝 `bankai` 才能正常使用


### 使用說明書

bankai-docker 透過 Ruby 的 DSL 讓我們可以客製化產生的 Docker Image，輸入`rails generate bankai:docker:install` 產生 `config/docker.rb` ，我們可以透過編輯 `config/docker.rb` 來產生不同的 Docker Image，以下是預設情況下 `config/docker.rb` 的內容

```ruby
# frozen_string_literal: true

# rubocop:disable Metrics/BlockLength
Bankai::Docker.setup do
  runtime_package 'tzdata', 'libstdc++', 'git'

  stage :gem do
    package 'build-base', 'ca-certificates', 'zlib-dev', 'libressl-dev',
            runtime: false

    run 'mkdir -p /src/app'

    add 'Gemfile', '/src/app'
    add 'Gemfile.lock', '/src/app'

    workdir '/src/app'
    run 'bundle install --deployment --without development test ' \
        '--no-cache --clean && rm -rf vendor/bundle/ruby/**/cache'

    produce '/src/app/vendor/bundle'
    produce '/usr/local/bundle/config'
  end

  stage :node, from: 'node:10.15.2-alpine' do
    run 'mv /opt/yarn-v${YARN_VERSION} /opt/yarn'

    produce '/usr/local/bin/node'
    produce '/opt/yarn'
  end

  main do
    run 'mkdir -p /src/app'

    env 'PATH=/opt/yarn/bin:/src/app/bin:$PATH'
    env 'RAILS_ENV=production'

    add '.', '/src/app'
    workdir '/src/app'

    expose 80

    entrypoint 'docker-entrypoint'
    cmd 'serve'
  end
end
# rubocop:enable Metrics/BlockLength

```

可以透過 `rake docker:preview` 看看 `config/docker.rb` 產生的 Dockerfile 長什麼樣子

```
FROM ruby:2.6.0-alpine AS gem
RUN apk add --no-cache postgresql-dev build-base ca-certificates zlib-dev libressl-dev

RUN  mkdir -p /src/app
ADD  Gemfile /src/app
ADD  Gemfile.lock /src/app
WORKDIR  /src/app
RUN  bundle install --deployment --without development test --no-cache --clean && rm -rf vendor/bundle/ruby/**/cache

FROM node:10.15.2-alpine AS node


RUN  mv /opt/yarn-v${YARN_VERSION} /opt/yarn

FROM ruby:2.6.0-alpine
RUN apk add --no-cache tzdata libstdc++ git postgresql-libs
COPY --from=gem /src/app/vendor/bundle /src/app/vendor/bundle
COPY --from=gem /usr/local/bundle/config /usr/local/bundle/config
COPY --from=node /usr/local/bin/node /usr/local/bin/node
COPY --from=node /opt/yarn /opt/yarn
RUN  mkdir -p /src/app
ENV  PATH=/opt/yarn/bin:/src/app/bin:$PATH
ENV  RAILS_ENV=production
ADD  . /src/app
WORKDIR  /src/app
EXPOSE  80
ENTRYPOINT  ["docker-entrypoint"]
CMD  ["serve"]
```

大概看一下這個 Dockerfile 可以知道，為了優化 Image 的大小，我們用了 3 個 stage 來產生我們要的 Image，一個是安裝 Rails 所需要的 gem，另一個是裝 Node.js 的，在處理完這兩個步驟後，只需要把必要的檔案放到主要的 stage 讓它產生 Image 即可。

稍微比對一下可以發現，幾乎 `config/docker.rb` 的每一行都可以對應到它產生的 Dockerfile，唯有以下兩行多裝了 postgresql？

```
RUN apk add --no-cache tzdata libstdc++ git postgresql-libs

RUN apk add --no-cache postgresql-dev build-base ca-certificates zlib-dev libressl-dev
```

`config/docker.rb` 中完全沒有做 `postgresql` 的相關設定，但產生的 Dockerfile 卻自動裝好了 `postgresql`？這就是 bankai-docker 貼心的地方啦，它會去偵測 Gemfile 中有使用的 gem 然後安裝對應的 Package，譬如我的 Gemfile 有使用 `gem 'pg'` ，Dockerfile 上就會加上 postgresql 的 Package，這是哪裡設定的呢？bankai-docker 的 rake 執行的時候，第一步就是執行以下程式

```ruby

Bankai::Docker.setup do
  detect_package :database, :gem do |package|
    if gem?('pg')
      package.add_dependency 'postgresql-dev', runtime: false
      package.add_runtime_dependency 'postgresql-libs'
    end

    if gem?('mysql2')
      package.add_dependency 'mariadb-dev', runtime: false
      package.add_runtime_dependency 'mariadb-client-libs'
    end
  end

  detect_package :sassc, :gem do |package|
    package.add_runtime_dependency 'libstdc++' if gem?('sassc')
  end

  detect_package :mini_magick, :gem do |package|
    package.add_runtime_dependency 'imagemagick' if gem?('mini_magick')
  end
end
```
> 上面的 code 是原始碼的 `templates/auto_packages.rb` 檔案，這是目前 master 的程式，如果直接 `gem install 'bankai-docker'` 載下來的套件可能跟這個不太一樣

> 再研究 gem 的過程很容易把原始碼玩壞，而且 gem 裡面也沒有 git 可以復原，這時候可以用 `gem pristine bankai-docker` 來讓 gem 復原唷！

其實這邊的 DSL 就是一個一個去檢查 Gemfile 有裝哪些 gem ，用 `gem?` 方法檢查，如果有裝就用 `package.add_dependency` 或是 `package.add_runtime_dependency` 把對應的 Package 裝在 Dockerfile 裡面，看到這邊可能會好奇，誒！？就只有這麼幾行嗎？如果我還有其他需要裝 Package 的 gem 也在 Rails 專案中，這時 bankai-docker 產生的 Dockerfile 會有那個 Package 嗎？答案是不會ＸＤ，需要在 `config/docker.rb` 另外設定才能裝其它 Package。

### 如何偵測專案下有哪些 gem ？

剛剛前面說的用 `gem?` 方法檢查 Gemfile 有安裝哪些 gem ，這個是怎麼做到的呢？其實在 bankai-docker 的 `lib/bankai/docker/dsl/gemfile.rb` 檔案就是在做這件事

```ruby
# frozen_string_literal: true

module Bankai
  module Docker
    module DSL
      # Gemfile detect
      module Gemfile
        def gem?(name)
          gemfile.match?(/^\s*gem .#{name}./)
        end

        private

        def gemfile
          @gemfile ||= ::File.read(Rails.root.join('Gemfile'))
        end
      end
    end
  end
end
```

使用 `@gemfile ||= ::File.read(Rails.root.join('Gemfile'))` 讀取專案中的 Gemfile 檔案，把它以 string 的狀態儲存在 @gemfile 變數中。接下來要找出 Gemfile 是否有安裝這個 gem，在這邊我們使用 `match?` 方法，裡面接的參數 `(/^\s*gem .#{name}./)` 是正規表達式，可以判斷 @gemfile 中是否有符合規則的字串，有的話就是符合 `match?` 的規則，正規表達式中的 `^\s*gem` 代表

> 從一行程式的開始可以有一個或多個空白，然後接上 `gem` 這個字

為什麼要這樣設定呢？因為如果有些 gem 被註解掉像是`# gem 'some_gem'`，這時候要判斷它不符合我們要找的規則，因為他不是 Gemfile 最後會裝的 gem。以上就是如何用 `gem?` 方法檢查 Gemfile 是否有安裝指定的 gem。


### 把需要的 Package 印在 Dockerfile 上吧！

可以透過 `gem?` 方法偵測專案使用哪些 gem 以後，要如何透過 `add_dependency` 方法將需要的 Package 印在 Dockerfile 上面呢？使用 `add_dependency` 的時候會呼叫 `current_stage` 或是 `stage(:main)` ，他們都是回傳實體變數 `@stages` 的方法，接著我們會在實體變數 `@stages` 上再使用 `concat` 把要註冊的 Package 存進去。

```ruby
def add_dependency(*packages, runtime: true)
  current_stage.concat(packages)
  stage(:main).concat(packages) if runtime
end
```

完成以上工作後，就要用 template 來幫忙印出 Dockerfile， bankai-docker 中的 `templates/stage.erb` 是其中一個 template，在裡面用到 `<%= package_command %>
` 來處理 Dockerfile 裡面的 `RUN apk add --no-cache ...` 這行設定，就讓我們來看看 `package_command` 這方法怎麼做出來的吧！

```ruby
def package_command
  return unless Package.any?(@name)

  "RUN #{Package.command_for @name}"
end
```

在 `Package.any?(@name)` 確認完 Package 的 `@stages` 不是空的後就用  `Package.command_for @name` 印出剛剛放在 `@stages` 裡面的 Package 套件名稱，以目前的例子
```
"RUN #{Package.command_for @name}"
```

將會變成

```
"RUN apk add --no-cache postgresql-dev build-base ca-certificates zlib-dev libressl-dev"
```

> 更多更詳細 bankai-docker 中的 Package 的用法要看 `lib/bankai/docker/package.rb` 檔案

以上就是簡略地說明 `add_dependency` 方法如何印 Package 的設定到 Dockerfile 上的，如果要完整知道過程還是需要把 bankai-docker 的 gem 拉到自己電腦跑跑看會比較清楚XD

### 啟動 Docker Container

講完 bankai-docker 大概是用什麼樣的方式做出來之後，另一個很重要的事情是，要怎麼讓這個 Docker Image 可以產生 Docker Container 跑起來？至少要跑起來才知道這 Image 是不是能用的，這邊我是用自己寫的 `docker-compose.yml` 來做的，以下是可以在本地端跑起來的設定

```
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
    image: $(whoami)/[RAILS_APP_NAME]
    ports:
      - "3000:80"
    environment:
      - DATABASE_URL=postgres://postgres:postgres@db/postgres
      - SECRET_KEY_BASE=xxxxxx
      - RAILS_SERVE_STATIC_FILES=true
    networks:
      - johnson
    depends_on:
      - db
    command: "rake db:migrate && rake assets:precompile && bundle exec puma -p 80"

networks:
  johnson:

```

如果你也想玩玩看用 Docker 跑 Rails 專案，可以設定一個 postgres 的 Container 來搭配。為了讓我 Rails 的 Container 跑起來之後，可以執行 `rake db:migrate` ，我在這邊用 `docker-compose.yml` 的 commond 設定 Container 跑起來後要執行的指令為 `rake db:migrate && rake assets:precompile && bundle exec puma -p 80`，這樣就能把 bankai-docker 產生的 Image 用 Container 在本地跑起來囉！

### 心得

研究 bankai-docker 過程，讓我更熟悉怎麼透過 multi-stage builds 的方式產生 Docker Image，要怎麼拆 stage？有哪些東西是要帶到主要的 stage 使用等等。同時，bankai-docker 中大量使用 DSL 的技巧，透過 block 的傳遞來讓使用者有更大的彈性來訂製 Dockerfile。目前 bankai-docker 還有許多功能仍在開發中，自動偵測哪些 gem 需要額外裝 Package 也是其中之一，上面的會偵測 `mini_magick` 的部分也是我寫這篇文章期間加上去的，看到自己加的功能讓有切圖功能的 Rails 專案用 Docker Container 跑起來後真的蠻有成就感的！希望未來有更多人可以對 bankai-docker 有興趣，一起來研究或是使用看看～
