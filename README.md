# Mista的安卓之路

借鉴的很好的一个博客样板，用来学习。

[![Devices Mockup](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/commons/devices-mockup.png)](https://chirpy.cotes.info)

## 目录

- [特性](#特性)
- [前提](#前提)
- [安装](安装)
- [使用](#使用)
- [Documentation](#documentation)
- [Contributing](#contributing)
- [Credits](#credits)
- [Support](#support)
- [License](#license)

## 特性

- 指定post，`pin = true`
- 自定义主题模式
- 两级分类
- 显示post的上次修改日期
- 目录表
- 自动推荐相关posts
- 语法高亮
- 数学表达式
- 优美的diagram & flowchart
- 搜索功能
- Atom Feeds
- Comments功能
- 谷歌分析
- GA Pageviews reporting (Advanced)
- SEO and Performance Optimization

## 前提

参考 [Jekyll Docs](https://jekyllrb.com/docs/installation/) 安装 `Ruby`, `RubyGems`, `Jekyll` 和 `Bundler`.

## 安装

两种方式:

- **Install from RubyGems** -方便更新。参考原作者。
- **Fork on GitHub** - 方便自定义，但是不方便更新。这里用的是这种方式。

### Fork on GitHub

[Fork **Chirpy**](https://github.com/cotes2020/jekyll-theme-chirpy/fork) 在Github上clone到本地。

安装gem依赖:

```console
$ bundle
```

然后执行:

```console
$ bash tools/init.sh
```

脚本完成:

1. 移出仓库中的一些文件:
    - `.travis.yml`
    - `_posts`文件夹下内容
    - `docs`文件夹

2. 设置GitHub工作流程通过移除 `.github/workflows/pages-deploy.yml.hook`的 `.hook`, 然后移除 `.github`文件夹下内容。

3. 自动创建commit并提交。

## 使用

### 配置

按需更闹心 `_config.yml` 的变量。一些典型的变量如下:

- `url`
- `avatar`
- `timezone`
- `lang`

### 运行本地服务器

运行下面命令，预览网站内容:

```console
$ bundle exec jekyll s
```

浏览器打开 _<http://localhost:4000>_.

### 发布

部署网站前，确认 `_config.yml` 文件中 `url` 配置正确。 如果不打算部署网站到**GitHub Pages**，记得更改 `baseurl` 为项目名称，比如`Github Pages/project-name`。

#### 发布 GitHub Pages

出于安全考虑，**Github Pages**运行在安全模式，限制插件来生成额外的页面文件。 因此，可以使用 **GitHub Actions** 来建站，存储生产文件到新的分支，并用新的分支作为Github Pages的源文件。

快速检查**GitHub Actions**需要的文件:

- 确保有 `.github/workflows/pages-deploy.yml`文件。不然，创建新文件并复制内容 [workflow file][workflow]，并且 `on.push.branches` 值和默认分支同名。
- 确保 `tools/test.sh` 和 `tools/deploy.sh`文件。否则，从原项目拷贝过来。

然后重命名Github仓库为 `<GH-USERNAME>.github.io` 。

然后发布网站:

1. Push任何commit到远端来触发**GitHub Actions**工作。一旦建立成功，会生成新的分支`gh-pages` 保存生成的页面文件。

2. 浏览仓库地址并把 `gh-pages` 分支左右网站源文件。_Settings_ → _Options_ → _GitHub Pages_:

    ![gh-pages-sources](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/posts/20190809/gh-pages-sources.png)

3. 浏览网站，网址Github Setting会显示。
