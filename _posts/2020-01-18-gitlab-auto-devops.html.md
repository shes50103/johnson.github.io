---
:title: GitLab Auto DevOps 深入淺出，自動部署，連設定檔不用？！
:date: 2020-01-18 00:00:00 UTC
:slug: gitlab-auto-devops
:author: Johnson
:tags: 5xruby, 心得分享, 五倍紅寶石, Rails, GitLab, Docker, Kubernetes, CI/CD, Auto Devops
:category: techpost
:image:
  :original: post/image/2020-01-18-gitlab-auto-devops/gitlab-auto-devops.jpg
  :preview: post/image/2020-01-18-gitlab-auto-devops/gitlab-auto-devops.jpg
  :thumb: post/image/2020-01-18-gitlab-auto-devops/gitlab-auto-devops.jpg
---
> 圖片作者：MustangJoe
> [圖片來源連結](https://pixabay.com/photos/gears-cogs-machine-machinery-1236578/)


開發網站過程中，環境部署或是寫設定檔是一件不簡單的事。比起寫程式，寫設定檔不是開發過程中那麼「頻繁」需要做的事情，所以熟練度跟寫程式比起來多少有落差。

每次要部署或是做 CI / CD 相關的設定時，幾乎都是拿手邊的設定檔來修修改改，如果沒有什麼太特殊的功能，每個專案要做的設定也不會差太多。有沒有什麼服務可以幫我們把這些事做好，讓我們把程式推到 GitHub / GitLab 上的時候，就自動幫我們跑測試或是部署？

在以往只想得到 Heroku 的服務能自動部署，然而 GitLab Auto DevOps 也是能在沒什麼設定的情況下，幫忙我們跑完 CI / CD 的流程！

---
### 目錄

* 深入淺出
  * 什麼是 GitLab Auto DevOps
  * 工具之間的關係
  * GitLab CI/CD 介紹
* 只要五十元！？沒有設定檔的 Auto Devops
  * 快速建立 Rails 專案
  * 用 GKE 設定 Cluster 好輕鬆
  * 誒！又是設定檔！？
* 動手實作 Auto Devops
  * Build: 打包 Docker Image
  * Deploy: 部署/更新 Cluster
* 心得
* 參考資料


---

> 文章前半段會介紹使用到的工具，和他們之間的關係，後半段會介紹自己寫 `.gitlab-ci.yml` 來客製化 Auto Devops


## 什麼是 GitLab Auto DevOps

GitLab 的 Auto DevOps 透過預先定義好的設定檔，來做到自動偵測、測試、部署等等工作，甚至讓我們可以用設定 GitLab 環境變數的方式，就一定程度調整 CI / CD 流程，是不是很方便呢？

但要使用 Auto DevOps 之前，他有四個前置任務

- Kubernetes (以下簡稱 K8S)
  - 一個 K8S 的 Cluster，Auto DevOps 將會把網站部署到這個 Cluster，最簡單的方式是透過 Google 的 GKE 服務來設定
- Base domain
  - 需要有一個 wildcard 的 DNS 讓部署在這個環境的網站有 Domain name
- GitLab Runner
  - 一個可以跑 Docker 的 GitLab Runner，將會為由它來執行 CI / CD 的流程。如果用官方 GitLab 預設下可以直接使用官方的 shared Runners。
- Prometheus (選用)
  - 如果要使用 Auto Monitoring 功能的話才需要裝它，細節可以再看[官方文件](https://docs.gitlab.com/ee/topics/autodevops/)說明XD

完成以上任務並稍微設定 GitLab 後，再來就是把一份「沒有」gitlab-ci.yml 設定檔的專案推到 GitLab 上就能在 CI / CD Pipelines 看到 Auto DevOps 的成果囉！

> 為什麼是「沒有」gitlab-ci.yml 設定檔的專案呢？其實 Auto DevOps 就是一份官方寫好的 gitlab-ci.yml，在啟動 Auto DevOps 的專案裡，如果找不到 gitlab-ci.yml 檔，那就會直接用官方 gitlab-ci.yml 去跑 CI / CD 流程。因此，如果想要客製化調整 Auto DevOps 也可以拿官方的 gitlab-ci.yml 來改！


## 工具之間的關係

在摸索 Auto DevOps 過程中，最困難的地方就是搞不清楚每個工具、服務之間的關係，因為篇幅有限，我知道的也有限，所以就簡單說明它們之間的關係！更多內容說明請參考最下面列的參考資料。

> 接下來會依序介紹 Docker、Kubernetes、Helm 和 GitLab Runners 這些工具
> 如果已經有概念可以跳過


### Docker 介紹
Docker 是一個容器化技術，可以把 Process 隔離到獨立的空間，讓我們更好限制管理 Process 的資源、權限等。我們稱這個被隔離的 Process 和空間為 Docker Container `(以下簡稱 Container)`，要啟動 Container 需要有 Docker Image `(以下簡稱 Image)`，它們之間的關係有點像寫程式時，類別跟實體的關係，有了 Image 就能啟動 Container。我們可以透過 Dockerfile 客製化自己的 Image，而 Register 則是存放 Image 的倉庫。


### Kubernetes 介紹
在一個複雜的系統裡，要如何管理那麼多 Container 呢？這時候就需要  Kubernetes 了！`(以下簡稱 K8S)`，K8S 可以自動化地部署及管理多台機器上的多個 Container，除了部署，它還有偵測並重新啟動系統中故障的容器等功能。在 K8S 中我們需要知道有以下幾個元件：

- Pod 是 K8S 中可以被部署的最小元件，一個 Pod 是由一到多個 Container 組成，同個 Pod 的不同 Container 之間彼此共享網路資源。 每個 Pod 都會有它的 `yaml` 檔，用以描述 Pod 會使用的 Image 還有連接的 Port 等資訊。

- Node 就是一台機器、一台電腦或可以是 AWS 上的一個 EC2，在上面可以跑多個 Pod。其中 Node 又分成 Worker Node 和 Master Node 兩種，前者是實際運行 Pod 的機器，後者負責管理 Worker Node。

- Cluster 是 K8S 中一個或多個 Node 的集合。本文會直接用 GCP 的 Google Kubernetes Engine (GKE) 服務來啟用 K8S Cluster。

![](https://i.imgur.com/r8MfjOK.png)

可以在上圖看到 Cluster 、 Node 還有 Pod 之間的關係，在 K8S 中可以用 `kubectl` 指令對 cluster 操作，使用 `kubectl` 後會透過 Master Node 上的 API server 收到來自使用者終端機的指令，再繼續透過 API server 來操作 Worker Node。

### Helm 介紹

除了上述元件外，K8S 有大量的 resource object 像是可以做橫向擴展、控制 Pod 數量的 Deployment 或是控制 Pod 連線 port 的 Service 等等，要管理那麼多的 resource object 可能需要很多設定檔，並且使用很多次 `kubectl` 指令，如果系統複雜起來將難以管理！因此這時候就可以用 Helm 這個 K8S 的套件管理工具。


Helm 和 K8S 的關係圖：


![](https://i.imgur.com/LOYsA4Y.png)


Helm 透過參數 (parameter) 跟模板 (template) 的方式，讓我們可以在只修改參數的方式重複利用模板。如上圖透過 `values.yaml` 設定參數再注入 `deployment.yaml` 這些模板中，最後透過 helm install 這類指令，把設定打包成一個 chart 並部署到 K8S cluster。使用 helm 的時候還需要在 Cluster 中建立 Tiller，他就像前面說到 `kubectl` 負責跟 API Server 溝通。


### GitLab Runners 介紹

使用 GitLab 有兩種選擇，自己架 GitLab Server 或是註冊 [gitlab.com](https://gitlab.com/) 直接用官方的服務，本文將使用官方 gitlab.com 並搭配它免費的 Shared Runners 來跑我們 CI/CD 的 Job。

誒！什麼是 Shared Runners 呢？為了要有 CI CD 的功能我們會把 `.gitlab-ci.yml` 放在專案的根目錄裡， GitLab 會依造 `.gitlab-ci.yml` 的設定產生 CI/CD Pipeline，每個 Pipeline 裡面可能有多個 Job，這時候就會需要有 GitLab Runner 來執行這些 Job 並把執行的結果回傳給 GitLab 讓它知道這個 Job 是否有正常執行。同樣的 Runner 也可以自行架設，或是在使用 gitlab.com 時，也能直接用官方的 Shared Runners。


### 他們之間的關係

介紹完 GitLab 還有 K8S 後，那麼 Auto Devops 到底是怎麼部署的呢？


![](https://i.imgur.com/M7ARihQ.png)


再推了一個 commit 到 GitLab 後，依據 `.gitlab-ci.yml` 產生了對應的 CI/CD Pipeline，接著 Runner 就會去找 Pipeline 上的工作來執行，為了方便建立各種環境，很多 job 會用 Docker 的 Container 來跑，譬如把專案打包成 Docker Image 這工作又或是 helm 的操作都會在 Container 內執行。最後有了專案的 Image 以及設定好 helm chart 後，就會把它部署到 Cluster 上了！


> 這樣介紹可能太簡略了，更多細節會在下方實作處再說明

## GitLab CI/CD 介紹

為接下來的 Auto Devops 做準備，我們需要對 `.gitlab-ci.yml` 的設定檔有一些了解，CI/CD Pipeline 是由 stage 還有 job 組成的，stage 是有順序性的，前面的 stage 完成後才會開始下一個 stage。如下圖，在有兩個 stage 的 Pipeline 時，如果上一個 stage 失敗，下一個 stage 就不會啟動。


![](https://i.imgur.com/lw3whTq.png)

每個 stage 裡面包含一到多個 Job，像是 `stage_a` 有 job1, job2 而 `stage_b` 有 job3 如下圖：


![](https://i.imgur.com/eTfLrum.png)

![](https://i.imgur.com/JKVPpCR.png)




要如何做出上圖的 Pipeline 呢？可以透過下面這個 `gitlab-ci.yml`

```
stages:
  - stage_a
  - stage_b

job1:
  stage: stage_a
  script:
    - echo 'this is stage a job 1'

job2:
  stage: stage_a
  script:
    - echo 'this is stage a job 2'

job3:
  image: postgres:latest
  stage: stage_b
  script:
    - echo 'this is stage b job 3'
    - psql --version
```

在上圖可以看到有兩個 stage 依序為 `stage_a` 再 `stage_b`，每個 job 可以設定哪個 stage 會用這 job。另外可以注意 job3 設定哪個 Image 來啟動這個 job ，所以 job3 就是在 postgres Container 內運行的，也因此可以印出 postgres 的版本，Auto Devops 裡也會大量用到這種在指定 Container 內運行的工作。

![](https://i.imgur.com/s9cVu6y.png)




# 只要五十元！？沒有設定檔的 Auto Devops

介紹完工具之間關係後，終於可以來使用 GitLab 的 Auto Devops 了！我們會先練習一次沒有設定檔的版本，完成 GKE 跟 GitLab 基本設定然後部署成功後，才會進入到客製化 Auto Devops 的練習。


在最剛開始學習 Auto Devops 時，因為聽說是 K8S 整合 GitLab 的服務，所以知道都在本機跑十分麻煩，但要租用機器又需要白白燒錢，幸好這次練習中大部分服務都是免費或是有充足試用額度的，先是註冊 [gitlab.com](https://gitlab.com/) 官方的服務，再搭配 GCP 的 Google Kubernetes Engine (GKE) 啟用 K8S Cluster，GCP 有 300 美元的試用額度，應該是蠻夠練習了。


最後是本文唯一需要花錢的地方，申請 wildcard 的 DNS ，我在 [godaddy](https://tw.godaddy.com/) 找便宜的可能需要 50 元這樣。只要一份雞排的錢就能開始囉💪





> 以下練習我用的 DNS 是 *.thejohnsonhi.site


## 快速建立 Rails 專案

再來是要推上 GitLab 上面的專案，我用以下指令建出名為 auto_example 的 Rails 專案

```
# -T 省略測試
# -d 指定使用的資料庫是 postgresql
rails new auto_example -T -d postgresql
```
> 版本資訊
> Rails 6.0.2.1
> Ruby 2.6.5

然後用 scaffold 做出一些基本頁面

```
rails generate scaffold user title start:time
```


完成後還需要在 `config/routes.rb` 新增這路徑，確保首頁有內容可以通過 health checks

```
  root 'users#index'
```

## 用 GKE 設定 Cluster 好輕鬆


在 GitLab 開一個 Public 的專案

> 開 private 的話還要注意使用 Container Registry 的權限問題

![](https://i.imgur.com/fGnXRDx.png)

到 Operations > Kubernetes 選擇 GKE

![](https://i.imgur.com/1rRgaVM.png)

我選擇 1 個 n1-standard-1 大小的機器


![](https://i.imgur.com/tb3wdTw.png)


填入申請好的 wildcard 的 DNS

![](https://i.imgur.com/Jo9EUsr.png)


預設情況下，我們會啟用 GitLab-managed cluster 也就是 GitLab 可以幫我們在 Cluster 內安裝一些元件，這邊就一路把 Helm Tiller、Ingress、Cert-Manager、Prometheus 和 GitLab Runner 都裝起來吧

>可以注意到 Ingress 裝起來後會得到一組 IP ，也要記得設定 DNS 指到這組 IP

![](https://i.imgur.com/IiyRdi3.png)

再來是到 Settings > CI/CD 的 Auto DevOps 去勾選啟用

![](https://i.imgur.com/hsZpWga.png)

再來還需要設定幾個 CI/CD 需要的環境變數，如果是 Rails 搭配 postgres 最少需要設定這兩個

```
DB_INITIALIZE = RAILS_ENV=production /bin/herokuish procfile exec bin/rails db:setup
DB_MIGRATE = RAILS_ENV=production /bin/herokuish procfile exec bin/rails db:migrate
```

另外 Auto Devops 也提供只要設定環境變數就能一定程度客製化的選項，譬如我這邊設定就不會跑測試跟語法檢查

```
CODE_QUALITY_DISABLED = true
TEST_DISABLED = true
```

![](https://i.imgur.com/kbo9mTf.png)


完成以上步驟就把 Rails 專案推上來就能看到 Auto Devops 的 Pipeline 囉！

![](https://i.imgur.com/cFMXtn8.png)

然後變成

![](https://i.imgur.com/NqDrTgC.png)

所以說部署好的網站勒？可以到 Operations > Environments 裡面看到

![](https://i.imgur.com/QYyxqTB.png)

點下去然後看到畫面，YA 都沒有用設定檔就部署成功了！

![](https://i.imgur.com/fOndcJB.png)

希望大家最後有跑成功，假如有個萬一沒跑成功，可以先看 CI/CD 的 log 或是按 Retry 重新跑，如果還不行也能透過 GKE 的介面的 Cloud Shell 連到 K8S 內操作，以獲取更多資訊，可能會用到的指令有

```
# 查看 CLuster 內所有資源狀態
kubectl get all --all-namespaces
```

```
# log 出特定 Pod 的內容
# kubectl logs --namespace [namespace 名稱] [pod 名稱]
kubectl logs --namespace auto-example-16272281-production pod/production-b69b589b9-7x2vw
```

> 為了區別是誰的資源，這邊要特別注意 namespace 有沒有設定對，不然會找不到資料喔


## 誒！又是設定檔！？

為了役物而不役於物，這篇號稱不用設定檔的文章，接下來設定檔講好講滿...

在沒有 `.gitlab-ci.yml` 的情況下完成了 Auto Devops，如果想要進一步的客製化，而且是改 GitLab 環境變數都無法實現的客製化，這時候還是得回到 `.gitlab-ci.yml` 設定檔，首先來測試一下在專案新增長這樣的 `.gitlab-ci.yml`

```
image: alpine:latest

variables:
  POSTGRES_USER: user
  POSTGRES_PASSWORD: testing-password
  POSTGRES_ENABLED: "true"
  POSTGRES_DB: $CI_ENVIRONMENT_SLUG
  POSTGRES_VERSION: 9.6.2
  DOCKER_DRIVER: overlay2
  ROLLOUT_RESOURCE_TYPE: deployment
  DOCKER_TLS_CERTDIR: ""  # https://gitlab.com/gitlab-org/gitlab-runner/issues/4501

stages:
  - build
  - test
  - deploy  # dummy stage to follow the template guidelines
  - review
  - dast
  - staging
  - canary
  - production
  - incremental rollout 10%
  - incremental rollout 25%
  - incremental rollout 50%
  - incremental rollout 100%
  - performance
  - cleanup

include:
  - template: Jobs/Build.gitlab-ci.yml  # https://gitlab.com/gitlab-org/gitlab-foss/blob/master/lib/gitlab/ci/templates/Jobs/Build.gitlab-ci.yml
  - template: Jobs/Test.gitlab-ci.yml  # https://gitlab.com/gitlab-org/gitlab-foss/blob/master/lib/gitlab/ci/templates/Jobs/Test.gitlab-ci.yml
  - template: Jobs/Code-Quality.gitlab-ci.yml  # https://gitlab.com/gitlab-org/gitlab-foss/blob/master/lib/gitlab/ci/templates/Jobs/Code-Quality.gitlab-ci.yml
  - template: Jobs/Deploy.gitlab-ci.yml  # https://gitlab.com/gitlab-org/gitlab-foss/blob/master/lib/gitlab/ci/templates/Jobs/Deploy.gitlab-ci.yml
  - template: Jobs/DAST-Default-Branch-Deploy.gitlab-ci.yml  # https://gitlab.com/gitlab-org/gitlab-foss/blob/master/lib/gitlab/ci/templates/Jobs/DAST-Default-Branch-Deploy.gitlab-ci.yml
  - template: Jobs/Browser-Performance-Testing.gitlab-ci.yml  # https://gitlab.com/gitlab-org/gitlab-foss/blob/master/lib/gitlab/ci/templates/Jobs/Browser-Performance-Testing.gitlab-ci.yml
  - template: Security/DAST.gitlab-ci.yml  # https://gitlab.com/gitlab-org/gitlab-foss/blob/master/lib/gitlab/ci/templates/Security/DAST.gitlab-ci.yml
  - template: Security/Container-Scanning.gitlab-ci.yml  # https://gitlab.com/gitlab-org/gitlab-foss/blob/master/lib/gitlab/ci/templates/Security/Container-Scanning.gitlab-ci.yml
  - template: Security/Dependency-Scanning.gitlab-ci.yml  # https://gitlab.com/gitlab-org/gitlab-foss/blob/master/lib/gitlab/ci/templates/Security/Dependency-Scanning.gitlab-ci.yml
  - template: Security/License-Management.gitlab-ci.yml  # https://gitlab.com/gitlab-org/gitlab-foss/blob/master/lib/gitlab/ci/templates/Security/License-Management.gitlab-ci.yml
  - template: Security/SAST.gitlab-ci.yml  # https://gitlab.com/gitlab-org/gitlab-foss/blob/master/lib/gitlab/ci/templates/Security/SAST.gitlab-ci.yml

```

> [官方完整專案點我](https://gitlab.com/gitlab-org/gitlab-foss)

Auto Devops 就是去看專案中沒有 `.gitlab-ci.yml` 的話，就執行這個官方預設的 [.gitlab-ci.yml](https://gitlab.com/gitlab-org/gitlab-foss/blob/master/lib/gitlab/ci/templates/Auto-DevOps.gitlab-ci.yml) ，這是一份可以在很多環境下都跑的動的設定檔，他有許多 stage 和 template，如果要看更多設定要找註解起來的網址去看 template 裡面的設定。沒意外的話改成這份設定檔跑 CI/CD 也會通過喔！


![](https://i.imgur.com/gnnNwEH.png)


# 動手實作 Auto Devops


為了更了解裡面都做了什麼事，所以我參考了官方的 build 跟 production 這兩個 stage 的設定，自己客製化了一份 `.gitlab-ci.yml`，它有以下兩個 stage

![](https://i.imgur.com/ExxOahV.png)

上圖是 build 階段

1. 在 Docker in Docker 的環境用 Dockerfile 打包 Image
2. 把 Image 上傳 GitLab 的 Container Registry



![](https://i.imgur.com/F3rcBug.png)

上圖是 production 階段

1. 取得 build 階段打包的 Image
2. 取得 Chart Repository 上的 chart
3. 用 helm upgrade 把 chart 部署到 K8S 上

我的客製化 `.gitlab-ci.yml` 內容如下

```
variables:
  POSTGRES_USER: user
  POSTGRES_PASSWORD: testing-password
  POSTGRES_ENABLED: "true"
  POSTGRES_DB: $CI_ENVIRONMENT_SLUG
  POSTGRES_VERSION: 9.6.2
  DB_MIGRATE: 'RAILS_ENV=production /bin/herokuish procfile exec bin/rails db:migrate'
  TRACE: 'true'

stages:
  - build
  - production

build:
  stage: build
  image: "docker:stable"
  services:
    - docker:stable-dind
  script:
    - export CI_APPLICATION_REPOSITORY="$CI_REGISTRY/johnsonzhan/auto_example/master"
    - echo "Logging to GitLab Container Registry with CI credentials..."
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - docker image pull "$CI_APPLICATION_REPOSITORY:$CI_COMMIT_BEFORE_SHA" || \
    - docker build
      --cache-from "$CI_APPLICATION_REPOSITORY:$CI_COMMIT_BEFORE_SHA"
      --tag "$CI_APPLICATION_REPOSITORY:$CI_COMMIT_SHA" .
    - docker push "$CI_APPLICATION_REPOSITORY:$CI_COMMIT_SHA"

production:
  image: "registry.gitlab.com/johnsonzhan/auto_example/deploy"
  stage: production
  script:
    - auto-deploy preparation
    - auto-deploy deploy
  environment:
    name: production
    url: http://$CI_PROJECT_PATH_SLUG.$KUBE_INGRESS_BASE_DOMAIN
```

GitLab CI 的環境變數主要有三個來源，優先度高到低依序為

- Settings > CI/CD 介面定義的變數
- gitlab_ci.yml 定義環境變數
- GitLab 預設環境變數

> 更多環境變數說明可以參考[官方文件](https://docs.gitlab.com/ee/ci/variables/)

在這次練習中除了預設變數之外，其他變數都會放在 `gitlab_ci.yml` 方便說明

## Build: 打包 Docker Image

把專案打包成 Docker Image 首先需要在專案下新增一份 Dockerfile

```
FROM gliderlabs/herokuish as builder
COPY . /tmp/app
ENV USER=herokuishuser
RUN /bin/herokuish buildpack build

FROM gliderlabs/herokuish
COPY --chown=herokuishuser:herokuishuser --from=builder /app /app
ENV PORT=5000
ENV USER=herokuishuser
EXPOSE 5000
CMD ["/bin/herokuish", "procfile", "start", "web"]
```
> 我直接參考 Auto Devops 裡面的做法，用 herokuish 提供的 Image 來打包專案


再來是 `gitlab_ci.yml` 部分，在 build 階段要打包 Image，在 Runner 的環境中是沒有 docker 指令可以用的，所以這邊啟動一個 Docker Container 在裡面執行就可以用 docker 指令了。為了加速打包，我們都會先從 Container Registry 把上次的 Image 拉下來，放在下次要打包的 --cache-from 設定裡，其中 `$CI_COMMIT_SHA` `$CI_COMMIT_BEFORE_SHA` 這兩個都是 GitLab 預設環境變數，代表這次 commit 還有上次 commit 的 SHA 值。

```
build:
  stage: build
  image: "docker:stable"
  services:
    - docker:stable-dind
  script:
    - export CI_APPLICATION_REPOSITORY="$CI_REGISTRY/johnsonzhan/auto_example/master"
    - echo "Logging to GitLab Container Registry with CI credentials..."
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - docker image pull "$CI_APPLICATION_REPOSITORY:$CI_COMMIT_BEFORE_SHA" || \
    - docker build
      --cache-from "$CI_APPLICATION_REPOSITORY:$CI_COMMIT_BEFORE_SHA"
      --tag "$CI_APPLICATION_REPOSITORY:$CI_COMMIT_SHA" .
    - docker push "$CI_APPLICATION_REPOSITORY:$CI_COMMIT_SHA"
```

在一開始最讓我疑惑就是使用 docker 這段

```
image: "docker:stable"
services:
    - docker:stable-dind
```

> 為什麼需要兩個 Docker 的 Image ？他們有什麼不同？能只用一個嗎？

可以先來測試本地跑兩種 docker 的差別，不指定 entrypoint 時

```
docker run --privileged -it docker:stable
```
stable 就會用 sh 來啟動

```
/ #
```

而 dind 則是直接啟動 docker daemon，此外 dind 還會自動產生 TLS certificates

```
docker run --privileged -it docker:dind

```

```
Generating RSA private key, 4196 bit long modulus (2 primes)
....................................................................................................
...
```


> 為了在 Docker Container 內運行 Docker，會把 Host 上面的 Docker API 分享給 Container。通常這個 Docker API 是只有 Host 上面的 root 使用者才能用，所以如果 Container 有需要用 Docker API 就需要 TLS certificates 建立的安全連線。


- docker:stable 有執行 docker 需要的執行檔，他裡面也包含要啟動 docker 的程式(docker daemon)，但啟動 Container 的 entrypoint 是 sh

- docker:dind 繼承自 docker:stable，而且它 entrypoint 就是啟動 docker 的腳本，此外還會做完 TLS certificates


說到這好像只要用 `docker:stable-dind` 就能完成 Docker in Docker 的操作了？對！差不多，可以試著把 `gitlab_ci.yml` 從用 dind 當作 services 改成用它當 Image

```
build:
  stage: build
  image: "docker:stable"
  services:
      - docker:stable-dind
```

改成

```
build:
  stage: build
  image: "docker:stable-dind"
```

改成這樣後運行 CI 時你可能會看到這問題

```
error during connect: Get http://docker:2375/v1.40/containers/json: dial tcp: lookup docker on 169.254.169.254:53: no such host
```

於是用 `ps aux` 指令發現 `dind` 沒有啟動 docker daemon，所以只好手動啟用囉，改成這樣

```
build:
  image: "docker:dind"
  stage: build
  script:
    - docker-entrypoint.sh dockerd &
    - sleep 15
    ...
```

`docker-entrypoint.sh dockerd &` 就是啟動 docker daemon 並在背景執行，然後 sleep 等他確定跑完就有 docker 能用了吧？

```
error during connect: Get http://docker:2375/v1.40/containers/json: dial tcp: lookup docker on 169.254.169.254:53: no such host
```

一樣的錯誤訊息

```
$ ps aux
 PID   USER     TIME  COMMAND
     1 root      0:00 /bin/sh
    14 root      0:00 /bin/sh
    16 root      0:00 dockerd
    33 root      0:00 containerd --config /var/run/docker/containerd/containerd.toml --log-level info
   144 root      0:00 ps aux
```

而且這次確定有啟用 docker daemon，問題出在哪裡呢？剛剛說到 Container 要去連 Host 上的 Docker API 。但現在連線失敗卻是找 `http://docker:2375`，現在的 dind 已經不是被當做 services 來用了，而是要直接在裡面跑 Docker，所以他應該是要 `unix:///var/run/docker.sock` 用這種連線，於是把環境變數 `DOCKER_HOST` 從 `tcp://docker:2375` 改成空字串，讓 docker daemon 走預設連線就能成功囉！


## Deploy: 部署/更新 Cluster

如果只是要在 Container 內執行一些 script 可以用 build stage 裡面的做法，直接把要做的事寫在`gitlab_ci.yml` 的 script 裡就好。但是要客製化 Image 內容的時候，可能就需要自己本地打包 Image 然後上傳到 GitLab Container Registry 了，接下來就要自己打包名為 `registry.gitlab.com/johnsonzhan/auto_example/deploy` 的 Image 了。


> 接下來目標是讓客製化 Container 內可以用 `auto-deploy 參數` 指令的方式來操作。

```
production:
  image: "registry.gitlab.com/johnsonzhan/auto_example/deploy"
  stage: production
  script:
    - auto-deploy preparation
    - auto-deploy deploy
  environment:
    name: production
    url: http://$CI_PROJECT_PATH_SLUG.$KUBE_INGRESS_BASE_DOMAIN
```

Dockerfile 的部分會用 Gitlab 提供的 `helm-install-image` Image，在這環境下就可以直接使用 helm 指令。

`Dockerfile`

```
FROM "registry.gitlab.com/gitlab-org/cluster-integration/helm-install-image/releases/2.16.1-kube-1.13.12"

COPY src/ build/

# Install Dependencies
RUN apk add --no-cache openssl curl tar gzip bash

RUN ln -s /build/bin/* /usr/local/bin/
```

在這邊我們會建立 `src/bin/auto-deploy` 腳本，然後在用 `ln -s` 的方式把它連結到執行檔目錄下，就是因為這樣才能使用 `auto-deploy 參數` 這些指令，以下是我們的 `src/bin/auto-deploy`

以下的 script 提供了兩種執行方式：

- auto-deploy preparation
  - helm init 建立 helm 專案
  - 設定 tiller 在背景執行
  - 設定 cluster 的 namespace
- auto-deploy deploy
  - 使用 helm upgrade 部署 chart 到 K8S 上
  - 透過 --set 來設定要注入 template 的參數

```
#註解1
[[ "$TRACE" ]] && set -x

export RELEASE_NAME=${HELM_RELEASE_NAME:-$CI_ENVIRONMENT_SLUG}
export auto_database_url=postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@${RELEASE_NAME}-postgres:5432/${POSTGRES_DB}
export DATABASE_URL=${DATABASE_URL-$auto_database_url}
export TILLER_NAMESPACE=$KUBE_NAMESPACE
export HELM_HOST="localhost:44134"

function preparation() {
  local auto_chart="gitlab/auto-deploy-app"

  helm init

  #註解2
  helm repo add gitlab https://charts.gitlab.io

  #註解3
  helm fetch ${auto_chart} --untar

  #註解4
  kubectl get namespace "$KUBE_NAMESPACE" || kubectl create namespace "$KUBE_NAMESPACE"

  #註解5
  nohup tiller -listen ${HELM_HOST} >/dev/null 2>&1 &
  echo "Tiller is listening on ${HELM_HOST}"
}

function deploy() {
  local track="stable"
  local name="production"

  #註解6
  local image_repository=${CI_APPLICATION_REPOSITORY:-$CI_REGISTRY_IMAGE/$CI_COMMIT_REF_SLUG}
  local image_tag=${CI_APPLICATION_TAG:-$CI_COMMIT_SHA}
  local service_enabled="true"
  local postgres_enabled="$POSTGRES_ENABLED"
  local secret_name=''

  echo "Deploying new $track release..."
  helm upgrade --install \
    --wait \
    --set service.enabled="$service_enabled" \
    --set gitlab.app="$CI_PROJECT_PATH_SLUG" \
    --set gitlab.env="$CI_ENVIRONMENT_SLUG" \
    --set gitlab.envName="$CI_ENVIRONMENT_NAME" \
    --set gitlab.envURL="$CI_ENVIRONMENT_URL" \
    --set releaseOverride="$RELEASE_NAME" \
    --set image.repository="$image_repository" \
    --set image.tag="$image_tag" \
    --set image.pullPolicy=IfNotPresent \
    --set image.secrets[0].name="$secret_name" \
    --set application.track="$track" \
    --set application.database_url="$DATABASE_URL" \
    --set service.commonName="le-$CI_PROJECT_ID.$KUBE_INGRESS_BASE_DOMAIN" \
    --set service.url="$CI_ENVIRONMENT_URL" \
    --set replicaCount=1 \
    --set postgresql.enabled="$postgres_enabled" \
    --set postgresql.nameOverride="postgres" \
    --set postgresql.postgresDatabase="$POSTGRES_DB" \
    --set postgresql.imageTag="$POSTGRES_VERSION" \
    --set application.migrateCommand="$DB_MIGRATE" \
    --namespace="$KUBE_NAMESPACE" \
    "$name" \
    auto-deploy-app/
}

## End Helper functions

option=$1
case $option in
  preparation) preparation ;;
  deploy) deploy ;;
  *) exit 1 ;;
esac

```

```
[[ "$TRACE" ]] && set -x
```

> 註解1：
> 如果有環境變數 TRACE 就 set -x，這樣就能在執行前，顯示指令內容。

```
helm repo add gitlab https://charts.gitlab.io
```

> 註解2：
> 註冊一個 Chart Repository
> 可以用 helm repo list 看看現在有註冊哪些 Chart Repository


```
helm fetch ${auto_chart} --untar
```

> 註解3：
> 把 repo 裡的 helm 專案抓下來，好奇裡面長什麼樣子的話，也可以在本機用這行把他抓下來
> `helm fetch gitlab/auto-deploy-app --untar`

```
kubectl get namespace "$KUBE_NAMESPACE" || kubectl create namespace "$KUBE_NAMESPACE"
```
> 註解4：
> 新增一個 namespace 之後部署的 resource object 都整理進去

```
nohup tiller -listen ${HELM_HOST} >/dev/null 2>&1 &
```
> 註解5：
> nohup 可以讓你在離線或登出系統後，還能夠讓工作繼續進行
> \>/dev/null 2>&1 是把正確輸出還有錯誤輸出都忽略掉
> 最後的 & 是讓工作背景執行


```
local image_repository=${CI_APPLICATION_REPOSITORY:-$CI_REGISTRY_IMAGE/$CI_COMMIT_REF_SLUG}
```
> 註解6
> 在不特別設定 `CI_APPLICATION_REPOSITORY` 的情況下，`image_repository` 的值就是預設環境變數 `CI_REGISTRY_IMAGE/CI_COMMIT_REF_SLUG`，實際上 Auto Devops 的 template 也大量使用這寫法。

> 先解釋這 :- 符號

> A:-B 的意思是如果有 A 就用它，沒有就用 B

> `CI_REGISTRY_IMAGE` 跟 `CI_COMMIT_REF_SLUG` 都是 GitLab 預設環境變數，在我的範例中

> `CI_REGISTRY_IMAGE` 就是 `registry.gitlab.com/johnsonzhan/auto_example`

> `CI_COMMIT_REF_SLUG` 就是 master



完成 `Dockerfile` 還有 `src/bin/auto-deploy` 後，就能打包 Image 然後上傳囉！

```
docker build . -t registry.gitlab.com/johnsonzhan/auto_example/deploy
docker push registry.gitlab.com/johnsonzhan/auto_example/deploy
```

## 心得

這次練習除了認識了 K8S 跟 GitLab 的一些設定外，也透過看 Auto Devops 的 `gitlab_ci.yml` 裡的 template 認識了很多 Shell Scripts 的寫法，算是收穫很多！對我來說研究 Auto Devops 難度最高的地方就是太多工具整合在一起，搞不清楚他們之間的關係，出錯也不知道從何查起，尤其最一開始還挑戰用 RKE + rancher 來建 K8S Cluster，真的是從頭到尾都卡關，幸好過程中同事[蒼時弦也](https://blog.frost.tw/)在卡關的時候幫我解惑，不然應該會研究到放棄XD


## 參考資料


[GitLab Auto DevOps](https://docs.gitlab.com/ee/topics/autodevops/)

[和艦長一起 30 天玩轉 GitLab ](https://ithelp.ithome.com.tw/users/20120986/ironman/2733)

[Kubernetes 基礎教學（一）原理介紹](https://medium.com/@C.W.Hu/kubernetes-basic-concept-tutorial-e033e3504ec0)

[Tips on Moving your Dev Env from Docker Compose to Kubernetes](https://blog.tilt.dev/2019/09/16/tips-on-moving-your-dev-env-from-docker-compose-to-kubernetes.html)

[[Kubernetes] Package Manager - Helm 簡介](https://godleon.github.io/blog/Kubernetes/k8s-Helm-Introduction/)

[鳥哥：認識與學習BASH](http://linux.vbird.org/linux_basic/0320bash.php)、[學習 Shell Scripts](http://linux.vbird.org/linux_basic/0340bashshell-scripts.php)
