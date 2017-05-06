## 使用 Github 来管理 Hexo 博客源代码，并使用 Travis 来自动部署

[![Build Status](https://travis-ci.org/cnu4/cnu4.github.io.svg?branch=source)](https://travis-ci.org/cnu4/cnu4.github.io)

### 管理以及定制 Hexo 主题

将主题 Fork 到自己的账号下后，在博客源代码根目录使用 `git submodule` 将主题的仓库作为子仓库

`git submodule add git@github.com:cnu4/hexo-theme-apollo.git themes/apollo`

进入主题目录，修改主题后按正常流程 `git commit`, `git push` 更新主题的 git 仓库，再到博客根目录 `commit` 源代码

### 持续集成

要求：持续部署到 Github 和 Coding

#### 配置

将 Hexo 源代码 `push` 到 Github 的仓库，我是将博客的仓库的 `source` 分支作为源代码仓库

登录 `travis` 并使用 github 登录后开启上述源代码的仓库

安装 `travis cli` 工具后登录，使用一下命令生成私钥的加密文件，以下是 Mac 环境下的命令

`gem install travis`

`travis login`

`travis encrypt-file ~/.ssh/id_rsa --add -r /cnu4/cnu4.github.io`

以上命令会自动在 `.travis.yml` 文件的 `before_install` 中添加解密得到私钥的代码，并生成文件 `id_rsa.enc`

创建 `.travis` 文件夹，在里面新建 `ssh_config` 文件，填入一下内容，避免第一次连接时的询问

```
Host *
StrictHostKeyChecking no
IdentityFile ~/.ssh/id_rsa
IdentitiesOnly yes
```
将 `id_rsa.enc` 文件移入 `.travis` 文件夹

下面是 `.travis.yml` 配置文件

``` yml
language: node_js
node_js: 6.10.1
install:
- npm install
- npm install hexo-cli -g
script:
- hexo g
- hexo d
- rm -rf ~/.ssh
branches:
  only:
  - source
git:
  submodules: false
before_install:
- openssl aes-256-cbc -K $encrypted_5b8902d6a626_key -iv $encrypted_5b8902d6a626_iv
  -in .travis/id_rsa.enc -out ~/.ssh/id_rsa -d
- chmod 600 ~/.ssh/id_rsa
- sudo cp .travis/ssh_config /etc/ssh/ssh_config
- sudo chmod 755 /etc/ssh/ssh_config
- eval "$(ssh-agent -s)"
- ssh-add ~/.ssh/id_rsa
- git config --global user.name "cnu4"
- git config --global user.email "fangxw1004@qq.com"
- git submodule update --init --recursive

```

在 `before_install` 中先解密得到私钥并放到 `.ssh` 目录下，接下来的 `npm install` 安装 `hexo`，然后就是构建和部署命令了

配置中需要注意：

 - 配置中通过 `git: submodules: false` 使得 travis 不用 `https` 的方式处理 git 子模块，否则此时会出现克隆时身份验证不通过的情况

 - 通过将代码 `git submodule update --init --recursive` 添加在 `ssh-add ~/.ssh/id_rsa` 之后，使用 `ssh` 处理子模块

 - 注意加密的 ssh 私钥不能有密码保护，也就是创建私钥的时候不要输入密码

接下来写文章后只需要一下命令便可以完成自动部署到 Github 和 Coding

``` bash
git add .
git commit -m "commit message"
git push origin source # 我的源文件仓库是 source 分支
```

#### 新环境

在换了新电脑后，使用以下流程

``` bash
git clone repo_of_source.git # 克隆源码
git submodule init # 初始化主题
git submodule update
hexo new
git add .
git commit
git push origin source # 推送到 source 分支触发自动部署
```