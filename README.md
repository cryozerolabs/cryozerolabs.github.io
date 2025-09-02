# 1.基础工具
```shell
xcode-select --install
```

# 2.装 Ruby（建议用 rbenv 管理多个版本）
```shell
brew install rbenv ruby-build
rbenv init - zsh >> ~/.zshrc
source ~/.zshrc
rbenv install 3.3.4
rbenv global 3.3.4
```

# 3.装 Bundler
```shell
gem install bundler
```


用 Bundler 本地预览是 GitHub Pages 官方推荐的做法，能避免环境不一致导致的构建问题。


在你的仓库里启动本地服务
``` shell
git clone https://github.com/cryozerolabs/cryozerolabs.github.io
cd cryozerolabs.github.io
```

# 4.安装依赖（Gemfile 里会包含 github-pages 等依赖）
```shell
bundle install
```

# 5.启动本地预览（自动监测变更）
``` shell
bundle exec jekyll serve --livereload


启动后访问：http://127.0.0.1:4000/
（jekyll serve 就是官方的本地预览命令；--livereload 支持热刷新。) 
Jekyll
+1

有用的启动参数

预览草稿与未发布文章：
bundle exec jekyll serve --livereload --drafts --unpublished

更接近线上（生产）环境预览：
JEKYLL_ENV=production bundle exec jekyll serve

局域网同网段设备访问：
bundle exec jekyll serve --host 0.0.0.0 --port 4000
```


# 6.草稿
## ✅ 方式一：_drafts/ 目录（官方草稿机制）
- 在仓库根目录建 _drafts/，里面写草稿文件（无日期也行）。

- 本地预览草稿：

```shell
bundle exec jekyll serve --drafts --livereload
```
- 生成时，Jekyll 会把草稿当成今天的日期来渲染（仅本地生效）。

- 同步：把 _drafts/ 纳入 Git，推到远端（最好私有仓库/私有分支），多台机器都能拉取继续写。

✅ 方式二：published: false（适合已在 _posts/ 的条目）

在 Front Matter 标个记号：
```yaml
---
title: 未完成
date: 2025-09-02
published: false
---
```
- 本地预览“未发布文章”：
```shell
bundle exec jekyll serve --unpublished --livereload
```
- 优点：文件仍在 _posts/，以后改成 published: true 就能上线；同样可以用 Git 同步。