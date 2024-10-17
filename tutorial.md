# Vertex AI Search for Retail で作る Google の検索技術で作る商品検索サービス

## このハンズオンについて

このハンズオンでは Vertex AI Search for Retail を使った商品情報のセマンティック検索を体験できます。

このハンズオンを通して、次の機能の使い方を学習できます。

- BigQuery へのデータインポート
- Vertex AI Search for Retail へのデータインポート
- Vertex AI Search for Retail の検索の評価

## プロジェクトの課金が有効化されていることを確認する

まず初めに、Google Cloud プロジェクトの課金が有効になっているか確認しましょう。

```bash
gcloud beta billing projects describe ${GOOGLE_CLOUD_PROJECT} | grep billingEnabled
```

**Cloud Shell の承認** という確認メッセージが出た場合は **承認** をクリックします。

出力結果の `billingEnabled` が **true** になっていることを確認してください。**false** の場合は、こちらのプロジェクトではハンズオンが進められません。別途、課金を有効化したプロジェクトを用意し、本ページの #1 の手順からやり直してください。

## 環境準備

<walkthrough-tutorial-duration duration=10></walkthrough-tutorial-duration>

最初に、ハンズオンを進めるための環境準備を行います。

下記の設定を進めていきます。

- gcloud コマンドラインツールの設定
- Google Cloud 機能（API）有効化の設定

### **1. gcloud コマンドラインツールとは?**

gcloud コマンドライン インターフェースは、Google Cloud でメインとなる CLI ツールです。このツールを使用すると、コマンドラインから、またはスクリプトや他の自動化により、多くの一般的なプラットフォーム タスクを実行できます。

たとえば、gcloud CLI を使用して、以下のようなものを作成、管理できます。

- Google Compute Engine 仮想マシン
- Google Kubernetes Engine クラスタ
- Google Cloud SQL インスタンス

**ヒント**: gcloud コマンドラインツールについての詳細は[こちら](https://cloud.google.com/sdk/gcloud?hl=ja)をご参照ください。

### **2. gcloud のデフォルト設定**

gcloud CLI に、デフォルトの設定を行っておきます。この設定を行っておくことで、よく行う設定（リージョン、プロジェクトの指定など）をコマンドを実行するたびに指定せずに済みます。

```bash
gcloud config set project ${GOOGLE_CLOUD_PROJECT}
```

### **3. 使用するサービスの API の有効化**

Google Cloud では、プロジェクトで使用するサービスの API を一つずつ有効化する必要があります。BigQuery の API を gcloud コマンドで有効化します。Vertex AI Search for Retail の API の有効化は、このあとの手順でコンソール上から行います。

```bash
gcloud services enable bigquery.googleapis.com cloudresourcemanager.googleapis.com iamcredentials.googleapis.com
```

## 商品カタログデータを Cloud Storage にアップロードする

商品カタログデータの元データはこの Git リポジトリの `data/products_raw.csv` に用意してあります。

自分の Google Cloud プロジェクトで Cloud Storage バケットを新規作成し、商品カタログデータをアップロードします。

まずは、専用の Cloud Storage バケットを新規作成します。

```bash
gcloud storage buckets create gs://vaisr-${GOOGLE_CLOUD_PROJECT} --location asia-northeast1
```

次のコマンドで CSV ファイルをアップロードします。

```bash
gcloud storage cp data/products_raw.csv gs://vaisr-${GOOGLE_CLOUD_PROJECT}/
```

ファイルがアップロードできているか確認しましょう。

```bash
gcloud storage ls gs://vaisr-${GOOGLE_CLOUD_PROJECT}
```

Cloud Storage コンソールから、実際の画像ファイルを確認することもできます。

## 商品データの CSV を BigQuery テーブルにインポートする

JSON ファイルのデータから、BigQuery テーブルを作成します。

まず、BigQuery データセットを作成します。

```bash
bq mk --dataset ${GOOGLE_CLOUD_PROJECT}:catalog
```

次のコマンドで、すでにアップロードしてある商品データ (CSV ファイル) を元に BigQuery テーブルを作成します。

```bash
bq load \
  --source_format=CSV \
  --autodetect \
  ${GOOGLE_CLOUD_PROJECT}:catalog.products_raw \
  gs://vaisr-${GOOGLE_CLOUD_PROJECT}/products_raw.csv
```

## Vertex AI Search for Retail の利用を開始する

次に Vertex AI Search for Retail に商品データをインポートし、検索を試してみましょう。

Vertex AI Search for Retail が Google Cloud プロジェクトで使用できるように設定します。Vertex AI Search for Retail の利用を開始するには、Vertex AI Search for Retail コンソールからデータ利用規約に同意する必要があります。

1. Google Cloud コンソールで [小売業向け Vertex AI Search] ページに移動します。ナビゲーションで見つからない場合は、検索ボックスに「小売業向け Vertex AI Search」と入力します。
2. [Vertex AI Search for Retail を設定する] ページで、[API を有効にする] をクリックします。
3. Vertex AI Search for Retail と Recommendations AI が オン と表示されたら、[続行] をクリックします。
4. Vertex AI Search for Industry のデータ使用条件を確認します。データ使用条件に同意した場合は、[承諾] をクリックします。有効化が完了するまでには 30 秒程度かかります。完了したら、[続行] をクリックします。
5. 検索機能と閲覧機能の有効化で [オンにする] をクリックします。
6. 最後に [開始] をクリックします。

## Vertex AI Search for Retail にデータをインポートする

Vertex AI Search for Retail に、BigQuery テーブルに作成済みの商品カタログデータをインポートします。

Vertex AI Search for Retail では、インポートすることができる商品カタログデータのスキーマが定義されています。そこでまずはじめに、現在保存している BigQuery テーブルを Vertex AI Search for Retail 用のデータに変換します。変換には BigQuery のビューの機能を利用し、実際のデータのコピーを行わずにインポートできるようにします。

ビューは `bq` コマンドから作成できます。

```bash
bq mk \
  --use_legacy_sql=false \
  --view "SELECT id, name AS title, description, [STRUCT(image AS uri)] AS images, [category] AS categories, STRUCT(price AS originalPrice, \"JPY\" as currencyCode) AS priceInfo, SPLIT(size, \",\") AS sizes FROM \`${GOOGLE_CLOUD_PROJECT}.catalog.products_raw\`" \
  "${GOOGLE_CLOUD_PROJECT}:catalog.products_view"
```

Vertex AI Search for Retail に商品カタログデータをインポートします。Vertex AI Search for Retail の API はサービスアカウントからしか呼び出せないため、サービスアカウントを作成し、サービスアカウントからインポート API を呼び出します。

`retail-api-client-sa` という名前のサービスアカウントを作成します。

```bash
gcloud iam service-accounts create retail-api-client-sa
```

`retail-api-client-sa` サービスアカウントには `roles/retail.editor` のロールを付与します。

```bash
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
     --member "serviceAccount:retail-api-client-sa@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com" \
     --role "roles/bigquery.dataEditor"

gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
     --member "serviceAccount:retail-api-client-sa@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com" \
     --role "roles/retail.editor"

gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
     --member "serviceAccount:retail-api-client-sa@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com" \
     --role "roles/retail.editor"
```

次のコマンドで、サービスアカウントのアクセストークンを取得し、そのアクセストークンを使ってインポート API を呼び出します。

```bash
curl -X POST \
  "https://retail.googleapis.com/v2/projects/${GOOGLE_CLOUD_PROJECT}/locations/global/catalogs/default_catalog/branches/default_branch/products:import" \
  -H "Authorization: Bearer $(gcloud auth print-access-token --impersonate-service-account retail-api-client-sa@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com)" \
  -H "Content-Type: application/json" \
  --data "{\"inputConfig\":{\"bigQuerySource\":{\"tableId\":\"products_view\",\"datasetId\":\"catalog\",\"projectId\":\"${GOOGLE_CLOUD_PROJECT}\"}}}"
```

## インポートされた商品カタログデータを確認する

インポートされた商品カタログデータは [データ] メニューから確認できます。

[カタログ] タブでは、インポートされた商品カタログデータが一覧できます。商品カタログデータのリストの各アイテムの [製品を表示] をクリックすると、対象の商品の JSON 形式のデータが確認できます。

[アクティビティのステータス] では、インポートしたジョブの現在の進捗が確認できます。インポート中にエラーが発生した際はこの画面から確認できます。

## Vertex AI Search for Retail の検索を評価する

Vertex AI Search for Retail のコンソールから、キーワードによる検索結果を試すことができます。どのように動作するか、実際に試してみましょう。

1. Vertex AI Search for Retail (小売向け検索) コンソールの [評価] をクリックします。
2. [検索] タブをクリックします。
3. 任意の検索クエリを入力し、[検索プレビュー] をクリックします。

様々な検索キーワードで検索を試してみてください。

## お疲れ様でした

以上でハンズオンは終了です。お疲れ様でした。

<walkthrough-conclusion-trophy></walkthrough-conclusion-trophy>