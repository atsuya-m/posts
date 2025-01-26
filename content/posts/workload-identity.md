+++
title = "Workload Identity 連携"
date = "2025-01-26T15:04:40+09:00"
author = "Atsuya"
tags = ["eks", "workloadidentity", "aws", "gcp"]
+++

# はじめに
AWS 上のアプリケーションから Google API を呼び出す必要があり、
その認証・認可の手法として Workload Identity 連携を設定したので備忘録です。

# Workload Identity 連携について
Workload Identity 連携の特徴はざっと以下のような感じです。

- 他のクラウドサービス上で動くアプリケーションから Google サービスアカウントの権限を借用することができる
- アプリケーション側に設定する credential 情報にシークレット情報は含まれないので、安全に取り回すことができる

前者自体は他の方法でも達成することができますが、
シークレット情報を含むためセキュリティ観点から取扱いが難しいです。

では、 Workload Identity 連携でどうやって後者を実現しているかというと、
サービスアカウントを所有している Google Cloud 側とアプリケーション側にそれぞれ受入れ設定とリクエスト設定を入れるからです。

イメージとしては、インターホンが鳴った時に声と顔を見て相手が誰かを識別して対応を変える感じですかね。

# Google Cloud 側の設定
Google Cloud 側にはワークロードの受入れ設定をしていきます。
が、その前に Workload Identity 連携で使われる重要な二つの概念を紹介しておきます。
把握しておくと読み進めやすいです。


用語 | 説明
-- | --
Pool | 複数のプロバイダをグループ化し、Google Cloud の IAM との橋渡しをする
Provider | 実際に認証を行う外部 ID ソース（OIDC/SAML）

## 1. Google サービスアカウントの作成
いつも通り Google サービスアカウントを作成しましょう。そしていつも通り自分の欲しい権限を付与しておきます。

そして Workload Identity 連携経由でサービスアカウントの認証が取れるように**ロールに「Workload Identity ユーザー」を付与しておきます。**

## 2. Workload Identity プールの作成
[コンソール](https://console.cloud.google.com/iam-admin/workload-identity-pools)からプールを作成します。
最初からプロバイダ設定も行う必要があるみたいですが、この時点では難しいのでプール名以外は適当に入力してとりあえずプールを作成します。

## 3. プロバイダの設定を追加する
コンソールから先ほど作成したプールのページを開くと「プロバイダを追加」ボタンがあります。

### プロバイダの種類
最初にプロバイダの種類を選択することになるのですが、自分の場合は EKS on Fargate (AWS) でアプリケーションを動かしているのでここでは OIDC を選択します。

> AWS じゃなくて OIDC なの？
>
> 逆に AWS という選択肢は ECS 専用だと思って良いと思います。 
> EKS の場合は、ポッド起動時にポッドに設定してある Kubernetes Service Account の認証情報を
> `/var/run/secrets/eks.amazonaws.com/serviceaccount/token` にマウントします。
> この認証情報の発行者は EKS クラスタ上にある OIDC プロバイダなので、まさに OIDC を選択できます。

### 発行元・オーディエンス
次に発行元とオーディエンスを設定していくのですが、
OIDC プロバイダの場合は発行されている jwt トークンをベースに設定項目を埋めていきます。

サンプルとして以下に EKS 上の OIDC プロバイダが発行した jwt トークンを記載します。
（なお、一部はセキュリティ観点からダミーの情報に書き換えています）

```
{
  "aud": [
    "sts.amazonaws.com"
  ],
  "exp": 1111111111,
  "iat": 1111111111,
  "iss": "https://oidc.eks.ap-northeast-1.amazonaws.com/id/XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
  "jti": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
  "kubernetes.io": {
    "namespace": "namespace-hoge",
    "node": {
      "name": "fargate-ip-172-21-157-189.ap-northeast-1.compute.internal",
      "uid": "ffffffff-gggg-hhhh-iiii-jjjjjjjjjjjj"
    },
    "pod": {
      "name": "atsuya-8b657f89d-wjzfc",
      "uid": "kkkkkkkk-llll-mmmm-nnnn-oooooooooooo"
    },
    "serviceaccount": {
      "name": "atsuya-service-account",
      "uid": "pppppppp-qqqq-rrrr-ssss-tttttttttttt"
    }
  },
  "nbf": 1111111111,
  "sub": "system:serviceaccount:namespace-hoge:atsuya-service-account"
}
```

上記のような jwt トークンの場合は以下のように設定します。

設定 | 値
-- | --
発行元(URL) | `https://oidc.eks.ap-northeast-1.amazonaws.com/id/XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX` 
オーディエンス | `sts.amazonaws.com`

### 属性
ここまでで、一応は発行元を設定したので同じ発行元であればOKというぐらいには緩い設定が完了しています。
とはいえそれだけだとセキュリティ敵に厳しいということがあるのでさらにこの jwt トークンから必要な値を属性として設定していきます。

この時、 jwt トークンの構文解析のために [CEL式](https://github.com/google/cel-spec/blob/master/doc/intro.md#introduction)を使用します。

基本的には `google.subject = assertion.sub` と入力しておけばOKですが、
自分の場合は「EKS上の Namespace は任意の値であってほしい」という需要があったので、マッピングを追加して
`attribute.ksa_name = assertion.sub.split(":")[3]` という属性を追加しました。

## 4. サービスアカウントとプールを紐付ける
ここまで設定が完了したら後は「どのサービスアカウント」の権限を借用するかを設定します。

コンソールの「アクセスを許可」から 1. で作成したサービスアカウントを選択し、 3. で設定した属性とその属性がどういう値の場合かを入力します。

例えば自分の場合は、

属性名 | 属性値
-- | --
ksa_name | atusya-service-account

です。
以上で Google Cloud 側の設定は完了です。


# アプリケーション側の設定
Google Cloud 側の設定が完了すると Workload Identity 用の credentials.json をダウンロードできます。
ダウンロードした json をアプリケーション側の任意のディレクトリに配置し、環境変数`GOOGLE_APPLICATION_CREDENTIALS`でそのファイルパスを指定します。

後はアプリケーション側の Google API Client Library がよしなに Workload Identity 連携フローに沿って Google サービスアカウントの認証を得てくれます。

> クライアントライブラリの具体的な処理順は Go クライアントの実装が一番分かり易かったです。
> もし、クライアントライブラリを使用せずに Workload Identity 連携をしたい場合は、そちらを参照ください。
