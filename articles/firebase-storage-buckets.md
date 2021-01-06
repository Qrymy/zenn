---
title: "Firebase Storage に bucket を追加する"
emoji: "📦"
type: "tech"
topics: ["firebase", "cloudstorage"]
published: true
---

## 動機

Firebase は利用を始めると勝手に Cloud Storage の bucket がアサインされ、特に何も考えずに使うことができます。
しかし、bucket 毎の細かな権限管理をしたい場合、ライフサイクルを適用してテンポラリな bucket を使いたいときなど、bucket 自体を増やしたいケースも出てくると思います。
Firebase SDK のドキュメントには別の bucket を使うコードも載っているのですが、Rules を個別に管理する方法等、少し悩む部分があったので記事を書くことにしました。

## bucket の追加

![コンソール](https://storage.googleapis.com/zenn-user-upload/1arhj6bp6zqq4m1oqfj5xa8nyplb)
_右上の三点アイコンを押下します_
![bucket を追加ボタン](https://storage.googleapis.com/zenn-user-upload/whtvbxo6wlrq31vcjgvo1o0nha9z)
まず、bucket を追加するのは非常にかんたんです。コンソールを開き、Storage のページに移動した上で右上の三点アイコンを押すと、「バケットを追加」というボタンが表示されるのでそこからは既存の bucket を利用するなり、新規作成するなりどちらでも構いません。
:::message
bucket を追加するには、Blaze プランにする必要があります
:::

## Rules をそれぞれに適用する

:::message
Rules をコンソールからベタ書きする運用の場合、このセクションは飛ばしてください
:::
悩んだのはこの部分です。現状のプロジェクトだと、`***-production` と `***-staging` という 2 つの環境を用意しており、それぞれに 2 つずつ bucket が存在している状況です。この場合に、Firebase CLI から Rules をデプロイする方法がわからず、しばらく悩みました、
結果としては、`.firebaserc` と `firebase.json` に以下のような記述をすると、それぞれに Rules を適用することができました。

```json:.firebaserc
{
  "projects": {
    "staging": "***-staging",
    "production": "***-production"
  },
  "targets": {
    "***-staging": {
      "storage": {
        "bucket-group-1": [
          "***-staging-bucket-1"
        ],
        "bucket-group-2": [
          "***-staging-bucket-2"
        ]
      }
    },
    "***-production": {
      "storage": {
        "bucket-group-1": [
          "***-production-bucket-1"
        ],
        "bucket-group-2": [
          "***-production-bucket-2"
        ]
      }
    }
  }
}
```

まずはこのような感じで、デプロイターゲットを設定します。この設定は、[ドキュメントのこの部分](https://firebase.google.com/docs/cli/targets?hl=ja#set-up-deploy-target-storage-database)で、CLI から設定する内容が記載されていますが、結果としては、`.firebaserc` に上記内容が追加されるのと同義のようです。

次に `firebase.json` にそれぞれのグループにどのルールを適応するかを記載します。

```json:firebase.json
{
  "storage": [
    {
      "target": "bucket-group-1",
      "rules": "storage.group1.rules"
    },
    {
      "target": "bucket-group-2",
      "rules": "storage.group2.rules"
    }
  ],
}
```

これでそれぞれの Rules を読み込んだ上でデプロイしてくれます。

## クライアントから bucket を切り替える

デフォルトの(init 時から自動で適応されている) bucket については、何も考えずに、

```ts
const storage = firebase.storage();
```

で利用できます。追加した bucket については、

```ts
const storage = firebase.app().storage("gs://bucket-name");
```

とすると利用できます。注意点としては、bucket 名の先頭に `gs://` を付与する必要があるということです。後述の admin SDK ではこれが必要ないので混同ポイントではあります。

## Admin SDK から bucket を切り替える

デフォルトの bucket については、

```ts
const storage = admin.storage();
```

これで大丈夫です。追加した bucket については、

```ts
const storage = admin.storage().bucket("bucket-name");
```

これで利用できます。
