---
title: Hexo Action
date: 2025-09-04 10:35:00
updated: 2025-09-08 01:20:00
tags:
  - hexo
categories:
  - 前端
description: "两个仓库一个工作流轻松搭建hexo博客（无需本地）"
cover:
---

# 基于 GitHub Actions 的 Hexo + Anzhiyu 主题全自动化部署指南
 
本文档提供一套无需本地执行命令的方案，所有初始化、主题安装及部署流程均通过 GitHub Actions 完成，核心是利用工作流文件操作仓库代码，仅需两步即可搭建博客。
 
## 一、前置准备（约1分钟）
 
1. 创建两个 GitHub 仓库
- 仓库 A（私有）：用于存放工作流配置和最终源代码，命名任意（如  hexo-anzhiyu-source ）。
- 仓库 B（公开）：作为 GitHub Pages 载体，必须命名为  <你的用户名>.github.io 。
2. 生成 GitHub Token
- 路径：GitHub 个人设置 →  Developer settings  →  Personal access tokens 。
- 权限：勾选  repo （全选） 和  workflow ，生成后复制保存。
3. 配置仓库 A 的 Token
- 路径：仓库 A →  Settings  →  Secrets and variables  →  Actions  →  New repository secret 。
- 名称： HEXO_DEPLOY_TOKEN （需与工作流代码一致），值为上述生成的 Token。
 
## 二、核心操作：提交 Action 工作流文件到仓库 A
 
直接在 GitHub 网页端操作仓库 A，创建包含完整流程的工作流文件，无需本地设备参与。
 
步骤 1：新建工作流文件
 
1. 进入仓库 A，点击右上角  Add file  →  Create new file 。
2. 文件名严格填写： .github/workflows/full-deploy.yml （路径和名称错误会导致 Action 无法识别）。
 
步骤 2：粘贴并修改工作流代码
 
将下方代码粘贴到文件中，仅需修改 3 处个人信息（已标注）：
 
```yaml    
name: Hexo Full Auto Deploy  # 工作流名称：全自动化部署

# 触发条件：1. 推送代码到仓库 A 的 main 分支时执行；2. 可手动触发（方便测试）
on:
  push:
    branches: [main]
  workflow_dispatch:  # 手动触发开关（在 Action 页面点击「Run workflow」即可执行）

jobs:
  full-setup-and-deploy:
    runs-on: ubuntu-latest  # 运行环境：Ubuntu 服务器
    steps:
      # 步骤 1：拉取仓库 A 的代码（初始为空，首次执行后会写入 Hexo 源码）
      - name: Checkout Source Repo (A)
        uses: actions/checkout@v4
        with:
          ref: main
          persist-credentials: false  # 关闭默认凭证，后续用自定义 Token

      # 步骤 2：初始化 Hexo 项目（若仓库 A 为空，首次执行会创建完整 Hexo 结构）
      - name: Initialize Hexo Project (If Not Exists)
        run: |
          # 检查是否已初始化 Hexo（通过根目录是否有 _config.yml 判断）
          if [ ! -f "_config.yml" ]; then
            npm install -g hexo-cli  # 全局安装 Hexo 脚手架
            mkdir temp-hexo && cd temp-hexo  # 创建临时空目录并进入
            hexo init .  # 在空临时目录执行初始化（避开非空问题）
            # 将临时目录的 Hexo 文件全部移动到仓库 A 根目录
            mv ./* ../ && mv ./.gitignore ../
            cd .. && rm -rf temp-hexo  # 回到根目录并删除临时目录
            npm install  # 安装 Hexo 依赖
          fi

      # 步骤 3：安装 Anzhiyu 主题（若未安装则克隆，已安装则更新）
      - name: Install/Update Anzhiyu Theme
        run: |
          THEME_DIR="themes/anzhiyu"
          # 检查主题目录是否存在，不存在则克隆，存在则拉取最新版本
          if [ ! -d "$THEME_DIR" ]; then
            git clone https://github.com/anzhiyu-c/hexo-theme-anzhiyu.git "$THEME_DIR"
            # 复制主题配置文件到根目录（方便后续修改）
            cp "$THEME_DIR/_config.yml" "_config.anzhiyu.yml"
          else
            cd "$THEME_DIR" && git pull  # 已安装则更新主题到最新版
          fi

      # 新增步骤：清除主题目录的 Git 信息（关键，解决子模块报错）
      - name: Remove Git Info from Theme Dir
        run: |
          THEME_DIR="themes/anzhiyu"
          if [ -d "$THEME_DIR/.git" ]; then
            rm -rf "$THEME_DIR/.git"  # 删除主题目录内的 Git 仓库文件，避免被识别为子模块
            rm -f "$THEME_DIR/.gitignore" "$THEME_DIR/.gitattributes"  # 清理残留的 Git 配置文件
          fi

      # 步骤 4：配置 Hexo 主配置（启用 Anzhiyu 主题，避免手动改文件）
      - name: Configure Hexo Main Settings
        run: |
          # 用 sed 命令直接修改 _config.yml，将主题改为 anzhiyu（覆盖默认的 landscape）
          sed -i 's/^theme: .*/theme: anzhiyu/' _config.yml

      # 步骤 5：安装 Hexo 依赖（确保依赖最新，尤其是主题新增依赖）
      - name: Install Dependencies
        run: |
          npm install -g hexo-cli
          npm install
          # 安装 Anzhiyu 主题推荐依赖（部分功能需要）
          npm install hexo-renderer-pug hexo-renderer-stylus --save

      # 步骤 6：构建 Hexo 静态文件（生成 public 目录）
      - name: Build Hexo Static Files
        run: |
          hexo clean  # 清理缓存
          hexo generate  # 构建静态文件到 public 目录

      # 步骤 7：将 public 目录部署到仓库 B（GitHub Pages）
      - name: Deploy to GitHub Pages (Repo B)
        uses: peaceiris/actions-gh-pages@v4
        with:
          personal_token: ${{ secrets.HEXO_DEPLOY_TOKEN }}  # 调用仓库 A 配置的 Token
          external_repository: newstartsw/newstartsw.github.io  # 替换为仓库 B 地址（例：zhangsan/zhangsan.github.io）
          publish_branch: main  # 仓库 B 用于承载静态文件的分支（默认 main）
          publish_dir: ./public  # Hexo 构建后的静态文件目录

      # 步骤 8：（关键）将 Hexo 源代码（含主题、配置）推回仓库 A 保存
      - name: Push Hexo Source Back to Repo A
        run: |
          # 配置 Git 身份（Action 环境需要）
          git config --global user.name "newstartsw"  # 替换为你的 GitHub 用户名
          git config --global user.email "newstartsw@outlook.com"  # 替换为你的 GitHub 绑定邮箱
          # 添加所有修改（初始化的 Hexo 代码、主题文件等）
          git add .
          # 提交信息（首次是初始化，后续是更新）
          git commit -m "Auto: Initialize Hexo + Anzhiyu theme / Update source" || echo "No changes to commit"
          # 用 Token 授权推送到仓库 A 的 main 分支
          git push https://${{ secrets.HEXO_DEPLOY_TOKEN }}@github.com/newstartsw/hexo-action.git main  # 替换为仓库 A 地址（例：zhangsan/hexo-anzhiyu-source）
```
 
## 三、触发工作流并验证
 
1. 提交文件：在仓库 A 的文件编辑页，填写提交信息（如  add full auto workflow ），点击  Commit new file 。
2. 触发执行
- 提交后自动触发工作流，可在仓库 A →  Actions  中查看  Hexo Full Auto Deploy  运行状态。
- 若未自动触发，手动点击  Run workflow  执行。
3. 验证结果
- 工作流成功（显示绿色对勾）后，仓库 A 会新增 Hexo 源代码（ _config.yml 、 themes/anzhiyu  等）。
- 访问  https://<你的用户名>.github.io ，即可看到 Anzhiyu 主题博客。

## 四、后续使用：修改博客内容
 
所有操作直接在 GitHub 网页端修改仓库 A 文件，提交后自动触发部署：
 
- 写文章：在  source/_posts  目录新建  .md  文件（Markdown 格式）。
- 改配置：修改根目录  _config.yml （Hexo 主配置）或  _config.anzhiyu.yml （主题配置）。
- 更主题：工作流会自动通过  git pull  更新 Anzhiyu 主题，无需手动操作。
 
## 五、关键说明
 
- 执行耗时：首次执行约 1-2 分钟（初始化、安装依赖），后续修改仅需 30 秒左右。
- 错误排查：失败时点击 Action 日志中红色叉号步骤，常见问题包括 Token 权限不足、仓库地址填错、Git 用户名/邮箱未修改。
- 安全性：仓库 A 设为私有，可避免  _config.yml  中的个人配置泄露。
