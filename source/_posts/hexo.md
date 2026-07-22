---
title: Hexo & github 构建个人博客
date: 2024-01-07
tags: Hexo & github 构建个人博客
---
# 安装nodejs 环境
官网 http://nodejs.p2hp.com/

# 使用 npm 一键安装 Hexo 博客程序：

```bash
npm install -g hexo-cli
```
# Hexo 初始化和本地预览
初始化并安装所需组件：
```
hexo init      # 初始化
npm install    # 安装组件
```
完成后依次输入下面命令，启动本地服务器进行预览：
```
hexo g   # 生成页面
hexo s   # 启动预览
```
访问 http://localhost:4000 出现 Hexo 默认页面，本地博客安装成功！

部署 Hexo 到 GitHub Pages
本地博客测试成功后，就是上传到 GitHub 进行部署，使其能够在网络上访问。

首先安装 hexo-deployer-git：
```
npm install hexo-deployer-git --save
```
然后修改 _config.yml 文件末尾的 Deployment 部分，修改成如下：
```
deploy:
  type: git
  repository: git@github.com:用户名/用户名.github.io.git
  branch: main
```
完成后运行 hexo d 将网站上传部署到 GitHub Pages。

完成！这时访问我们的 GitHub 域名 https://用户名.github.io 就可以看到 Hexo 网站了。

#  开始使用 发布文章
进入博客所在目录，右键打开 Git Bash Here，创建博文：

- hexo new "My New Post"
- 也可以直接在_posts 目录下新建md文件
写完后运行下面代码将文章渲染并部署到 GitHub Pages 上完成发布。以后每次发布文章都是这两条命令。
```
hexo g   # 生成页面
hexo d   # 部署发布
```
也可以不使用命令自己创建 .md 文件，只需在文件开头手动加入如下格式 Front-matter 即可，写完后运行 hexo g 和 hexo d 发布。

# 网站设置
包括网站名称、描述、作者、链接样式等，全部在网站目录下的 _config.yml 文件中，参考官方文档按需要编辑。

注意：冒号后要加一个空格！

# 更换主题
在 Themes | Hexo 选择一个喜欢的主题，比如 NexT，进入网站目录打开 Git Bash Here 下载主题：
```
git clone https://github.com/LenChou95/hexo-theme-ZenMind.git themes/ZenMind
```
然后修改 _config.yml 中的 theme 为新主题名称 ZenMind _config.yml 替换为主题自带的，参考主题说明。）

# 常用命令
```
hexo new "name"       # 新建文章
hexo new page "name"  # 新建页面
hexo g                # 生成页面
hexo d                # 部署
hexo g -d             # 生成页面并部署
hexo s                # 本地预览
hexo clean            # 清除缓存和已生成的静态文件
hexo help             # 帮助
```
# 常见问题
## about页
source目录下下新建about/index.html
## Hexo 设置显示文章摘要，首页不显示全文
Hexo 主页文章列表默认会显示文章全文，浏览时很不方便，可以在文章中插入 <!--more--> 进行分段。
该代码前面的内容会作为摘要显示，而后面的内容会替换为 “Read More” 隐藏起来。

## 设置网站图标

进入 themes/主题 文件夹，打开 _config.yml 配置文件，找到 favicon 修改，一般格式为：favicon: 图标地址。（不同主题可能略有差别）

## 修改并部署后没有效果

使用 hexo clean 清理后重新部署。


## 创建图片资源文件夹
网上有关的解决方式几乎很大一部分会提到这一点：将_config.yml 文件中的post_asset_folder 选项设为 true 来打开。事实上这正是hexo官方文档给出的解决方案之一中的一个步骤。仔细阅读后会发现如下几点：

该操作的作用就是在使用hexo new xxx指令新建博文时，在相同路径下同步创建一个xxx文件夹，而xxx文件夹的作用就是用来存放图片资源；
就我个人而言，我偏好于直接在source\_posts文件夹下新建md文件，而不是通过hexo new xxx指令；
那么直接新建xxx.md再新建xxx文件夹，这种操作的最终效果和使用hexo new xxx指令新建博文的效果一样吗？经过实测，是一样的。
