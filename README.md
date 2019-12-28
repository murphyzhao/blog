
<p align="left">
    <a href="https://travis-ci.com/murphyzhao/blog" alt="Travis-CI"><img src="https://api.travis-ci.com/murphyzhao/blog.svg?branch=master" /></a>
</p>

## 概述

本博客托管与 GitHub Pages，基于 [fluid hexo 主题](https://github.com/fluid-dev/hexo-theme-fluid)。

本仓库使用 Travis-CI 自动构建，将 master 分支中的内容自动部署到 gh-pages 分支，详细的部署脚本请查看 [.travis.yml 文件](./.travis.yml)。

## 借用

如果你想基于我的仓库构建您自己的博客，这是非常快速的方式，你只需要如下步骤即可，不需要本地 Hexo 和 Nodejs 环境：

- fork 我的仓库到你的 github
- 克隆你 fork 的仓库到你的本地
- 修改以下配置文件

    - _config.yml 文件

        根据该文件中的注释，进行定制化修改，包括博客的名字信息。

    - .travis.yml

        修改该文件底部的自定义域名。

    - source/_data/fluid_config.yml

        这里修改您页面使用到的头部图片资源，各种功能使能与配置，尤其注意替换您自己的 `web_analytics` 网页统计 key。

- 自定义图片资源

    自定义图片资源存储在 `source/site-img` 目录下。

完成以上内容后，提交并推送变动到您的 github 仓库，以及配置 Travis-CI。

- 使用 github 账户登录 [Travis](https://travis-ci.com)
- 在 Travis 中选择构建你的博客仓库
- 在 Travis 中设置环境变量 `GITHUB_TOKEN`（GITHUB_TOKEN 的值在 github 设置中增加）
- 在 github 用户设置中，`Settings/Developer settings` 选择增加 `Personal access tokens`
- 博客仓库中设置中选择启用 github pages
- 博客仓库中设置使用 gh-pages 分支，并选择使用 HTTPS

更多详细的使用说明，请参考本博客使用的主题 [fluid](https://github.com/fluid-dev/hexo-theme-fluid)。

## 发布新文章

在 `source/_posts` 目录中增加文章。

增加文章并提交到 github master 分支后，Travis 会自动部署，等待几分钟即可在你的博客地址中看到新增的文章。

## 参考

博客搭建过程中主要参考了一下博文或个人网页，感谢分享。

- [Fluid 使用指南](https://fluid-dev.github.io/hexo-fluid-docs/guide/)
- [edsion 的博客](https://blog.i1hao.com/2018/09/01/hexo-and-githubpages-best-practices/)
