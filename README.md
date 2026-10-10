# Zenn CLI

* [📘 How to use](https://zenn.dev/zenn/articles/zenn-cli-guide)

## Create Article

```bash
npx zenn new:article --slug python-error-setup

# title: "" # 記事のタイトル
# emoji: "😸" # アイキャッチとして使われる絵文字（1文字だけ）
# type: "tech" # tech: 技術記事 / idea: アイデア記事
# topics: [] # タグ。["markdown", "rust", "aws"]のように指定する
# published: true # 公開設定（falseにすると下書き）

npx zenn preview    # http://localhost:8000
# or specific port
npx zenn preview --port 3000

git add .
git commit -m "add article"
git push origin main
```

* publish at specific date and time

```bash
published: true
published_at: 2025-05-01 10:00
```

## Initial Setup

* create github repository: https://github.com/new
* connect repository to Zenn: https://zenn.dev/dashboard/deploys
* install Zenn CLI

```bash
npm init --yes
npm install zenn-cli

npx zenn init
git commit -m "Initial commit"
```

## Reference

* https://zenn.dev/zenn/articles/connect-to-github
* https://zenn.dev/zenn/articles/install-zenn-cli
* https://zenn.dev/zenn/articles/zenn-cli-guide
