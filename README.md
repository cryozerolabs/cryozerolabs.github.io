# 1.基础工具
xcode-select --install

# 2.装 Ruby（建议用 rbenv 管理多个版本）
brew install rbenv ruby-build
rbenv init - zsh >> ~/.zshrc
source ~/.zshrc
rbenv install 3.3.4
rbenv global 3.3.4

# 3.装 Bundler
gem install bundler


用 Bundler 本地预览是 GitHub Pages 官方推荐的做法，能避免环境不一致导致的构建问题。
GitHub Docs
+1

在你的仓库里启动本地服务
git clone https://github.com/cryozerolabs/cryozerolabs.github.io
cd cryozerolabs.github.io

# 4.安装依赖（Gemfile 里会包含 github-pages 等依赖）
bundle install

# 5.启动本地预览（自动监测变更）
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