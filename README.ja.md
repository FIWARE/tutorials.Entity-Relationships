[![FIWARE Banner](https://fiware.github.io/tutorials.Entity-Relationships/img/fiware.png)](https://www.fiware.org/developers)

[![FIWARE Core Context](https://img.shields.io/badge/FIWARE-Core_Context-233c68.svg)](https://www.fiware.org/developers/catalogue/)
[![Documentation](https://readthedocs.org/projects/fiware-tutorials/badge/?version=latest)](https://fiware-tutorials.readthedocs.io/en/latest)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![NGSI v2](https://img.shields.io/badge/NGSI-v2-blue.svg)](http://fiware.github.io/context.Orion/api/v2/stable/)

このチュートリアルでは、FIWARE ユーザにバッチコマンドとエンティティのリレーションシップについて説明しています。チュートリアルでは、以前の[ストア・ファインダの例](https://github.com/Fiware/tutorials.Getting-Started)で作成されたデータを基にして、一連の関連するデータ・エンティティを作成して関連付けて、単純な在庫管理システムを作成します。

このチュートリアルでは、[cUrl](https://ec.haxx.se/) コマンドを使用していますが、[Postman documentation](http://fiware.github.io/tutorials.Entity-Relationships/) も利用できます。

[![Run in Postman](https://run.pstmn.io/button.svg)](https://www.getpostman.com/collections/0671934f64958d3200b3)
 

# コンテンツ

- [エンティティとリレーションシップの理解](#understanding-entities-and-relationships)
  * [在庫システム内のエンティティ](#entities-within-a-stock-management-system)
- [アーキテクチャ](#architecture)
- [前提条件](#prerequisites)
  * [Docker とDocker Compose](#docker-and-docker-compose)
  * [Cygwin for Windows](#cygwin-for-windows)
- [起動](#start-up)
- [データ・エンティティの作成と関連付け](#creating-and-associating-data-entities)
  * [一度に複数のエンティティを作成](#creating-several-entities-at-once)
  * [1対多のリレーションシップの作成](#creating-a-one-to-many-relationship)
  * [外部キーのリレーションシップの読み取り](#reading-a-foreign-key-relationship)
    + [子エンティティから親エンティティへの読み取り](#reading-from-child-entity-to-parent-entity)
    + [親エンティティから子エンティティへの読み取り](#reading-from-parent-entity-to-child-entity)
  * [多対多のリレーションシップの作成](#creating-many-to-many-relationships)
  * [ブリッジ・テーブルからの読み込み](#reading-from-a-bridge-table)
  * [データの整合性](#data-integrity)
- [次のステップ](#next-steps)

<a name="understanding-entities-and-relationships"></a>
# エンティティとリレーションシップの理解

FIWARE プラットフォーム内では、エンティティのコンテキストは、実世界に存在する物理的または概念的オブジェクトの状態を表します。

<a name="entities-within-a-stock-management-system"></a>
## 在庫システム内のエンティティ

シンプルな在庫管理システムでは、4つのタイプのエンティティのみが必要となります。エンティティ間のリレーションシップは、次のように定義されます :

![](https://fiware.github.io/tutorials.Entity-Relationships/img/entities.png)

* **Store** : ストアは実世界のレンガとモルタルの建物です。ストアには次のようなプロパティがあります :
    + name : ストアの名前。例えば、"Checkpoint Markt"
    + address : ストアの住所。例えば、"Friedrichstraße 44, 10969 Kreuzberg, Berl
in"
    + location : ストアの物理的なローケーション。例えば、*52.5075 N, 13.3903 E*
* **Shelf** : 棚は販売したいオブジェクトを保持するための、現実世界のデバイスです。各棚には次のようなプロパティがあります :
    + name : 棚の名前。例えば、"Wall Unit"
    + location : 棚の物理的なローケーション。例えば、*52.5075 N, 13.3903 E*
    + maximum_capacity : 棚の最大容量
    + 棚(shelf)が存在するストア(store)への関連付け
* **Product** : 製品は販売するものとして定義されています。それは概念的なオブジクトです。製品には次のような特性があります :
    + name : 製品の名前。例えば、"Vodka"
    + price : 製品の価格。例えば、13.99ユーロ
    + size : 製品のサイズ。例えば、小さい
* **Inventory Item** : インベントリ項目は製品、店舗、棚、および物理的な物を関連付けるために使用される別の概念エンティティです。以下のようなプロパティを持ちます :
    + product : 販売されている製品へのアソシエーション
    + store : 製品が販売されている店舗への関連付け
    + shelf : 製品が展示されている棚への関連
    + stock_count : 倉庫で利用可能な製品の在庫数
    + stelf_count : 棚で利用可能な製品の在庫数


ご覧のとおり、上記で定義されたエンティティのそれぞれは、変更される可能性のあるいくつかのプロパティを含んでいます。製品の価格が変わる可能性があり、在庫が売却され、在庫の在庫数が減る可能性があります。


> **注** このチュートリアルでは、次の表記スタイルを使用しています :
> 
> * エンティティタイプは**太字**です
> * データ属性は  `monospace text` に記述されています
> * 現実世界のアイテムはプレーン・テキストを使用します
>
> したがって、現実世界のストアは、**Store** エンティティによってコンテキスト・データ内に表され、ストア内にある現実世界の棚は、`refStore` 属性を有する **Shelf** エンティティによってコンテキスト・データに表されます。


<a name="architecture"></a>
# アーキテクチャ

このアプリケーションは、[Orion Context Broker](https://catalogue.fiware.org/enablers/publishsubscribe-context-broker-orion-context-broker) という1つの FIWARE コンポーネントしか使用しません。アプリケーションが *"Powered by FIWARE"* と認定するには、Orion Context Broker を使用するだけで十分です。

現在、Orion Context Broker はオープンソースの [MongoDB](https://www.mongodb.com/) 技術を利用して、コンテキスト・データの永続性を維持しています。したがって、アーキテクチャは2つの要素で構成されます :

* [NGSI](https://fiware.github.io/specifications/OpenAPI/ngsiv2) を使用してリクエストを受信する [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/)
* バックエンドの [MongoDB](https://www.mongodb.com/) データベース
  + Orion Context Broker が、データ・エンティティなどのコンテキスト・データ情報、サブスクリプション、登録などを保持するために使用します

2つの要素間のすべての対話は HTTP リクエストによって開始されるため、エンティティはコンテナ化され、公開されたポートから実行されます。

![](https://fiware.github.io/tutorials.Entity-Relationships/img/architecture.png)

必要な設定情報は、関連する `docker-compose.yml` ファイルの services セクションにあります:

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

両方のコンテナが同じネットワークに常駐しています。Orion Context Broker はポート `1026` でリッスンしており、MongoDB はデフォルト・ポート `27071` でリッスンしています。 どちらのコンテナも同じポートを外部に公開しています。これはチュートリアルのアクセス専用です。これにより、cUrl または Postman は同じネットワークに参加することなくアクセスできます。 コマンドラインの初期化は、一目瞭然でなければなりません。

<a name="prerequisites"></a>
# 前提条件

<a name="docker-and-docker-compose"></a>
## Docker と Docker Compose

物事を単純にするために、両方のコンポーネントは [Docker](https://www.docker.com) を使用して実行されます。**Docker** は、さまざまコンポーネントをそれぞれの環境に分離することを可能にするコンテナ・テクノロジです。

* Docker を Windows にインストールするには、[こちら](https://docs.docker.com/docker-for-windows/)の手順に従ってください
* Docker を Mac にインストールするには、[こちら](https://docs.docker.com/docker-for-mac/)の手順に従ってください
* Docker を Linux にインストールするには、[こちら](https://docs.docker.com/install/)の手順に従ってください

**Docker Compose** は、マルチコンテナ Docker アプリケーションを定義して実行するためのツールです。[YAML file](https://raw.githubusercontent.com/Fiware/tutorials.Getting-Started/master/docker-compose.yml) ファイルは、アプリケーションのために必要なサービスを設定する使用されています。つまり、すべてのコンテナ・サービスは1つのコマンドで呼び出すことができます。Docker Compose は、デフォルトで Docker for Windows とD ocker for Mac の一部としてインストールされますが、Linux ユーザは[ここ](https://docs.docker.com/compose/install/)に記載されている手順に従う必要があります。

<a name="cygwin-for-windows"></a>
## Cygwin for Windows

シンプルな Bash スクリプトを使用してサービスを開始します。Windows ユーザは [cygwin](http://www.cygwin.com/) をダウンロードして、Windows の Linux ディストリビューションに似たコマンドライン機能を提供する必要があります。


<a name="start-up"></a>
# 起動

リポジトリ内で Bash スクリプトが提供する [services](https://github.com/Fiware/tutorials.Entity-Relationships/blob/master/services) を実行することにより、コマンドラインからすべてのサービスを初期化することができます。リポジトリを複製し、以下のコマンドを実行して必要なイメージを作成してください :

```console
git clone git@github.com:Fiware/tutorials.Entity-Relationships.git
cd tutorials.Entity-Relationships

./services start
``` 

このコマンドは、起動時に以前の[ストア・ファインダのチュートリアル](https://github.com/Fiware/tutorials.Getting-Started)のシード・データもインポートします。

>**注** : クリーンアップをやり直したい場合は、次のコマンドを使用して再起動することができます :
>
```console
./services stop
``` 
>


<a name="creating-and-associating-data-entities"></a>
#  データ・エンティティの作成と関連付け

<a name="creating-several-entities-at-once"></a>
## 一度に複数のエンティティを作成

前のチュートリアルでは、各ストアのエンティティを個別に作成しました。

同時に5つの棚ユニットを作成できます。このリクエストでは、簡易バッチ処理エンドポイントを使用して5つの棚エンティティを作成します。バッチ処理では、2つの属性を持つペイロードで `/v2/op/update` エンドポイントを使用します。`actionType=APPEND` は、既存のエンティティが存在する場合はそれを上書きすることを意味しますが、`entities` 属性は更新するエンティティの配列を保持します。

**Store** エンティティから **Shelf** エンティティを区別するために、各棚が `type=Shelf` に割り当てられています。'name' と 'location' のような実世界のプロパティが各棚にプロパティとして追加されました。

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

どちらの場合も、NGSI-LD [ドラフト勧告](https://docbox.etsi.org/ISG/CIM/Open/ISG_CIM_NGSI-LD_API_Draft_for_public_review.pdf)に従って各エンティティ `id` をエンコードしました。提案は、それぞれの `id` が 標準フォーマットに従った URN というものです : `urn:ngsi-ld:<entity-type>:<entity-id>`。これは、システム内のすべての `id` がユニークであることを意味します。

棚情報は、`/v2/entities` エンドポイントで GET リクエストを行うことでリクエストできます。たとえば、`id=urn:ngsi-ld:Shelf:unit001` を使用すると、その **Shelf** エンティティのコンテキスト・データが得られます :

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
        "coordinates": [
            13.3986112,
            52.554699
        ]
    },
    "maxCapacity": 50,
    "name": "Corner Unit"
}
```

このように、存在する3つの追加のプロパティ属性 `location`, `maxCapacity` と `name` があります。


<a name="creating-a-one-to-many-relationship"></a>
## 1対多のリレーションシップの作成

データベースでは、外部キーは1対多のリレーションシップを指定するためによく使用されます。たとえば、すべての棚が1つのストアにあり、1つのストアには多くの棚ユニットがあります。この情報を記憶するためには、外部キーと同様のアソシエーション・リレーションシップを追加する必要があります。バッチ処理を再び使用して、既存の **Shelf** エンティティを修正して、各ストアにリレーションシップを保持する `refStore` 属性を追加することができます。[リンクト・データ](http://fiware-datamodels.readthedocs.io/en/latest/guidelines/index.html#modelling-linked-data) に関するFIWARE データ・モデリング・ガイドラインによると、エンティティ属性が他のエンティティへのリンクとして使用される場合、プレフィックス `ref` と ターゲットのリンクト・エンティティ・タイプの名前を付けて名前を付ける必要があります。

`refStore` 属性値は、**Store** エンティティ自体に関連付けられた URN に対応します。

URN は標準フォーマットに従います : `urn:ngsi-ld:<entity-type>:<entity-id>`

#### :four: リクエスト :

次のリクエストは、3つの棚を `urn:ngsi-ld:Store:001` に、2つの棚を `urn:ngsi-ld:Store:002` に関連付けます。

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

ここで棚情報が再度リクエストされると、レスポンスが変更され、前の手順で追加された新しい `refStore` 属性が含まれます。

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
        "coordinates": [
            13.3986112,
            52.554699
        ]
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

また、`options=values` 設定を使用して、既知の棚エンティティから `refStore` 属性のリレーションシップ情報を取得するようリクエストすることもできます :

#### :six: リクエスト :

```console
curl -X GET \
  'http://localhost:1026/v2/entities/urn:ngsi-ld:Shelf:unit001/?type=Shelf&options=values&attrs=refStore'
```

#### レスポンス :

```json
[
    "urn:ngsi-ld:Store:001"
]
```

これは、"私は、`id=urn:ngsi-ld:Store:001` で **Store**エンティティと関連しています"と、解釈することができます。

<a name="reading-from-parent-entity-to-child-entity"></a>
### 親エンティティから子エンティティへの読み取り

親から子への読み込みは、`options=count` 設定を使用して行うことができます :

#### :seven: リクエスト :

```console
curl -X GET \
  'http://localhost:1026/v2/entities/?q=refStore==urn:ngsi-ld:Store:001&options=count&attrs=type&type=Shelf' 
```

このリクエストは、URN `urn:ngsi-ld:Store:001` に関連付けられているすべての **Shelf** エンティティの `id` をリクエストしています。レスポンスは、次に示されているように JSON 配列です。

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

普通の英語では、これは"`urn:ngsi-ld:Store:001` の中に3つの棚があります"と解釈することができます。リクエストを変更して、`options=values` と `attrs` パラメータを使用して、関連する関連エンティティの特定のプロパティを返すことができます。例えば、次のようなリクエストです : 

#### :eight: リクエスト :

```console
curl -X GET \
  'http://localhost:1026/v2/entities/?q=refStore==urn:ngsi-ld:Store:001&type=Shelf&options=values&attrs=name'
```

*"`urn:ngsi-ld:Store:001` にあるすべての棚の名前を教えてください。"* というリクエストとして解釈することができます。

#### レスポンス :

```json
[
    [
        "Corner Unit"
    ],
    [
        "Wall Unit 1"
    ],
    [
        "Wall Unit 2"
    ]
]
```


<a name="creating-many-to-many-relationships"></a>
## 多対多のリレーションシップの作成

ブリッジ・テーブルは、多対多のリレーションシップを関連付けるためによく使用されます。たとえば、各ストアではさまざまな種類の商品が販売され、各商品はさまざまなストアで販売されます。

ブリッジ・テーブルは、多対多のリレーションシップを関連付けるためによく使用されます。たとえば、各ストアではさまざまな種類の商品が販売され、各商品はさまざまなストアで販売されます。

コンテキスト情報を"所与のストアの棚に製品を置く"ように保持するためには、他のエンティティからのデータを関連付けるために存在する新しいデータ・エンティティ **InventoryItem** を作成する必要があります。それは、**Store**, **Shelf** と **Product** エンティティとは外部キー関係を持つので、`refStore`、`refShelf`、および `refProduct` というリレーションシップ属性が必要です。

製品を棚に割り当てることは、単にリレーションシップ情報と、`stockCount` と `shelfCount` のようなその他の追加プロパティを保持するエンティティを作成することによって行われます。

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

ブリッジ・テーブル・エンティティから読み込む場合、エンティティの `type` を知る必要があります。

少なくとも1つの **InventoryItem** エンティティを作成した後、次のリクエストを行うことで、*"`urn:ngsi-ld:Store:001` の中にどの製品が販売されているか？"* をクエリすることができます。

#### :one::zero: リクエスト :

```console
curl -X GET \
  'http://localhost:1026/v2/entities/?q=refProduct==urn:ngsi-ld:Product:001&options=values&attrs=refStore&type=InventoryItem' 
```

#### レスポンス :

```json
[
    [
        "urn:ngsi-ld:Store:001"
    ]
]
```


同様に、次のようにリクエストを変更することで、どのストアで `urn:ngsi-ld:Product:001` が売れているのかを知ることができます :

#### :one::one: リクエスト :

```console
curl -X GET \
  'http://localhost:1026/v2/entities/?q=refStore==urn:ngsi-ld:Store:001&options=values&attrs=refProduct&type=InventoryItem' 
```

#### レスポンス :

```json
[
    [
        "urn:ngsi-ld:Product:prod001"
    ]
]
```



<a name="data-integrity"></a>
## データの整合性

コンテキスト・データのリレーションシップは、存在するエンティティ間でのみ設定および維持する必要があります。つまり、URN `urn:ngsi-ld:<entity-type>:<entity-id>` はコンテキスト内の別の既存エンティティにリンクする必要があります。したがって、ダングリング・リファレンスが残っていないエンティティを削除するときは、注意が必要です。`urn:ngsi-ld:Store:001` が削除されたと想像してください。関連する **Shelf** のエンティティに何が起こるべきですか？

以下のようにリクエストすることにより、削除前に残っているエンティティのリレーションシップが存在するかどうかを確認するリクエストを出すことができます :

#### :one::two: リクエスト :

```console
curl -X GET \
  'http://localhost:1026/v2/entities/?q=refStore==urn:ngsi-ld:Store:001&options=count&attrs=type'
```


#### :one::three: リクエスト :

このレスポンスには一連の **Shelf** と **InventoryItem** エンティティがリストされています。製品とストアの間に直接のリレーションシップがないため、**Product** エンティティはありません。

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

高度な機能を追加することで、アプリケーションに複雑さを加える方法を知りたいですか？このシリーズの他のチュートリアルを読むことで見つけることができます :


