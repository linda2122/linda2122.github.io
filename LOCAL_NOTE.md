# Hexo / Git / Markdown 

本文件是项目根目录的个人操作手册。提交前检查内容，不要在此记录密码、访问令牌或其他密钥。

## Hexo 文章与草稿

在项目根目录 `E:\vscode\myblog` 打开 PowerShell 执行命令。

### 新建文章

```powershell
npx hexo new "文章标题"
```

文章生成在 `source/_posts/`。当前 `post_asset_folder: true`，新文章会有对应的资源目录，可把本篇图片放在该目录中。
使用hexo new "文章标题"时，除了新建一个"文章标题.md"文件，还会有一个同名文件夹，将文章的图片放在其中，方便图片维护。若配置改为
false,则不会增加同名文件夹。

### 新建草稿

```powershell
npx hexo new draft "草稿标题"
```

草稿生成在 `source/_drafts/`。当前 `render_drafts: false`，普通生成和部署不会把草稿发布到网站。若仓库是公开的，还要确认草稿没有被 Git 跟踪；草稿内容不应提交到公开仓库。

### 本地预览

预览已发布文章：

```powershell
npx hexo server
```

在浏览器打开终端显示的本地地址，通常是 `http://localhost:4000/`。按 `Ctrl+C` 停止服务器。

预览草稿时，先在根目录 `_config.yml` 临时把 `render_drafts` 改为 `true`，再运行 `npx hexo server`。预览结束后恢复为 `false`；部署前可运行：

```powershell
npx hexo clean
npx hexo generate
```

确认配置仍为 `false` 后再提交和推送。

### 发布草稿

草稿文件例如 `source/_drafts/neural-network.md`，准备发布时运行：

```powershell
npx hexo publish neural-network
```

检查文件已移到 `source/_posts/`，再本地预览并提交推送。

### 排查预览失败

若提示 `Post not found`，检查文章中的 `{% post_link ... %}` 是否指向真实存在的文章文件。示例占位文字不能当作真实链接；未创建的章节先不要添加有效的 `post_link`。

## Git 提交与发布

正常发布流程：

```powershell
git status
git add source/_posts/文章文件名.md
git commit -m "Add article title"
git push origin main
```

按需替换要暂存的路径。若这次确实要提交全部改动，才使用 `git add .`。提交后到 GitHub 仓库的 **Actions** 页面确认部署成功。

常用检查：

```powershell
git status --short --branch
git remote -v
git log --oneline -5
```

若推送提示远端有新提交，先同步再推送：

```powershell
git pull --rebase origin main
git push origin main
```

本机 Git 网络备注：此仓库配置了代理 `http://127.0.0.1:7890`，需要代理程序正在运行。代理设置保存在本机 Git 配置中，不会随仓库提交。

## Markdown 常用语法

### 标题与文字

```markdown
## 二级标题
### 三级标题
**粗体**  *斜体*  ~~删除线~~  `行内代码`
```

标题前后留空行。文章标题由 Hexo 的 front matter `title` 提供，正文通常从 `##` 开始。

### 段落、列表与引用

段落之间空一行。无序列表使用 `-`，有序列表使用数字和点：

```markdown
- 第一项
- 第二项

1. 第一步
2. 第二步

> 引用一段文字
```

### 代码块

在三个反引号后标注语言，便于语法高亮：

````markdown
```powershell
npx hexo server
```
````

常用语言标记：`powershell`、`bash`、`javascript`、`python`、`yaml`、`text`。

### 链接

```markdown
[Hexo 文档](https://hexo.io/docs/)
[站内页面](/learning/deep-learning/)
```

链接到 Hexo 文章可使用 `post_link`，文章文件名不带 `.md`，且目标文章必须存在：

```markdown
{% post_link dl-01-linear-algebra 线性代数基础 %}
```

### 图片

全站共用图片放入 `source/images/`，例如 `source/images/diagram.png`，正文引用：

```markdown
![图片说明](/images/diagram.png)
```

单篇文章图片可放在与文章同名的资源目录中。当前 `post_asset_folder: true`，Hexo 支持使用资源标签：

```markdown
{% asset_img diagram.png 图片说明 %}
```

不要使用 `E:\...` 或 `file:///...` 这样的电脑本地路径；网站访客无法访问你的电脑文件。

### 表格、分隔线与任务清单

```markdown
| 名称 | 说明 |
| --- | --- |
| Hexo | 静态博客生成器 |

---

- [ ] 待完成
- [x] 已完成
```

## 本站设置速查

- 草稿目录：`source/_drafts/`
- 正式文章目录：`source/_posts/`
- 文章图片资源目录：开启 `post_asset_folder: true`
- 草稿默认不参与生成：`render_drafts: false`
- 推送 `main` 后由 GitHub Actions 自动构建和部署。