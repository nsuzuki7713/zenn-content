---
title: "【GCP】Cloud IAMの概念を整理してみた"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [GCP,IAM,CloudIAM]
published: true
---


## はじめに

IAMを触る時、「ロールとポリシーの関係ってなんだっけ？」、「サービスアカウントってなんだっけ？」と毎回調べて雰囲気のまま使っていました。
今回は、再度勉強しなおしてCloud IAMについて纏めてみました。

## Cloud IAMとは

Cloud IAMとは「誰（ID）」が「どのリソースに対して」「どのようなアクセス権（ロール）」を持つかを定義することにより、アクセス制御を管理できます。
例えば、下記のようなこと設定をすることがで、適切なアクセル管理をすることが可能です。

- 「揮発者A」に「Cloud FunctionsとPub Sub」の「閲覧と編集」を行える
- 「Cloud Functionsの関数A」は「Cloud Strage」の「作成」のみを行える

## Cloud IAMの概念

公式ドキュメンに正しく詳細な説明があるので、こちらも参照いただければです。
https://cloud.google.com/iam/docs/overview?hl=ja#concepts_related_identity

ここでは自分が理解したものを、掻い摘んで説明します。

まずはイメージ図です。
![](https://storage.googleapis.com/zenn-user-upload/3xcojo6i4aa5qru4b3d17a0iad8p)

登場人物としてはメンバー、ロール、権限の3つです。
メンバは複数のロールを持つことができ、ロールは1つ以上の権限を持つ構成になります。メンバーが直接、権限を持つことはありません。

### 権限とは

リソースに対してどのような操作ができるのかを表しています。
権限はGCPであらかじめ用意されており、ユーザが編集などはしません。

例えばCloud Functionsの権限はこういうのがあります。この権限をもとに、メンバーはリソースへのアクセス可否が決まります。
![](https://storage.googleapis.com/zenn-user-upload/gq9zdcdc02utyc5d3byt5ynxsd4i)

### ロールとは

ロールとは権限を纏めたものです。
ロールには3種類のロールが存在します。

1. [基本ロール](https://cloud.google.com/iam/docs/understanding-roles?hl=ja#basic)
IAM の導入前に存在していたオーナー、編集者、閲覧者のロール。
Google Cloud のすべてのサービスに関して適応されるので、本番環境などには向いていません。
2. [事前定義ロール](https://cloud.google.com/iam/docs/understanding-roles?hl=ja#predefined_roles)
GCPがあらかじめ用意しているロールです。「App Engineロール」や「Pub/Sub のロール」などサービス毎に用意されています。
ベータ版になっているものは、互換性が保証されていないので本番環境で使用する際は注意が必要です。
また、新しい機能やサービスが追加された場合などは必要用に応じて権限を自動更新してくれます。
3. カスタムロール
カスタムロールはユーザが作成するロールです。
より細かい権限管理をできるので、最小権限の原則を適用できます。
以下の図はロールの作成画面です。任意の権限を自由に選択して、ロールを作成できます。
![](https://storage.googleapis.com/zenn-user-upload/zqtqbderzpsmqnhfmu0a1mv8xvsy)

### メンバーとは

リソースへのアクセス制御をする対象人物になります。「誰が」の箇所にあたります。

メンバーは下記タイプが存在します。

- Google アカウント
個人が使用するGoogle アカウントに関連付けられているメールアドレス
- サービス アカウント
個人ではなく、アプリケーションが使用するアカウント。
例えば、Cloud Functionsが作成する場合はサービスアカウントを指定します。
![](https://storage.googleapis.com/zenn-user-upload/ep3726e7wfubwapyz3wlds24cvzl)
- Google グループ
Google アカウントとサービス アカウントを複数まとめたもの
- Google Workspace ドメイン
組織の Google Workspace アカウントで作成されたすべての Google アカウントの仮想グループを表す
- Cloud Identity ドメイン
1つの組織内のすべての Google アカウントの仮想グループを表す
- 認証済みのすべてのユーザー
すべてのサービス アカウント、および Google アカウントで認証されたユーザー全員を表す
- 全ユーザー
認証されたユーザーと認証されていないユーザーの両方を含めて、インターネット上のユーザーを表す

より詳細に知りたい場合は[こちら](https://cloud.google.com/iam/docs/overview?hl=ja#concepts_related_identity)から確認してください。

## ポリシーとバインディング

Cloud IAMにはポリシーとバインディングという概念もあります。

イメージとしては下記のようになります。
![](https://storage.googleapis.com/zenn-user-upload/nof5szngz8ugfqdg1ksco9z6op2n)

「メンバー」と「ロール」の紐付けをバインディングと言い、それを纏めたものを「ポリシー」と言います。
1つ以上のメンバーを1つのロールにバインドします。`メンバー:ロール = 多:1` の関係です。

Cloud IAMではポリシーからメンバーが、対象リソースへの権限があるかを調べます。

コンソールを使用してメンバーを作成する際、「ポリシー」と「バインディング」を明示的に作成しないので少し分かりづらいです。
実際に下記のように作成した場合は2つのバインディングが作成されています。
またバインディングに条件を付けることができ、日付や時間帯の指定やリソース名の比較なども可能です。
![](https://storage.googleapis.com/zenn-user-upload/k6cb2jpz7wq380c73emu3czk6z6y)

ポリシーとバインディングに関しては実際のデータ構造を確認するのが一番分かりやすいと思います。
下記コマンドでポリシーの一覧を取得できます。
`gcloud projects get-iam-policy ${PROJECT_ID} --format json`

そうすると下記のようなオブジェクトが返ってきます。
構造を確認するとポリシーがオブジェクト全体で、その中に`bindings`があります。
`bindings`では`role`と`members`があり、1対多の関係になっているのが分かります。
```json
{
  "bindings": [
    {
      "members": [
        "serviceAccount:aaaa.gserviceaccount.com",
        "serviceAccount:bbbb@appspot.gserviceaccount.com"
      ],
      "role": "roles/editor"
    },
    {
      "members": [
        "user:xxxx@gmail.com"
      ],
      "role": "roles/owner"
    },
  ],
  "etag": "AAA-BvS0zfw=",
  "version": 1
}
```

## おわりに

今回、Cloud IAMを勉強しましたが、ロールやポリシーなどAWSのIAMとは概念が異なるので注意が必要ですね。(現場で同じように使ってましたが、もしかしたら話が噛み合ってなかったかも...)
また、AWS IAMの記事はたくさんありますが、Cloud IAMの記事はまだまだ少ないですね。

## 参照

https://www.isoroot.jp/blog/1244/

https://qiita.com/os1ma/items/df01080dfa81c185357e