---
title: 使用 Fastlane 在 GitLab 上搭建 Pipeline
date: 2022-05-26 20:00:00 +0800
categories: [iOS]
tags: [pipeline]     # TAG names should always be lowercase
image:
  src: /assets/img/post/pipeline/header.png
---

最近在 GitLab 上面给一个新的工程搭建了一条 Pipeline，所做的事情就是每次有代码提交，都会自动运行测试、安全检查然后打包上传到 TestFlight 以供测试。本以为基本上跟着文档做就会很顺利，不过这过程中还是踩了一些坑的，所以这边记录下来整个过程，希望可以形成一个完整的操作手册，只要一步步来基本就能完成搭建。

## 1. 给自己的工程配置 Fastlane

使用 [bundler](https://bundler.io/) 来安装 Fastlane

```
$ gem install bundler
```

在根目录下创建一个 `Gemfile` 文件，并将下面内容放进去：

```
source "https://rubygems.org"

gem "fastlane"
```

然后执行

```
$ bundle install
$ git add Gemfile Gemfile.lock
```

Fastlane 装好之后就该配置了，执行初始化之后选择手动配置，然后我们添加一个执行测试的自定义 lane.

```
$ bundle exec fastlane init
```

```ruby
// Fastfile

default_platform(:ios)

platform :ios do
  desc "Run tests"
  lane :tests do
    run_tests(
      scheme: "PipelineDemo",
      derived_data_path: "~/Library/Developer/Xcode/DerivedData",
    )
  end
end
```



## 2. 添加 GitLab pipeline 配置

GitLab 的 pipeline 配置是通过 `.gitlab-ci.yml` 文件来完成的，先在工程根目录创建一个配置文件并添加一个 `unit_tests` 的 stage。

```yaml
stages:
  - unit_tests

variables:
  LC_ALL: "en_US.UTF-8"
  LANG: "en_US.UTF-8"

before_script:
  - gem install bundler
  - bundle install

unit_tests:
  stage: unit_tests
  script:
    - bundle exec fastlane tests
  tags:
    - ios

```

有了配置之后，还需要有 runner 来运行。

> GitLab Runner is an application that works with GitLab CI/CD to run jobs in a pipeline.

这里有两个选项：

- 可以把 Runner 安装在自己的 macOS 上，等于是把自己的电脑作为服务器来跑 pileline。
- 使用 GItLab 提供的 shared runner，由 GitLab 管理，超过使用时长是要收费的。

如果是个人开发者或者是小团队的话，可以使用第一个选项，配置简单也不需要付费，我个人使用过这种方式，基本按照[官方文档](https://docs.gitlab.com/runner/install/osx.html) 配置就可以了，这里要注意的是要把 shell 换成 Bash 才可以，其他没有什么坑。具体安装完之后的配置可以看[这里](https://docs.gitlab.com/runner/configuration/macos_setup.html)。

我最近做的配置是使用了 GitLab SaaS runner，下面针对这个方式的配置来进行说明。

在 `.gitlab-ci.yml` 中添加对 runner 的指定：

```yaml
.macos_saas_runners:
  tags:
    - shared-macos-amd64
  image: macos-12-xcode-13

unit_tests:
  extends:
    - .macos_saas_runners
  stage: unit_tests
  ...
```

完成这些，pipeline 应该就可以先工作起来了。

## 3. 使用 match 来管理苹果开发者证书

要打包上传到 TestFlight 一定绕不过证书签名这个环节。使用 fastlane match 能够使证书管理更容易。

首先创建一个新的 private 的 repo 来存放 certificates 和 profiles。

然后在项目根目录下执行 `bundle exec fastlane match init`，会生成一个 `Matchfile` 的配置文件，根据需求更改一下 branch 和 type。

```
git_url("https://github.com/zhiying-fan/ios-certificates.git")
git_branch("main")
storage_mode("git")

type("appstore") # The default type, can be: appstore, adhoc, enterprise or development

app_identifier("tools.fastlane.app")
username("user@fastlane.tools") # Your Apple Developer Portal username
```

然后执行生成证书并上传的命令，这里需要一个有权限创建 destribution 类型证书的开发者账号，权限是要 App Manager/Admin。

```
$ bundle exec fastlane match appstore
```

这个过程中会要求输入自己电脑 keychain 的密码，还会要求输入一个 **passphrase**，这个很重要一定要记住，是用来加密存放 certificates 的 repo 的，后续解密需要用到它，需要把它放在 CI 的环境变量中。

![](/assets/img/post/pipeline/match-password.png)

执行成功之后，会自动生成 destribution 类型的 certificates 和 profiles，然后上传到 private repo。

## 4. 在 GitLab 配置 ssh key

根据 match 的文档，可以使用 ssh key 来让 pipeline 有权限获取到我们 private repo 的证书。做法是把私钥作为 CI 的环境变量，公钥作为证书 repo 的 deploy key，这个过程类似于我们平时使用 ssh 来和 github 上面的 repo 进行通信。

一般我们的电脑上都会有 ssh key，会用来和 github 等的 repo 进行通信。所以这里在一个不同的文件夹下新建一对 ssh key，专门给 CI 用。

```
ssh-keygen -t ed25519 -C "<comment>"
```

```
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/user/.ssh/id_ed25519): example/.ssh/id_ed25519
```

下一步不要给这个 ssh key 设置 passphrase，因为 CI 上面是不能进行交互的，会带来一些麻烦，这个在 GitLab 的文档中也有提到。

有了 ssh key 之后，分别把公钥配置在证书 repo 的 deploy key 中，私钥配置在 CI 的环境变量。

### 配置公钥

![](/assets/img/post/pipeline/deploy-key.png)

### 配置私钥

![](/assets/img/post/pipeline/private-key.png)

### 配置 known hosts

这一步文档上说的是 good practice，但是其实是必须要做的，不然 pipeline 会卡在 git clone 那一步，也就是不能实质性地访问到证书 repo。

因为我们的证书 repo 是放在 gitlab 的，因此执行 `ssh-keyscan gitlab.com` 来获取 known hosts 的 Value。然后把值放入 CI 环境变量。

![](/assets/img/post/pipeline/known-hosts.png)

### 在 pipeline 中 setup ssh

在 sync_code_signing 获取证书之前，需要在 runner 的环境中配置好 ssh，需要执行以下 script 里面的命令。

```yaml
test_flight_build:
  stage: test_flight
  script:
    - eval $(ssh-agent -s)
    - echo "${SSH_PRIVATE_KEY}" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - echo "${SSH_KNOWN_HOSTS}" >> ~/.ssh/known_hosts
    - chmod 644 ~/.ssh/known_hosts
    # - fastlane beta
  tags:
    - ios
  only:
     - /^release-.*$/
     - main
```

## 5. 在 pipeline 中获取证书

ssh 配置好之后，使用 fastlane 的 `sync_code_signing` Action 就能够获取到证书。但是这里在 CI runner 中碰到一个问题，就是默认的 keychain 不能访问使用，因此在此之前需要多加一步 `setup_ci` ，这个 Action 主要是创建了一个临时的 keychain 以供我们存放证书使用。

```ruby
// Fastfile

desc "Submit a new Beta Build to Apple TestFlight"
lane :beta do
  setup_ci
  sync_code_signing(readonly: true)
end
```

## 6. Build App

有了证书之后就能 Build App 生成 ipa 包了。这里需要注意的是需要修改一下 Xcode 的配置，将 Release 的签名方式改为手动，明确指定 Profile 为我们用 match 生成的 Profile，不然在 pipeline 会报错找不到对应的 profile。

![](/assets/img/post/pipeline/xcode-manual.png)

添加 build app 的 Action：

```ruby
// Fastfile

desc "Submit a new Beta Build to Apple TestFlight"
lane :beta do
  setup_ci
  sync_code_signing(readonly: true)
  build_app(
    scheme: "PipelineDemo",
    export_method: "app-store",
  )
end
```

## 7. 生成并配置 App Store Connect API Key

Build 好之后就可以上传了，这里涉及到苹果服务的 Authentication，即我们需要能够访问到我们自己的苹果开发者账户然后进行上传。Fastlane 提供了四种方式，分别如下：

- Method 1: App Store Connect API key (recommended)
- Method 2: Two-step or two-factor authentication
- Method 3: Application-specific passwords
- Method 4: Apple ID without 2FA (deprecated)

由于现在苹果的 2FA 是强制打开的，所以方式 4 不可行。

方式 2 要求提前在上传的机器上获取一个有效的 session，并提供给 Fastlane，由于我们使用的是 SaaS 服务并且 session 很快就会过期，因此在 CI 上其实也是行不通的。

方式 3 经过试验，还是会提示需要 2FA 的验证码，所以不行。

最后只剩下方式 1 可行，这个也是推荐的方式。

### 生成 API Key

到 [App Store Connect Keys](https://appstoreconnect.apple.com/access/api) 的界面，这一步需要有 Admin 权限的账号，并且需要 Account Holder 授权才行。被授权之前长这样，如果看不到 Keys 这个 Tab，说明你不是 Admin。

![](/assets/img/post/pipeline/api-key-without-access.png)

有了权限之后，就可以生成 Key 了。

![](/assets/img/post/pipeline/api-key-with-access.png)

生成之后，记得把下载下来的 API Key 保存好，因为只能下载一次。

### 配置 API Key

API Key 的这些信息是可以配置在 CI 的环境变量的，具体怎么知道 Fastlane Action 的哪些参数可以从环境变量获取，可以通过执行 `fastlane action action_name` 来获取。例如接下来要用到的命令：

![](/assets/img/post/pipeline/api-key-action.png)

`app_store_connect_api_key` 这个 Action 可以生成一个 `APP_STORE_CONNECT_API_KEY` 以供其他的 Action 使用。从上图中可以看到，它需要的 `key_id` `issuer_id` `key_content` 这些关键信息，都可以从环境变量读取，具体的环境变量的 Key 在后面也有提供。

注意这里使用 key_content 而不是 key_filepath 是因为在 CI 上设置值是可行的，目前没有找到在 GitLab 的环境变量中怎么配置文件。因此需要把下载下来的那个 `.p8` 的 Key 文件转换成 base64 的文本。

```
cat {KEYNAME} | base64
```

然后把这个作为值放在环境变量中，并且把 `is_key_content_base64` 设置为 `true` 。

```ruby
// Fastfile

  desc "Submit a new Beta Build to Apple TestFlight"
  lane :beta do
    setup_ci
    sync_code_signing(readonly: true)
    build_app(
      scheme: "PipelineDemo",
      export_method: "app-store",
    )
    app_store_connect_api_key(
      is_key_content_base64: true,
      duration: 1200,
    )
  end
```

## 8. 上传到 TestFlight

终于到了最后一步，可以上传了，准备工作都做好了之后，只需要在最后加上 `upload_to_testflight` 就可以了。

## 完整的配置总结

### .gitlab-ci.yml

```yaml
.macos_saas_runners:
  tags:
    - shared-macos-amd64
  image: macos-12-xcode-13

stages:
  - unit_tests
  - test_flight

variables:
  LC_ALL: "en_US.UTF-8"
  LANG: "en_US.UTF-8"

before_script:
  - gem install bundler
  - bundle install

unit_tests:
  extends:
    - .macos_saas_runners
  stage: unit_tests
  script:
    - bundle exec fastlane tests
  tags:
    - ios

test_flight_build:
  stage: test_flight
  script:
    - eval $(ssh-agent -s)
    - echo "${SSH_PRIVATE_KEY}" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - echo "${SSH_KNOWN_HOSTS}" >> ~/.ssh/known_hosts
    - chmod 644 ~/.ssh/known_hosts
    - bundle exec fastlane beta
  tags:
    - ios
  only:
     - /^release-.*$/
     - main
```

### Fastfile

```ruby
default_platform(:ios)

platform :ios do
  desc "Run tests"
  lane :tests do
    run_tests(
      scheme: "PipelineDemo",
      derived_data_path: "~/Library/Developer/Xcode/DerivedData",
    )
  end

  desc "Submit a new Beta Build to Apple TestFlight"
  lane :beta do
    setup_ci
    sync_code_signing(readonly: true)
    build_app(
      scheme: "PipelineDemo",
      export_method: "app-store",
    )
    app_store_connect_api_key(
      is_key_content_base64: true,
      duration: 1200,
    )
    upload_to_testflight
  end
end
```

### CI 环境变量

```
MATCH_PASSWORD: ******
SSH_PRIVATE_KEY: ******
SSH_KNOWN_HOSTS: ******
APP_STORE_CONNECT_API_KEY_KEY_ID: ******
APP_STORE_CONNECT_API_KEY_ISSUER_ID: ******
APP_STORE_CONNECT_API_KEY_KEY: ******
```



[![](/assets/img/post/download-demo.png){: width="200"}](https://github.com/zhiying-fan/Demo-GitLabPipeline.git)

## 参考

- [Fastlane Setup](http://docs.fastlane.tools/getting-started/ios/setup/)
- [Fastlane GitLab CI Integration](https://docs.fastlane.tools/best-practices/continuous-integration/gitlab/)
- [GitLab SaaS runners on macOS](https://docs.gitlab.com/ee/ci/runners/saas/macos_saas_runner.html)
- [Fastlane match](https://docs.fastlane.tools/actions/match/)
- [Generate an SSH key pair](https://docs.gitlab.com/ee/user/ssh.html#generate-an-ssh-key-pair)
- [Create a project deploy key](https://docs.gitlab.com/ee/user/project/deploy_keys/index.html#create-a-project-deploy-key)
- [Using SSH keys with GitLab CI/CD](https://docs.gitlab.com/ee/ci/ssh_keys/)
- [iOS Beta deployment using *fastlane*](http://docs.fastlane.tools/getting-started/ios/beta-deployment/)
- [Fastlane setup_ci](http://docs.fastlane.tools/actions/setup_ci/#setup_ci)
- [Authenticating with Apple services](http://docs.fastlane.tools/getting-started/ios/authentication/)
- [App Store Connect API](http://docs.fastlane.tools/app-store-connect-api/)
