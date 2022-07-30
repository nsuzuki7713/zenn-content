---
title: "gcloudのコマンド"
emoji: "😊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

PROJECT_ID: カスタマイズ可能な一意のプロジェクト ID。Google API で使用できる一意のユーザー割り当て ID
NAME: 人が読めるプロジェクト名。 Google API では使用されません
プロジェクト番号: 自動生成される一意のプロジェクト識別子。

https://cloud.google.com/resource-manager/docs/creating-managing-projects?hl=ja&visit_id=637524313412223386-3198541061&rd=1

## コマンド

バージョンとインストールされているコンポーネントを表示
`gcloud version`

現在の gcloud ツール環境の詳細を表示します。
`gcloud info`

cloud sdkのプロパティを見る
`gcloud config list`

自分のプロジェクト一覧を表示
`gcloud projects list`

activeのproject確認
`gcloud config configurations list`

projectの切り替え
`gcloud config set project ${projectName}`

実行時プロジェクト指定
`gcloud --project <project id>`


トークンを取得
gcloud auth print-identity-token

## 参照

https://cloud.google.com/sdk/gcloud/reference?hl=ja

https://qiita.com/masaaania/items/7a83c5e44e351b4a3a2c