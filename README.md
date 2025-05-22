# 我的个人博客

这是一个使用 [Hexo](https://hexo.io/) 静态博客框架搭建的个人博客项目，博客地址：[李博帅的博客](https://lbs.wiki/)，搭建教程：[Centos部署Hexo个人博客实战指南](https://lbs.wiki/pages/b3552560/)。

## 项目概览

本项目基于 Hexo，用于发布和管理个人文章、技术笔记或其他内容。通过 Hexo，可以将 Markdown 格式的文章转换成静态网页，方便部署到 GitHub Pages, Vercel, Netlify 等平台。

## 技术栈

- **框架:** Hexo
- **包管理:** pnpm (根据 `pnpm-lock.yaml` 判断)
- **前端:** 依赖 Hexo 主题（本项目可能使用了 `icarus` 和 `landscape` 主题，具**体使用哪个**可以在 `_config.yml` 中配置）
- **开发语言:** JavaScript (Node.js 环境)

## 主要内容

- **文章:** 存放于 `source/_posts/` 目录下，使用 Markdown 格式编写。
- **主题:** 不同主题位于 `themes/` 目录下。
- **配置:** 项目主要配置在 `_config.yml` 文件中。主题的配置可能在主题目录下的 `_config.yml` 或项目根目录下的 `_config.[theme_name].yml` 文件中。

## 环境要求

- [Node.js](https://nodejs.org/) (建议使用 LTS 版本)
- [pnpm](https://pnpm.io/installation) (或者 npm/yarn, 具体使用哪个取决于您首次安装依赖时的方式)

## 快速开始 (本地开发)

1.  **克隆仓库:**
    ```bash
    git clone <您的仓库地址>
    cd <您的仓库目录>
    ```
   (如果您已经在项目中，则跳过此步)

2.  **安装依赖:**
    ```bash
    pnpm install
    ```
    (如果您 prefer 或使用 npm 或 yarn，请替换为 `npm install` 或 `yarn install`)

3.  **生成并运行本地服务器:**
    ```bash
    hexo clean  # 清理缓存和生成的静态文件 (可选)
    hexo generate # 生成静态文件 (可选，hexo server 会自动生成)
    hexo server -L # 启动本地服务器并在文件变动时自动刷新 (-L 表示监听文件变动)
    ```
    或者直接简化为:
    ```bash
    hexo s -L
    ```

4.  **访问:** 在浏览器中打开 `http://localhost:4000` 查看您的博客。

## 创建新文章

运行以下命令创建一篇新的 Markdown 文章：

```bash
hexo new "文章标题"
```
这将在 `source/_posts/` 目录下创建一个新的 Markdown 文件。编辑该文件来撰写您的文章。

## 生成静态文件

当您对博客内容或配置进行更改后，需要重新生成静态文件：

```bash
hexo generate
```
生成的静态文件位于 `public/` 目录下。

## 部署

根据您的部署方式（如 Hexo-cli 部署、手动复制 `public` 文件夹等），执行相应的部署命令。
如果您配置了部署方式，例如到 GitHub Pages:

```bash
hexo deploy
```
请确保 `.deploy_git` 目录是用于存储部署到 Git 仓库的内容。

## 文件结构说明 (简要)

- `.deploy_git/`: 可能用于存放部署到远程 Git 仓库的内容。
- `.idea/`: IDE (如 IntelliJ IDEA) 相关的项目文件。
- `node_modules/`: 项目依赖的 Node.js 模块。
- `public/`: Hexo 生成的静态网页文件，用于部署。
- `scaffolds/`: 文章、页面等的模板文件。
- `source/`: 存放原始文件。
    - `_posts/`: 您的博客文章 Markdown 文件。
    - 其他目录可能包含页面、草稿等。
- `themes/`: 存放不同的 Hexo 主题。
- `_config.yml`: Hexo 的主配置文件。
- `_config.[theme_name].yml`: 特定主题的配置文件 (如 `_config.icarus.yml`, `_config.landscape.yml`)。
- `db.json`: Hexo 的数据缓存文件。
- `LICENSE`: 项目的许可证信息 (如果包含)。
- `package.json`: Node.js 项目的配置文件，记录项目信息和依赖。
- `pnpm-lock.yaml`: 使用 pnpm 安装依赖时生成的锁定文件，确保依赖版本一致性。
- `.gitignore`: Git 版本控制忽略文件，在此文件中的文件不会被 Git 跟踪。

---
