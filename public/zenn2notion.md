---
title: Zennで「いいねした投稿」と「読んでいる本」をNotionで管理する
tags:
  - Python
  - Notion
  - Zenn
private: false
updated_at: '2024-02-10T13:14:18+09:00'
id: 3f3b12b172db47c72d91
organization_url_name: null
slide: false
ignorePublish: false
---

# 何を作ったか

Zennで自分が「いいねした投稿」と「読んでいる本」をNotionのデータベースに保存するスクリプトを作りました。[Notion]((https://www.notion.so/ja-jp))のアカウントとpythonの実行環境があれば誰でも使えるようになっています。

すぐに試したい方は、[こちら](https://github.com/Taichiro-S/Zenn2Notion)のREADMEを読んで試してみてください

こんなもの↓ができます！（これは私がいいねした投稿の一覧です）

https://zealous-rosehip-7a8.notion.site/a3c1dd8fa96a42fbb6e162e91354a68d?v=324c9e7e593746838d64ee2123d175e6&pvs=4

# 前書き

皆様は**Zennでいいねした記事**や**読んでいる本**を、どうやって管理していますか？
全部覚えとるわ😤という方はこの記事を読む必要はありません！

私のように読んだ記事を片っ端からいいねして保存した気になっている人は、

- はぇ〜勉強になるなあ（いいねポチッ） → どこいった...🤔
- これ、後で読んどこ〜（いいねポチッ） → どこいった...🤔

となることがよくあります

そんなわけで、

- タイトルとかで**検索やソートが**できて
- 後で読むとか**タグを追加**できて
- **APIが公開**されてて

そんなデータベースみたいなもので管理できたら楽なんだけどな〜
そんな都合のいいものあるわけ...

ｱﾙﾖ( ´・ω・)_ ~ [Notion](https://www.notion.so/ja-jp)

あるんですねこんな便利なものが
じゃあ、早速実装でい

# 実装

## Zennでいいねした投稿を取得(postman)

ZennはAPIを公開してはいませんが内部的に利用しているようなので、今回はそれを使わせてもらいます

ブラウザでZennにログインした状態で、[ここ](https://zenn.dev/api/me/library/likes)です
自分がいいねした投稿がjsonで表示されましたね？

postmanなどでブラウザを介さずにこの情報を取得するには、ブラウザに保存されている認証情報が必要になります（認証情報がないと「ログインしてください」と返されます）

色々試して、Cookieに入っている**remember_user_token**の値が必要ということがわかりました
Zennにログインした状態でChromeの検証画面を開き、Application → Cookiesの順に選択すると出てきます

![認証情報](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3065616/9f1835c2-4ad5-8d2e-daf3-6a046739de4c.png)

ヘッダーに以下のようにクッキーをセットして、`https://zenn.dev/api/me/library/likes`にpostmanでAPIリクエストしてみましょう
|Key|Value|
|-|-|
|Cookie|remember_user_token=トークンの値|

いいねした投稿が取得できるハズです
読んだ本についても同様に取得できます

ここからはpythonで実装していきます

## Zennでいいねした投稿を取得(python)

### コードの解説

- リクエストの間隔は1秒開けます（**ここは変更しないようにしてください**）
- `fetch_likes`：いいねした投稿のリストを取得します（Notionのデータベースに入っている記事のURLを引数にとり、重複しないように記事を取得します）
- `fetch_article_topics`：いいねした各投稿の詳細情報を取得します
- `fetch_reading_books`：読んでいる本のリストを取得します
- `fetch_book_details`：読んでいる本の詳細情報を取得します

https://gist.github.com/Taichiro-S/d2b4caf14848b7746c8c732ea14e23fa

## 取得したデータをNotionに保存

### コードの解説

- Zennで使われている絵文字で、notionが対応していないものがあるので、`DEFAULT_EMOJI`(😅)で置き換えるようにしています（好きなものに変えてください！）
- `fetch_urls`：Notionnデータベースにすでに保存されている記事や本のURLを取得します
- `insert_article`：いいねした投稿をデータベースに保存します（ここで保存するプロパティを指定しています）
- `insert_book`：読んでいる本をデータベースに保存します

(注)いいねを取り消したものについてはデータベースからの削除は行わないため、手動での削除が必要になります。

https://gist.github.com/Taichiro-S/f5456adb8bf09fb94e4e360c6a04dea2

## できなかったこと

記事の本文をNotionのページに保存しようとしましたが、Notionではblockという独自の仕組みを使っており、またZennでもマークダウンに加えて独自の記法があるので、変換がメンドイです

以下2つのライブラリを試してみましたがダメでした（Zenn独自の記法のため？）

- [md2notion](https://pypi.org/project/md2notion/)：pythonのライブラリ
- [@tryfabric/martian](https://www.npmjs.com/package/@tryfabric/martian)：javascriptのライブラリ

参考記事

https://zenn.dev/cizneeh/articles/markdown-to-notion-db

本文が保存できればNotion AIとか使えそうなのにな〜
何かいい方法があればコメントで教えてください！

# 使い方

## 事前準備

[Notion](https://www.notion.so/)のアカウントを作成しておいてください
pyenvなどでpython3.11または3.12をインストールしてください（これ以外での動作確認はしていません）

## 実行環境の作成

[こちら](https://github.com/Taichiro-S/Zenn2Notion)のリポジトリをクローンしたら、以下のコマンドでvenvを使用して実行環境を作成します
`python -m venv .venv`
`source .venv/bin/activate`
`pip install -r requirements.txt`

## Notionの準備

[ここ](https://www.notion.so/my-integrations)からNotion Integrationを作成し、発行されるSecretを控えておきます

以下のURLで公開されている2つのNotionのデータベースをそれぞれ自分のワークスペースに複製し、IDを控えておきます（   IDはデータベースのURLに`https://www.notion.so/[DATABASE ID]?v=[VIEW ID]`という形で入っています）

- [いいねした投稿保存用](https://zealous-rosehip-7a8.notion.site/8d13f37a21914981840a995f70272d37?v=e498ea5550174a249f0dbae5af86b556&pvs=4)
- [読んでいる本保存用](https://zealous-rosehip-7a8.notion.site/636574be4b7648349f217a735402b3ba?v=244e9c4d26b64af09889f84b10151689&pvs=4)

複製したデータベースのコネクトに作成したIntegrationを追加します

**複製したデータベースのプロパティの名前や種類は基本的には変更しないでください**
変更したい場合は、ソースコードの`insert_article`で対応する部分を書き換える必要があります

## 環境変数の設定

`.env.example`を`.env`という名前に変えて以下の4つの値を設定します。
**tokenやsecret等の認証情報の取り扱いには注意してください**

```.env
NOTION_SECRET='Notion Integration の Secret'
NOTION_DATABASE_ID_FOR_LIKES='いいねした投稿保存用のデータベースID'
NOTION_DATABASE_ID_FOR_BOOKS='読んでいる本保存用のデータベースID'
REMEMBER_USER_TOKEN='Cookieに保存されている remember_user_token の値'
```

:::note alert

`.env`の中身はgithubにアップロードしないように注意してください(`.gitignore`には書いてあります)

:::

## 実行

`python save_zenn_likes.py`でいざ実行！

こんな感じ↓で動くはずです！

https://youtu.be/aaC8-syNSeM

# 最後に

Notionデータベースでは検索、ソート、フィルターが可能です
また、Notion AIや他のツールと連携すればもっと活用法はありそうです

この方法で記事の管理が少しでも楽になれば幸いです！

