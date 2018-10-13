---
title: "Github PagesにHugoでブログを作成した"
date: 2018-10-13T23:56:01+09:00
draft: false
---
# USERNAME.github.ioリポジトリを作成
shoooohei.github.ioで作成

# Project pages or User/Organization pages
静的ページを  
https://shoooohei.github.io/  
か  
https://shoooohei.github.io/blog/  
などのルートディレクトリ直下に作りたいかどうかで違う。  
前者なら、User/Organization pages  
後者なら、Project pages  
今回は、ルートディレクトリは自分で静的なポートフォリオを作りたいから、後者で作成した。  

# hugoのインストール
```bash
brew install hugo
```

# 自分のリポジトリをクローン
```bash
git clone your-repository-url
cd shoooohei.github.io
```

# 静的ページを生成
```bash
hugo new site blog
cd blog/
```

# themeを入れる方法
```bash
cd /Users/shoheikawasaki/study/shoooohei.github.io/blog  
git submodule add -b master テンプレートのURL themes/テンプレート名  
cp themes/テンプレート名/exampleSite/config.toml .  
cp themes/テンプレート名/archetypes/* archetypes/  
```
### 基本情報を編集する  
```bash  
vi themes/hugo-tranquilpeak-theme/exampleSite/config.toml  
# e.g
git submodule add -b master https://github.com/kakawait/hugo-tranquilpeak-theme.git themes/hugo-tranquilpeak-them  
cp themes/hugo-tranquilpeak-theme/exampleSite/config.toml  
cp themes/hugo-tranquilpeak-theme/archetypes/* archetypes  
vi themes/hugo-tranquilpeak-theme/exampleSite/config.toml  
```

# 下書き記事を生成
```bash
hugo new post/new-post.md
```


# 静的ファイル生成。
```bash
hugo -t テーマ名  
hugo -t hugo-tranquilpeak-theme  
```

# ローカルでのリアルタイム更新
```bash
hugo server -D
```

# ブラウザで表示
```bash
localhost:1313
```

# デプロイするためのシェル
```sh
#!/bin/bash
echo -e "\033[0;32mDeploying updates to GitHub...\033[0m"

# Build the project.
hugo -t hugo-tranquilpeak-theme # if using a theme, replace with `hugo -t <YOURTHEME>`

cd ../
# Add changes to git.
git add -A

# Commit changes.
msg="rebuilding site `date`"
if [ $# -eq 1 ]
  then msg="$1"
fi
git commit -m "$msg"

# Push source and build repos.
git push origin master

cd blog/
```

# デプロイ
```bash
./deploy.sh
```
