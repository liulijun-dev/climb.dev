---
title: 利用Hexo + Travis CI在Github上搭建个人博客
date: 2020-02-08 17:19:49
tags: hexo, github, theme
author: 刘利军
---
在本教程中，我们将会使用 Travis CI 将 Hexo 博客部署到 GitHub Pages 上，并配置Next主题。

# 安装Hexo
在本地依次执行如下命令，
```shell
npm install hexo-cli -g
hexo init blog
cd blog
npm install
hexo server
```
此时在本地浏览器上访问http://localhost:4000 可以看到使用默认主题的博客主页。

# 将 Hexo 部署到 GitHub Pages

- 新建一个 repository，并将 repository 命名为 <你的 GitHub 用户名>.github.io
- 将 [Travis CI](https://github.com/marketplace/travis-ci "Travis CI") 添加到你的 GitHub 账户中
- 前往 GitHub 的 [Applications settings](https://github.com/settings/installations "Applications settings")，配置 Travis CI 权限，使其能够访问你的 repository
- 前往 GitHub [Personal Access Token](https://github.com/settings/tokens "Personal Access Token")，只勾选 repo 的权限并生成一个新的 Token。Token 生成后请复制并保存好
- 使用github账号登录[Travis CI](https://travis-ci.com/ "Travis CI")，前往你的 repository 的设置页面，在 Environment Variables 下新建一个环境变量，Name 为 GH_TOKEN，Value 为刚才你在 GitHub 生成的 Token。确保 DISPLAY VALUE IN BUILD LOG 保持 不被勾选 避免你的 Token 泄漏。点击 Add 保存
- 在上一节新建的 Hexo 站点文件夹中新建一个 .travis.yml 文件，内容如下：
```yaml
sudo: false
language: node_js
node_js:
  - 10 # use nodejs v10 LTS
cache: npm
branches:
  only:
    - development # build development branch only
script:
  - hexo generate # generate static files
deploy:
  provider: pages
  skip-cleanup: true
  github-token: $GH_TOKEN
  keep-history: true
  target-branch: master #部署到master分支
  on:
    branch: development
  local-dir: public
```
- 将上一节生成的文件推送到 repository 中的development分支。
# 设置Theme Next
- 进入到Hexo 站点根目录，执行如下命令
`git clone https://github.com/theme-next/hexo-theme-next themes/next`
- 配置根目录下的_config.yml，配置结果如下：
```shell
# Extensions
## Plugins: https://hexo.io/plugins/
## Themes: https://hexo.io/themes/
theme: next
```
- 进入到themes/next，执行如下命令将next主题切换到v7.7.0，否则在编译时会出现"TypeError: Cannot read property 'path' of undefined"错误：
```shell
git check tag/v7.7.0
```
-重新编译hexo工程，并访问http://localhost:4000 可以看到应用新样式后的首页
# 配置博客
- 请参考[配置](https://hexo.io/zh-cn/docs/configuration.html "配置")配置博客的title, subtitle, description等
- 请参考[hero-theme-next](https://github.com/theme-next/hexo-theme-next "hero-theme-next")Theme Next主题
