---
title: "Hugo + PaperMod + Github Actions 个人博客搭建方案与问题解决"
date: 2025-06-23
draft: false
summary: "记录首次完整的 Hugo PaperMod 博客搭建流程，通过修改主题源码实现个人头像图片显示，并解决了 GitHub Actions 自动化部署中 Git Submodule 的核心错误。"
tags: ["Hugo", "PaperMod", "GitHub Actions", "Web开发", "教程", "博客"]
---

基于 PaperMod 主题，我们将从初始配置开始，重点解决两个核心问题：

1. 如何通过**修改主题布局**来正确添加个人头像（Avatar）；

2. 如何在**不破坏 Git 子模块（Submodule）完整性**的前提下，解决 GitHub Actions 自动化部署中的常见错误。

### 第一部分：基础配置与头像定制

#### 步骤 1：项目初始化

通过 Hugo CLI 创建站点，并使用 Git Submodule 添加 PaperMod 主题。

```bash
# 创建站点
hugo new site Mariooo7-blog-source
cd Mariooo7-blog-source
git init

# 添加 PaperMod 主题作为子模块
git submodule add [https://github.com/adityatelange/hugo-PaperMod.git](https://github.com/adityatelange/hugo-PaperMod.git) themes/PaperMod
```

#### 步骤 2：添加主题定制布局 (实现头像显示)

经过验证，PaperMod 主题的默认“Home-Info Mode”（即首页显示文章列表的模式）**本身并不支持显示个人头像**。我们需要通过 Hugo 的文件覆盖机制，提供一个我们自己的布局文件来添加这个功能。

1. **创建目录结构**: 在项目**根目录**下，创建以下路径的文件夹：`layouts/partials/`。
2. **创建自定义布局文件**: 在 `layouts/partials/` 文件夹中，创建一个新文件，命名为 `home_info.html`。
3. **添加代码**: 修改 `layouts/partials/home_info.html` 文件。如下修改能令其读取你在配置文件中指定的头像，并将其显示在首页。

```html
{{- with site.Params.homeInfoParams }}
<article class="first-entry home-info">
    {{- if .Avatar }}
        {{- $avatarPath := .Avatar }}
        {{- $avatar := "" }}
        {{- if or (hasPrefix $avatarPath "http://") (hasPrefix $avatarPath "https://") }}
            {{- $avatar = $avatarPath }}
        {{- else }}
            {{- with resources.Get $avatarPath }}
                {{- $avatarProc := .Resize "150x150" }}
                {{- $avatar = $avatarProc.RelPermalink }}
            {{- end }}
        {{- end }}
        {{- if $avatar }}
        <img src="{{ $avatar }}"
             alt="{{ .AvatarTitle | default "Author Avatar" }}"
             width="150"
             height="150"
             style="border-radius: 50%; margin: 0 auto 1em auto; display: block;">
        {{- end }}
    {{- end }}
    <header class="entry-header">
        <h1>{{ .Title | markdownify }}</h1>
    </header>
    <div class="entry-content">
        {{ .Content | markdownify }}
    </div>
    <footer class="entry-footer">
        {{ partial "social_icons.html" . }}
    </footer>
</article>
{{- end -}}
```

**核心机制**: 这段代码使用了 Hugo 的 `resources.Get` 函数。这个函数被设计为**只在 `/assets` 目录中寻找和处理资源**。

> 头像图片必须放在该目录下！

#### 步骤 3：提供图片资源与完整配置

1. **放置图片**: 将个人头像（如 `avatar.png`）和网站图标（如 `logo.png`）**全部放入项目根目录下的 `/assets` 文件夹**。
2. **配置 `hugo.yaml`**: 使用以下这份更完整的配置，它包含了头像、图标、菜单和常用功能的开关。注意，所有图片路径都只使用纯文件名。

`````yaml
baseURL: [https://Mariooo7.github.io/](https://Mariooo7.github.io/)
languageCode: zh-cn
title: "IU_Roam's Blog"
author: "IU_Roam"
theme: ["PaperMod"]
copyright: "© IU_Roam 2024"
enableRobotsTXT: true

params:
  defaultTheme: dark
  DateFormat: "2006-01-02"
  description: "IU_Roam's blog, portfolio and thoughts."

  homeInfoParams:
    Avatar: "avatar.png"
    Title: "IU_Roam"
    Content: "Young, Simple, Naive"

  socialIcons:
    - name: "github"
      url: "[https://github.com/Mariooo7](https://github.com/Mariooo7)"
    - name: "email"
      url: "mailto:maor7@mail2.sysu.edu.cn"

  assets:
    favicon: "logo.png"
    favicon16x16: "logo.png"
    favicon32x32: "logo.png"
    apple_touch_icon: "logo.png"

  ShowReadingTime: true
  ShowShareButtons: true
  ShowCodeCopyButtons: true
  ShowPostNavLinks: true

menu:
  main:
    - identifier: blogs
      name: Blogs
      url: /blogs/
      weight: 10
    - identifier: portfolio
      name: Portfolio
      url: /portfolio/
      weight: 20
    # ... 其他菜单项
`````

### 第二部分：解决自动化部署错误

在使用官方提供的 GitHub Actions 工作流进行自动化部署时，如果直接修改了 `themes/PaperMod` 文件夹内的代码，会导致部署失败。

**问题根源：私有修改与公共仓库的冲突**

在本地修改子模块（`themes/PaperMod`）并提交时，主项目会记录一个指向这个新 commit 的链接。然而，这个 commit 只存在于您的本地，并不存在于 PaperMod 的官方公共仓库。因此，当 GitHub Actions 尝试从官方仓库拉取这个不存在的 commit 时，就会报错 `did not contain [commit ID]`。

**最终解决方案：Fork 主题，掌握完全控制权**

最标准、最彻底的解决方案，将主题的修改纳入自己的版本控制体系。

1. **Fork 主题仓库**: 在 GitHub 上，访问 PaperMod 的官方仓库 (`adityatelange/hugo-PaperMod`)，点击 **"Fork"** 按钮。这会在账户下创建一个完全属于自己的主题副本，例如 `Mariooo7/hugo-PaperMod`。
2. **更新本地子模块的远程地址**: 在本地电脑上，进入主项目的**根目录**，打开终端，执行以下命令，将子模块的远程地址 `origin` 从官方仓库切换到自己的 Fork 仓库。

````bash
# 进入主题子模块的目录
cd themes/PaperMod

# 更改远程仓库的 URL
git remote set-url origin [https://github.com/Mariooo7/hugo-PaperMod.git](https://github.com/Mariooo7/hugo-PaperMod.git)

# （可选）验证更改是否成功
git remote -v
# 应该显示 origin 指向您的 Mariooo7/hugo-PaperMod.git
````

3. **推送主题修改**: 现在可以将本地对主题的所有修改，安全地推送到自己的 Fork 仓库中。

```bash
# 仍然在 themes/PaperMod 目录下
git add .
git commit -m "feat: Add my theme customizations"
git push origin master  # 或主题默认分支，如 main
```

4. **更新主项目的子模块引用**: 主题修改已经安全地保存在了自己的远程仓库。现在，回到主项目，记录下这个指向自己主题仓库的更新。

- **返回主项目根目录** `cd ../..`
- **提交子模块的更新** `git add themes/PaperMod` `git commit -m "fix: Point submodule to personal theme fork"` `git push origin main`

完成以上步骤后，自动化部署流程将完全正常。GitHub Actions 的 `checkout` 步骤会从自己的 Fork 仓库中拉取主题，其中包含了所有的自定义修改，部署自然会成功。



# 总结

无论是 Hugo 的资源管理，还是 Git 的子模块机制，它们都有自己的哲学。理解并遵循它们，事情会变得很简单；反之，则会陷入无尽的麻烦。

如果纠结怎么把事情做得尽善尽美不知道还要折腾多久，但按既定的目标完成一个博客的搭建还是有很多办法能多快好省地解决。