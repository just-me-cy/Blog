---
title: 私有npm部署
date: 2016-09-15 10:25:38
tags: ['npm','sinopia']
categories: ['npm']
---
## 目标
  通过sinopia部署私有npm，私有包托管在内部服务器，供内部成员使用；公有模块仍走外部npm下载，下载后会缓存到内部服务器。

## 部署环境
```javascript
uname -a
  Linux Centos6.4-template 2.6.32-573.18.1.el6.x86_64
npm -v
  3.3.1
node -v
  6.2.0
```

## 安装
### ssh登录linux服务器
### 新目录并安装
```javascript
cd ~
mkdir sinopia && cd sinopia
npm i sinopia
```
  编译失败
```javascript
gyp ERR! build error 
gyp ERR! stack Error: `make` failed with exit code: 2
gyp ERR! stack     at ChildProcess.onExit (/opt/soft/node/lib/node_modules/npm/node_modules/node-gyp/lib/build.js:276:23)
gyp ERR! stack     at emitTwo (events.js:106:13)
gyp ERR! stack     at ChildProcess.emit (events.js:191:7)
gyp ERR! stack     at Process.ChildProcess._handle.onexit (internal/child_process.js:204:12)
gyp ERR! System Linux 2.6.32-573.18.1.el6.x86_64
gyp ERR! command "/opt/soft/node/bin/node" "/opt/soft/node/lib/node_modules/npm/node_modules/node-gyp/bin/node-gyp.js" "configure" "build"
gyp ERR! cwd /opt/soft/node/lib/node_modules/sinopia/node_modules/fs-ext
gyp ERR! node -v v6.2.0
gyp ERR! node-gyp -v v3.3.1
gyp ERR! not ok 
```
  查看[`Issue`](https://github.com/rlidwka/sinopia/issues/311)，发现**有各种安装问题，而且作者已经1年多没有维护了**。
  使用指令
```javascritp
npm install sinopia --no-optional --no-shrinkwrap
```
  虽然编译无问题，但无法从npm中拉取开源包
  然后查看[travis CI](https://travis-ci.org/rlidwka/sinopia)，决定换个node版本试试。

  ![travis CI](../../../../images/npm-160915.png)

### 使用低版本node
  在server上安装node多版本管理工具[nvm](https://github.com/creationix/nvm)
```javascript
curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.31.7/install.sh | bash

// 安装低版本node
nvm install 0.12

// 重新安装
npm i sinopia
```
  编译成功了！
  nvm相关命令
```javascript
nvm ls-remote # 列出远程服务器上所有的可用版本
nvm ls  # 查看已安装版本
nvm use 4.2.2 # 使用版本
```

### 启动
```javascript
sinopia
```
  启动日志
```javascript
warn  --- config file  - /当前目录/sinopia/config.yaml
warn  --- http address - http://localhost:4873/
```
  启动后会在当前目录下生成sinopia文件夹，里面有config.yaml和storage，添加用户后会自动创建htpasswd。
  config.yaml是配置文件，storage是来用缓存从上游npm拉取的开源包，同时也保存内部发布的私有包，htpasswd是用来保存用户账号密码信息。

### 配置编辑
```javascrpt
# path to a directory with all packages
storage: ./storage #仓库保存的路径

#用户验证相关
auth:
  htpasswd:
    file: ./htpasswd
    # Maximum amount of users allowed to register, defaults to "+inf".
    # You can set this to -1 to disable registration.
    max_users: 1000 #如果设置为-1表示禁用 npm adduser 命令来创建用户

# a list of other known repositories we can talk to
# 配置上游的npm服务器，若请求的资源不存在，去上游服务器拉取
uplinks:
  npmjs:
    url: https://registry.npmjs.org/ #如果网速不好可以配置成淘宝镜像 http://registry.npm.taobao.org/

# 配置资源的发布、下载权限
packages:
  '@*/*': #对私有模块的配置
    # scoped packages
    access: $all #私有包安装权限配置
    publish: $authenticated # 私有包发布权限配置

  '*': # 仅有模块的访问配置
    # allow all users (including non-authenticated users) to read and
    # publish all packages
    #
    # you can specify usernames/groupnames (depending on your auth plugin)
    # and three keywords: "$all", "$anonymous", "$authenticated"
    access: $all

    # allow all known users to publish packages
    # (anyone can register by default, remember?)
    publish: $authenticated

    # if package is not available locally, proxy requests to 'npmjs' registry
    proxy: npmjs # 如果server上的storage没有缓存，就去外部拉取

# log settings
logs:
  - {type: stdout, format: pretty, level: http}
  #- {type: file, path: sinopia.log, level: info}
# 配置监听端口与主机名
listen:
 - 0.0.0.0:4873
# 公司内部使用代理访问外网
http_proxy:
 - http://用户名:密码@代理ip:8080
```
  完整配置：https://github.com/rlidwka/sinopia/blob/master/conf/full.yaml
  保存后重新启动，使用[`pm2`](https://wohugb.gitbooks.io/pm2/content/index.html)启动
  安装`pm2`: `npm install -g pm2`
  启动：
```javascript
pm2 start `which sinopia`
```
  ![pm2启动](../../../../images/npm-pm2-160915.jpg)

### 创建用户
  设置`npm`镜像地址
```
npm set registry http://server的ip:4873/
```
  也可以手动修改`.npmrc`文件 `registry = http://server的ip:4873/`

  添加用户
  `npm adduser`
  按照命令行中的提示，依次输入`Username`、`Passworld`、`Email`完成用户的创建

### 客户端使用
  浏览器查看 http://server的ip:4873/
  看到以下界面
  ![](../../../../images/npm-browser-160915.jpg)
  
  客户端配置`registry`源为服务器
```
npm set registry http://server的ip:4873/
```
  如果需要代理访问外网，客户端也不需要设置proxy，因为服务器已经设置了。
  可以使用nrm管理多镜像
```javascript
npm install -g nrm
nrm add sinopia http://serverIP:4873 # 添加npm镜像地址
nrm use sinopia # 使用
nrm ls #查看所有可用镜像
```
  安装模块
```
mkdir test && cd test
npm install react # 第一次拉取本地没有react包，会从npmjs指定的地址下载
rm -rf node-modules # 删除下载的模块
npm insatll react # 第二次安装，会从server缓存下载了，速度很快
```
  发布私有包
  在发布模块前，需要先登录 `npm login`
  私有包命名`@scope/package`
  `npm publish`
  `npm`包的版本号一般都是`x.y.z`的形式
  `x`表示主版本号，通常有重大改变或者达到里程碑才改变
  `y`表示次版本号,在保证主体功能基本不变的情况下，如果适当增加了新功能可以更新此版本号
  `z`表示补丁号，一些小范围的修修补补就可以更新补丁号。
  第一版本通常是`0.0.1`或者`1.0.0`，当修改了代码，需要更新版本号重新发布到`npm`
```javascript
npm version patch => z+1
npm version minor => y+1 && z=0
npm version major => x+1 && y=0 && z=0
```

  更新补丁再发布
```javascript
npm version patch
npm publish

// 移除已发布
npm unpublish
```
  发布私有包后发现浏览器多出了刚发布的私有包列表
![](../../../../images/npm-browser2-160915.jpg)

