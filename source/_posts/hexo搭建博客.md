---
title: hexo搭建博客
date: 2022-10-02
updated: 2023-09-16
tags: 
- hexo
- blog
---

## 1. 创建github仓库

首先在github上创建一个仓库，仓库名必须是形如`<用户名>.github.io`，
在项目都设置中找到`pages`选项

## 2. 安装hexo

执行如下命令安装`hexo`，如果没有安装`npm`，需要先安装`npm`，读者根据各自的发行版进行安装。

```bash
npm install -g hexo-cli
```

安装好`hexo`后，执行如下命令生成一个初始项目

```bash
hexo init <folder>
cd <folder>
npm install
```

## 3. 生成博客

初始项目中只有主页和一个博客，执行如下命令可以查看该初始博客

```bash
hexo clean # 清空博客
hexo g # 生成博客
hexo s # 调试博客
```

执行`hexo s`后，在浏览器中输入`http://localhost:4000/`，可以查看该初始博客。

## 4. 配置博客

博客的配置文件是`_config.yml`,

```yaml
theme: next

deploy:
  type: git
  repo: git@github.com:user-name/user-name.github.io.git
  branch: main

excerpt:
  depth: 3
  excerpt_excludes: []
  more_excludes: []
  hideWholePostExcerpts: true
```

### 4.1 theme 主题

其中`theme`配置主题，这里使用都`next`主题，该主题不是自带的，需要执行如下命令进行安装

```bash
git clone https://github.com/theme-next/hexo-theme-next themes/next
```

### 4.2 deploy 部署

先安装扩展`npm install hexo-deployer-git`

`deploy`配置部署的相关配置，其中的

1. `type`: 声明部署类型
2. `repo`: 声明远程地址
3. `branch`: 声明分支名

### 4.3. excerpt 首页内容隐藏

先安装扩展`npm install hexo-excerpt`

1. `depth`: 深度，可以理解为行

## 5. hexo 常用命令

1. `hexo clean`: 清空生成的博客
2. `hexo g`: 生成博客
3. `hexo s`: 开启本地服务(localhost:4000)，查看博客
4. `hexo d`: 部署博客，上传到git仓库中
