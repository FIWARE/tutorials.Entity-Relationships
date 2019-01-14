[![FIWARE Banner](https://fiware.github.io/tutorials.Entity-Relationships/img/fiware.png)](https://www.fiware.org/developers)

[![FIWARE Core Context Management](https://nexus.lab.fiware.org/repository/raw/public/badges/chapters/core.svg)](https://www.fiware.org/developers/catalogue/)
[![License: MIT](https://img.shields.io/github/license/fiware/tutorials.Entity-Relationships.svg)](https://opensource.org/licenses/MIT)
[![Support badge](https://nexus.lab.fiware.org/repository/raw/public/badges/stackoverflow/fiware.svg)](https://stackoverflow.com/questions/tagged/fiware)
[![NGSI v2](https://img.shields.io/badge/NGSI-v2-blue.svg)](https://fiware-ges.github.io/core.Orion/api/v2/stable/)
<br/>
[![Documentation](https://img.shields.io/readthedocs/fiware-tutorials.svg)](https://fiware-tutorials.rtfd.io)

このチュートリアルでは、FIWARE ユーザにバッチコマンドとエンティティのリレーショ
ンシップについて説明しています。チュートリアルでは、以前
の[ストア・ファインダの例](https://github.com/Fiware/tutorials.Getting-Started)で
作成されたデータを基にして、一連の関連するデータ・エンティティを作成して関連付け
て、単純な在庫管理システムを作成します。

このチュートリアルでは、[cUrl](https://ec.haxx.se/) コマンドを使用していますが
、[Postman documentation](https://fiware.github.io/tutorials.Entity-Relationships/)
も利用できます。

[![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/0671934f64958d3200b3)

# コンテンツ

<details>
<summary>詳細 <b>(クリックして拡大)</b></summary>

-   [エンティティとリレーションシップの理解](#understanding-entities-and-relationships)
    -   [在庫システム内のエンティティ](#entities-within-a-stock-management-system)
-   [アーキテクチャ](#architecture)
-   [前提条件](#prerequisites)
    -   [Docker と Docker Compose](#docker-and-docker-compose)
    -   [Cygwin for Windows](#cygwin-for-windows)
-   [起動](#start-up)
-   [データ・エンティティの作成と関連付け](#creating-and-associating-data-entities)
    -   [一度に複数のエンティティを作成](#creating-several-entities-at-once)
    -   [1 対多のリレーションシップの作成](#creating-a-one-to-many-relationship)
    -   [外部キーのリレーションシップの読み取り](#reading-a-foreign-key-relationship)
        -   [子エンティティから親エンティティへの読み取り](#reading-from-child-entity-to-parent-entity)
        -   [親エンティティから子エンティティへの読み取り](#reading-from-parent-entity-to-child-entity)
    -   [多対多のリレーションシップの作成](#creating-many-to-many-relationships)
    -   [ブリッジ・テーブルからの読み込み](#reading-from-a-bridge-table)
    -   [データの整合性](#data-integrity)
-   [次のステップ](#next-steps)

</details>

<a name="understanding-entities-and-relationships"></a>

# エンティティとリレーションシップの理解

FIWARE プラットフォーム内では、エンティティのコンテキストは、実世界に存在する物
理的または概念的オブジェクトの状態を表します。

<a name="entities-within-a-stock-management-system"></a>

## 在庫システム内のエンティティ

シンプルな在庫管理システムでは、4 つのタイプのエンティティのみが必要となります。
エンティティ間のリレーションシップは、次のように定義されます :

![](https://fiware.github.io/tutorials.Entity-Relationships/img/entities.png)

-   **Store** : ストアは実世界のレンガとモルタルの建物です。ストアには次のような
    プロパティがあります : + name : ストアの名前。例えば、"Checkpoint Markt" +
    address : ストアの住所。例えば、"Friedrichstraße 44, 10969 Kreuzberg, Berl
    in" + location : ストアの物理的なローケーション。例えば、_52.5075 N, 13.3903
    E_
-   **Shelf** : 棚は販売したいオブジェクトを保持するための、現実世界のデバイスで
    す。各棚には次のようなプロパティがあります :
    -   name : 棚の名前。例えば、"Wall Unit"
    -   location : 棚の物理的なローケーション。例えば、_52.5075 N, 13.3903 E_
    -   maximum_capacity : 棚の最大容量
    -   棚(shelf)が存在するストア(store)への関連付け
-   **Product** : 製品は販売するものとして定義されています。それは概念的なオブジ
    クトです。製品には次のような特性があります :
    -   name : 製品の名前。例えば、"Vodka"
    -   price : 製品の価格。例えば、13.99 ユーロ
    -   size : 製品のサイズ。例えば、小さい
-   **Inventory Item** : インベントリ項目は製品、店舗、棚、および物理的な物を関
    連付けるために使用される別の概念エンティティです。以下のようなプロパティを持
    ちます :
    -   product : 販売されている製品へのアソシエーション
    -   store : 製品が販売されている店舗への関連付け
    -   shelf : 製品が展示されている棚への関連
    -   stock_count : 倉庫で利用可能な製品の在庫数
    -   stelf_count : 棚で利用可能な製品の在庫数

ご覧のとおり、上記で定義されたエンティティのそれぞれは、変更される可能性のあるい
くつかのプロパティを含んでいます。製品の価格が変わる可能性があり、在庫が売却され
、在庫の在庫数が減る可能性があります。

> **注** このチュートリアルでは、次の表記スタイルを使用しています :
>
> -   エンティティタイプは**太字**です
> -   データ属性は `monospace text` に記述されています
> -   現実世界のアイテムはプレーン・テキストを使用します
>
> したがって、現実世界のストアは、**Store** エンティティによってコンテキスト・デ
> ータ内に表され、ストア内にある現実世界の棚は、`refStore` 属性を有する
> **Shelf** エンティティによってコンテキスト・データに表されます。

<a name="architecture"></a>

# アーキテクチャ

このアプリケーションは
、[Orion Context Broker](https://catalogue.fiware.org/enablers/publishsubscribe-context-broker-orion-context-broker)
という 1 つの FIWARE コンポーネントしか使用しません。アプリケーションが
_"Powered by FIWARE"_ と認定するには、Orion Context Broker を使用するだけで十分
です。

現在、Orion Context Broker はオープンソースの
[MongoDB](https://www.mongodb.com/) 技術を利用して、コンテキスト・データの永続性
を維持しています。したがって、アーキテクチャは 2 つの要素で構成されます :

-   [NGSI](https://fiware.github.io/specifications/OpenAPI/ngsiv2) を使用してリ
    クエストを受信する
    [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/)
-   バックエンドの [MongoDB](https://www.mongodb.com/) データベース
    -   Orion Context Broker が、データ・エンティティなどのコンテキスト・データ
        情報、サブスクリプション、登録などを保持するために使用します

2 つの要素間のすべての対話は HTTP リクエストによって開始されるため、エンティティ
はコンテナ化され、公開されたポートから実行されます。

![](https://fiware.github.io/tutorials.Entity-Relationships/img/architecture.png)

必要な設定情報は、関連する `docker-compose.yml` ファイルの services セクションに
あります:

```yaml
orion:
    image: fiware/orion:latest
    hostname: orion
    container_name: fiware-orion
    depends_on:
        - mongo-db
    networks:
        - default
    expose:
        - "1026"
    ports:
        - "1026:1026"
    command: -dbhost mongo-db -logLevel DEBUG
```

```yaml
mongo-db:
    image: mongo:3.6
    hostname: mongo-db
    container_name: db-mongo
    expose:
        - "27017"
    ports:
        - "27017:27017"
    networks:
        - default
    command: --bind_ip_all --smallfiles
```

両方のコンテナが同じネットワークに常駐しています。Orion Context Broker はポート
`1026` でリッスンしており、MongoDB はデフォルト・ポート `27071` でリッスンしてい
ます。 どちらのコンテナも同じポートを外部に公開しています。これはチュートリアル
のアクセス専用です。これにより、cUrl または Postman は同じネットワークに参加する
ことなくアクセスできます。 コマンドラインの初期化は、一目瞭然でなければなりませ
ん。

<a name="prerequisites"></a>

# 前提条件

<a name="docker-and-docker-compose"></a>

## Docker と Docker Compose

物事を単純にするために、両方のコンポーネントは [Docker](https://www.docker.com)
を使用して実行されます。**Docker** は、さまざまコンポーネントをそれぞれの環境に
分離することを可能にするコンテナ・テクノロジです。

-   Docker を Windows にインストールするには
    、[こちら](https://docs.docker.com/docker-for-windows/)の手順に従ってくださ
    い
-   Docker を Mac にインストールするには
    、[こちら](https://docs.docker.com/docker-for-mac/)の手順に従ってください
-   Docker を Linux にインストールするには
    、[こちら](https://docs.docker.com/install/)の手順に従ってください

**Docker Compose** は、マルチコンテナ Docker アプリケーションを定義して実行する
ためのツールです
。[YAML file](https://raw.githubusercontent.com/Fiware/tutorials.Entity-Relationships/master/docker-compose.yml)
ファイルは、アプリケーションのために必要なサービスを構成するために使用します。つ
まり、すべてのコンテナ・サービスは 1 つのコマンドで呼び出すことができます
。Docker Compose は、デフォルトで Docker for Windows と D ocker for Mac の一部と
してインストールされますが、Linux ユーザ
は[ここ](https://docs.docker.com/compose/install/)に記載されている手順に従う必要
があります。

次のコマンドを使用して、現在の **Docker** バージョンと **Docker Compose** バージ
ョンを確認できます :

```console
docker-compose -v
docker version
```

Docker バージョン 18.03 以降と Docker Compose 1.21 以上を使用していることを確認
し、必要に応じてアップグレードしてください。

<a name="cygwin-for-windows"></a>

## Cygwin for Windows

シンプルな Bash スクリプトを使用してサービスを開始します。Windows ユーザは
[cygwin](http://www.cygwin.com/) をダウンロードして、Windows の Linux ディストリ
ビューションに似たコマンドライン機能を提供する必要があります。

<a name="start-up"></a>

# 起動

リポジトリ内で Bash スクリプトが提供する
[services](https://github.com/Fiware/tutorials.Entity-Relationships/blob/master/services)
を実行することにより、コマンドラインからすべてのサービスを初期化することができま
す。リポジトリを複製し、以下のコマンドを実行して必要なイメージを作成してください
:

```console
git clone git@github.com:Fiware/tutorials.Entity-Relationships.git
cd tutorials.Entity-Relationships

./services start
```

このコマンドは、起動時に以前
の[ストア・ファインダのチュートリアル](https://github.com/Fiware/tutorials.Getting-Started)の
シード・データもインポートします。

> **注** : クリーンアップをやり直したい場合は、次のコマンドを使用して再起動する
> ことができます :

```console
./services stop
```

>

<a name="creating-and-associating-data-entities"></a>

# データ・エンティティの作成と関連付け

<a name="creating-several-entities-at-once"></a>

## 一度に複数のエンティティを作成

前のチュートリアルでは、各ストアのエンティティを個別に作成しました。

同時に 5 つの棚ユニットを作成できます。このリクエストでは、簡易バッチ処理エンド
ポイントを使用して 5 つの棚エンティティを作成します。バッチ処理では、2 つの属性
を持つペイロードで `/v2/op/update` エンドポイントを使用します
。`actionType=APPEND` は、既存のエンティティが存在する場合はそれを上書きすること
を意味しますが、`entities` 属性は更新するエンティティの配列を保持します。

**Store** エンティティから **Shelf** エンティティを区別するために、各棚が
`type=Shelf` に割り当てられています。'name' と 'location' のような実世界のプロパ
ティが各棚にプロパティとして追加されました。

#### :one: リクエスト :

```console
curl -iX POST \
  'http://localhost:1026/v2/op/update' \
  -H 'Content-Type: application/json' \
  -d '{
  "actionType":"APPEND",
  "entities":[
    {
      "id":"urn:ngsi-ld:Shelf:unit001", "type":"Shelf",
      "location":{
        "type":"geo:json", "value":{ "type":"Point","coordinates":[13.3986112, 52.554699]}
      },
      "name":{
        "type":"Text", "value":"Corner Unit"
      },
      "maxCapacity":{
        "type":"Integer", "value":50
      }
    },
    {
      "id":"urn:ngsi-ld:Shelf:unit002", "type":"Shelf",
      "location":{
        "type":"geo:json","value":{"type":"Point","coordinates":[13.3987221, 52.5546640]}
      },
      "name":{
        "type":"Text", "value":"Wall Unit 1"
      },
      "maxCapacity":{
        "type":"Integer", "value":100
      }
    },
    {
      "id":"urn:ngsi-ld:Shelf:unit003", "type":"Shelf",
      "location":{
        "type":"geo:json", "value":{"type":"Point","coordinates":[13.3987221, 52.5546640]}
      },
      "name":{
        "type":"Text", "value":"Wall Unit 2"
      },
      "maxCapacity":{
        "type":"Integer", "value":100
      }
    },
    {
      "id":"urn:ngsi-ld:Shelf:unit004", "type":"Shelf",
      "location":{
        "type":"geo:json", "value":{"type":"Point","coordinates":[13.390311, 52.507522]}
      },
      "name":{
        "type":"Text", "value":"Corner Unit"
      },
      "maxCapacity":{
        "type":"Integer", "value":50
      }
    },
    {
      "id":"urn:ngsi-ld:Shelf:unit005", "type":"Shelf",
      "location":{
        "type":"geo:json","value":{"type":"Point","coordinates":[13.390309, 52.50751]}
      },
      "name":{
        "type":"Text", "value":"Long Wall Unit"
      },
      "maxCapacity":{
        "type":"Integer", "value":200
      }
    }
  ]
}'
```

同様に、`type=Product` を使用して一連の **Product** エンティティを作成できます。

#### :two: リクエスト :

```console
curl -iX POST \
  'http://localhost:1026/v2/op/update' \
  -H 'Content-Type: application/json' \
  -d '{
  "actionType":"APPEND",
  "entities":[
    {
      "id":"urn:ngsi-ld:Product:001", "type":"Product",
      "name":{
        "type":"Text", "value":"Beer"
      },
      "size":{
        "type":"Text", "value": "S"
      },
      "price":{
        "type":"Integer", "value": 99
      }
    },
    {
      "id":"urn:ngsi-ld:Product:002", "type":"Product",
      "name":{
        "type":"Text", "value":"Red Wine"
      },
      "size":{
        "type":"Text", "value": "M"
      },
      "price":{
        "type":"Integer", "value": 1099
      }
    },
    {
      "id":"urn:ngsi-ld:Product:003", "type":"Product",
      "name":{
        "type":"Text", "value":"White Wine"
      },
      "size":{
        "type":"Text", "value": "M"
      },
      "price":{
        "type":"Integer", "value": 1499
      }
    },
    {
      "id":"urn:ngsi-ld:Product:004", "type":"Product",
      "name":{
        "type":"Text", "value":"Vodka"
      },
      "size":{
        "type":"Text", "value": "XL"
      },
      "price":{
        "type":"Integer", "value": 5000
      }
    }
  ]
}'
```

どちらの場合も、NGSI-LD
[ドラフト勧告](https://docbox.etsi.org/ISG/CIM/Open/ISG_CIM_NGSI-LD_API_Draft_for_public_review.pdf)に
従って各エンティティ `id` をエンコードしました。提案は、それぞれの `id` が 標準
フォーマットに従った URN というものです :
`urn:ngsi-ld:<entity-type>:<entity-id>`。これは、システム内のすべての `id` がユ
ニークであることを意味します。

棚情報は、`/v2/entities` エンドポイントで GET リクエストを行うことでリクエストで
きます。たとえば、`id=urn:ngsi-ld:Shelf:unit001` を使用すると、その **Shelf** エ
ンティティのコンテキスト・データが得られます :

#### :three: リクエスト :

```console
curl -X GET \
  'http://localhost:1026/v2/entities/urn:ngsi-ld:Shelf:unit001/?type=Shelf&options=keyValues'
```

#### レスポンス :

```json
{
    "id": "urn:ngsi-ld:Shelf:unit001",
    "type": "Shelf",
    "location": {
        "type": "Point",
        "coordinates": [13.3986112, 52.554699]
    },
    "maxCapacity": 50,
    "name": "Corner Unit"
}
```

このように、存在する 3 つの追加のプロパティ属性 `location`, `maxCapacity` と
`name` があります。

<a name="creating-a-one-to-many-relationship"></a>

## 1 対多のリレーションシップの作成

データベースでは、外部キーは 1 対多のリレーションシップを指定するためによく使用
されます。たとえば、すべての棚が 1 つのストアにあり、1 つのストアには多くの棚ユ
ニットがあります。この情報を記憶するためには、外部キーと同様のアソシエーション・
リレーションシップを追加する必要があります。バッチ処理を再び使用して、既存の
**Shelf** エンティティを修正して、各ストアにリレーションシップを保持する
`refStore` 属性を追加することができます
。[リンクト・データ](https://fiware-datamodels.readthedocs.io/en/latest/guidelines/index.html#modelling-linked-data)
に関する FIWARE データ・モデリング・ガイドラインによると、エンティティ属性が他の
エンティティへのリンクとして使用される場合、プレフィックス `ref` と ターゲットの
リンクト・エンティティ・タイプの名前を付けて名前を付ける必要があります。

`refStore` 属性値は、**Store** エンティティ自体に関連付けられた URN に対応します
。

URN は標準フォーマットに従います : `urn:ngsi-ld:<entity-type>:<entity-id>`

#### :four: リクエスト :

次のリクエストは、3 つの棚を `urn:ngsi-ld:Store:001` に、2 つの棚を
`urn:ngsi-ld:Store:002` に関連付けます。

```console
curl -iX POST \
  'http://localhost:1026/v2/op/update' \
  -H 'Content-Type: application/json' \
  -d '{
  "actionType":"APPEND",
  "entities":[
    {
      "id":"urn:ngsi-ld:Shelf:unit001", "type":"Shelf",
      "refStore": {
        "type": "Relationship",
        "value": "urn:ngsi-ld:Store:001"
      }
    },
    {
      "id":"urn:ngsi-ld:Shelf:unit002", "type":"Shelf",
      "refStore": {
        "type": "Relationship",
        "value": "urn:ngsi-ld:Store:001"
      }
    },
    {
      "id":"urn:ngsi-ld:Shelf:unit003", "type":"Shelf",
      "refStore": {
        "type": "Relationship",
        "value": "urn:ngsi-ld:Store:001"
      }
    },
    {
      "id":"urn:ngsi-ld:Shelf:unit004", "type":"Shelf",
      "refStore": {
        "type": "Relationship",
        "value": "urn:ngsi-ld:Store:002"
      }
    },
    {
      "id":"urn:ngsi-ld:Shelf:unit005", "type":"Shelf",
      "refStore": {
        "type": "Relationship",
        "value": "urn:ngsi-ld:Store:002"
      }
    }
  ]
}'
```

ここで棚情報が再度リクエストされると、レスポンスが変更され、前の手順で追加された
新しい `refStore` 属性が含まれます。

#### :five: リクエスト :

```console
curl -X GET \
  'http://localhost:1026/v2/entities/urn:ngsi-ld:Shelf:unit001/?type=Shelf&options=keyValues'
```

#### レスポンス :

`refStore` 属性を含む更新されたレスポンスを以下に示します :

```json
{
    "id": "urn:ngsi-ld:Shelf:unit001",
    "type": "Shelf",
    "location": {
        "type": "Point",
        "coordinates": [13.3986112, 52.554699]
    },
    "maxCapacity": 50,
    "name": "Corner Unit",
    "refStore": "urn:ngsi-ld:Store:001"
}
```

<a name="reading-a-foreign-key-relationship"></a>

## 外部キーのリレーションシップの読み取り

<a name="reading-from-child-entity-to-parent-entity"></a>

### 子エンティティから親エンティティへの読み取り

また、`options=values` 設定を使用して、既知の棚エンティティから `refStore` 属性
のリレーションシップ情報を取得するようリクエストすることもできます :

#### :six: リクエスト :

```console
curl -X GET \
  'http://localhost:1026/v2/entities/urn:ngsi-ld:Shelf:unit001/?type=Shelf&options=values&attrs=refStore'
```

#### レスポンス :

```json
["urn:ngsi-ld:Store:001"]
```

これは、"私は、`id=urn:ngsi-ld:Store:001` で **Store**エンティティと関連していま
す"と、解釈することができます。

<a name="reading-from-parent-entity-to-child-entity"></a>

### 親エンティティから子エンティティへの読み取り

親から子への読み込みは、`options=count` 設定を使用して行うことができます :

#### :seven: リクエスト :

```console
curl -X GET \
  'http://localhost:1026/v2/entities/?q=refStore==urn:ngsi-ld:Store:001&options=count&attrs=type&type=Shelf'
```

このリクエストは、URN `urn:ngsi-ld:Store:001` に関連付けられているすべての
**Shelf** エンティティの `id` をリクエストしています。レスポンスは、次に示されて
いるように JSON 配列です。

#### レスポンス :

```json
[
    {
        "id": "urn:ngsi-ld:Shelf:unit001",
        "type": "Shelf"
    },
    {
        "id": "urn:ngsi-ld:Shelf:unit002",
        "type": "Shelf"
    },
    {
        "id": "urn:ngsi-ld:Shelf:unit003",
        "type": "Shelf"
    }
]
```

普通の英語では、これは"`urn:ngsi-ld:Store:001` の中に 3 つの棚があります"と解釈
することができます。リクエストを変更して、`options=values` と `attrs` パラメータ
を使用して、関連する関連エンティティの特定のプロパティを返すことができます。例え
ば、次のようなリクエストです :

#### :eight: リクエスト :

```console
curl -X GET \
  'http://localhost:1026/v2/entities/?q=refStore==urn:ngsi-ld:Store:001&type=Shelf&options=values&attrs=name'
```

_"`urn:ngsi-ld:Store:001` にあるすべての棚の名前を教えてください。"_ というリク
エストとして解釈することができます。

#### レスポンス :

```json
[["Corner Unit"], ["Wall Unit 1"], ["Wall Unit 2"]]
```

<a name="creating-many-to-many-relationships"></a>

## 多対多のリレーションシップの作成

ブリッジ・テーブルは、多対多のリレーションシップを関連付けるためによく使用されま
す。たとえば、各ストアではさまざまな種類の商品が販売され、各商品はさまざまなスト
アで販売されます。

ブリッジ・テーブルは、多対多のリレーションシップを関連付けるためによく使用されま
す。たとえば、各ストアではさまざまな種類の商品が販売され、各商品はさまざまなスト
アで販売されます。

コンテキスト情報を"所与のストアの棚に製品を置く"ように保持するためには、他のエン
ティティからのデータを関連付けるために存在する新しいデータ・エンティティ
**InventoryItem** を作成する必要があります。それは、**Store**, **Shelf** と
**Product** エンティティとは外部キー関係を持つので、`refStore`、`refShelf`、およ
び `refProduct` というリレーションシップ属性が必要です。

製品を棚に割り当てることは、単にリレーションシップ情報と、`stockCount` と
`shelfCount` のようなその他の追加プロパティを保持するエンティティを作成すること
によって行われます。

#### :nine: リクエスト :

```console
curl -iX POST \
  'http://localhost:1026/v2/entities' \
  -H 'Cache-Control: no-cache' \
  -H 'Content-Type: application/json' \
  -H 'Postman-Token: 0588ef62-6b5c-4d1b-8066-172d63b516fd' \
  -d '{
    "id": "urn:ngsi-ld:InventoryItem:001", "type": "InventoryItem",
    "refStore": {
        "type": "Relationship",
        "value": "urn:ngsi-ld:Store:001"
    },
    "refShelf": {
        "type": "Relationship",
        "value": "urn:ngsi-ld:Shelf:unit001"
    },
    "refProduct": {
        "type": "Relationship",
        "value": "urn:ngsi-ld:Product:001"
    },
    "stockCount":{
        "type":"Integer", "value": 10000
    },
    "shelfCount":{
        "type":"Integer", "value": 50
    }
}'
```

<a name="reading-from-a-bridge-table"></a>

## ブリッジ・テーブルからの読み込み

ブリッジ・テーブル・エンティティから読み込む場合、エンティティの `type` を知る必
要があります。

少なくとも 1 つの **InventoryItem** エンティティを作成した後、次のリクエストを行
うことで、_"`urn:ngsi-ld:Store:001` の中にどの製品が販売されているか？"_ をクエ
リすることができます。

#### :one::zero: リクエスト :

```console
curl -X GET \
  'http://localhost:1026/v2/entities/?q=refStore==urn:ngsi-ld:Store:001&options=values&attrs=refProduct&type=InventoryItem'
```

#### レスポンス :

```json
[["urn:ngsi-ld:Product:prod001"]]
```

同様に、次のようにリクエストを変更することで、どのストアで
`urn:ngsi-ld:Product:001` が売れているのかを知ることができます :

#### :one::one: リクエスト :

```console
curl -X GET \
  'http://localhost:1026/v2/entities/?q=refProduct==urn:ngsi-ld:Product:001&options=values&attrs=refStore&type=InventoryItem'
```

#### レスポンス :

```json
[["urn:ngsi-ld:Store:001"]]
```

<a name="data-integrity"></a>

## データの整合性

コンテキスト・データのリレーションシップは、存在するエンティティ間でのみ設定およ
び維持する必要があります。つまり、URN `urn:ngsi-ld:<entity-type>:<entity-id>` は
コンテキスト内の別の既存エンティティにリンクする必要があります。したがって、ダン
グリング・リファレンスが残っていないエンティティを削除するときは、注意が必要です
。`urn:ngsi-ld:Store:001` が削除されたと想像してください。関連する **Shelf** の
エンティティに何が起こるべきですか？

以下のようにリクエストすることにより、削除前に残っているエンティティのリレーショ
ンシップが存在するかどうかを確認するリクエストを出すことができます :

#### :one::two: リクエスト :

```console
curl -X GET \
  'http://localhost:1026/v2/entities/?q=refStore==urn:ngsi-ld:Store:001&options=count&attrs=type'
```

#### :one::three: リクエスト :

このレスポンスには一連の **Shelf** と **InventoryItem** エンティティがリストされ
ています。製品とストアの間に直接のリレーションシップがないため、**Product** エン
ティティはありません。

```json
[
    {
        "id": "urn:ngsi-ld:Shelf:unit001",
        "type": "Shelf"
    },
    {
        "id": "urn:ngsi-ld:Shelf:unit002",
        "type": "Shelf"
    },
    {
        "id": "urn:ngsi-ld:Shelf:unit003",
        "type": "Shelf"
    },
    {
        "id": "urn:ngsi-ld:InventoryItem:001",
        "type": "InventoryItem"
    }
]
```

このリクエストによって空の配列が返された場合、エンティティには関連がありません。

<a name="next-steps"></a>

# 次のステップ

高度な機能を追加することで、アプリケーションに複雑さを加える方法を知りたいですか
？このシリーズ
の[他のチュートリアル](https://www.letsfiware.jp/fiware-tutorials)を読むことで見
つけることができます :

---

## License

[MIT](LICENSE) © 2018-2019 FIWARE Foundation e.V.
