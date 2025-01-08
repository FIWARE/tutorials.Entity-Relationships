# Linked Data Entity Relationships[<img src="https://img.shields.io/badge/NGSI-LD-d6604d.svg" width="90"  align="left" />](https://www.etsi.org/deliver/etsi_gs/CIM/001_099/009/01.08.01_60/gs_cim009v010801p.pdf)[<img src="https://fiware.github.io/tutorials.Entity-Relationships/img/fiware.png" align="left" width="162">](https://www.fiware.org/)<br/>

[![FIWARE Core Context Management](https://nexus.lab.fiware.org/repository/raw/public/badges/chapters/core.svg)](https://github.com/FIWARE/catalogue/blob/master/core/README.md)
[![License: MIT](https://img.shields.io/github/license/fiware/tutorials.CRUD-Operations.svg)](https://opensource.org/licenses/MIT)
[![Support badge](https://img.shields.io/badge/tag-fiware-orange.svg?logo=stackoverflow)](https://stackoverflow.com/questions/tagged/fiware)
[![JSON LD](https://img.shields.io/badge/JSON--LD-1.1-f06f38.svg)](https://w3c.github.io/json-ld-syntax/) <br/>
[![Documentation](https://img.shields.io/readthedocs/ngsi-ld-tutorials.svg)](https://ngsi-ld-tutorials.rtfd.io)

このチュートリアルでは、**NGSI-LD** ユーザにバッチコマンドとエンティティのリレーションシップについて説明します。
このチュートリアルは、以前の[スマート・ファームの例](https://github.com/FIWARE/tutorials.Getting-Started/tree/NGSI-LD)
で作成されたデータに基づいて構築され、一連の関連データ・エンティティを作成および関連付けて、
追加センサとファーム・ワーカをファームに作成します。

チュートリアルでは全体で [cUrl](https://ec.haxx.se/) コマンドを使用しますが、
[Postman documentation](https://fiware.github.io/tutorials.Entity-Relationships/ngsi-ld.html) としても利用できます。

[![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/d0f2b74c4beb8434595f)
[![Open in Gitpod](https://gitpod.io/button/open-in-gitpod.svg)](https://gitpod.io/#https://github.com/FIWARE/tutorials.Entity-Relationships/tree/NGSI-LD)

## コンテンツ

<details>
<summary><strong>詳細</strong></summary>

-   [エンティティとリレーションシップを理解](#understanding-entities-and-relationships)
    -   [ファーム管理情報システム (Farm Management Information System; FMIS) 内のエンティティ](#entities-within-a-farm-management-information-system-fmis)
-   [アーキテクチャ](#architecture)
-   [前提条件](#prerequisites)
    -   [Docker と Docker Compose](#docker-and-docker-compose)
    -   [WSL](#wsl)
-   [起動](#start-up)
-   [データ・エンティティの作成と関連付け](#creating-and-associating-data-entities)
    -   [一度に複数のエンティティを作成](#creating-several-entities-at-once)
    -   [1対1または1対多のリレーションシップの作成](#creating-one-to-one-or-one-to-many-relationships)
    -   [外部キー・リレーションシップ (Foreign Key Relationship) の読み取り](#reading-a-foreign-key-relationship)
        -   [子エンティティから親エンティティへの読み取り](#reading-from-child-entity-to-parent-entity)
        -   [親エンティティから子エンティティへの読み取り](#reading-from-parent-entity-to-child-entity)
    -   [多対多のリレーションシップの作成](#creating-many-to-many-relationships)
    -   [ブリッジ・テーブルからの読み取り](#reading-from-a-bridge-table)
    -   [プロパティのリレーションシップ](#relationships-of-properties)
        -   [納屋の温度を取得](#retriving-the-temperature-of-a-barn)
-   [次のステップ](#next-steps)

</details>

<a name="understanding-entities-and-relationships"/>

# エンティティとリレーションシップを理解

FIWARE プラットフォーム内では、エンティティのコンテキストは、実世界に存在する物理的または概念的なオブジェクトの
状態を表します。

<a name="entities-within-a-farm-management-information-system-fmis"/>

## ファーム管理情報システム (Farm Management Information System; FMIS) 内のエンティティ

NGSI-LD に基づく FMIS システム内のエンティティのリレーションシップを説明するために、一連のエンティティを作成する必要が
あります。この簡略化された FMIS の場合、必要なエンティティは少数です。エンティティ間のリレーションシップは、
次のように定義されています:

![](https://fiware.github.io/tutorials.Entity-Relationships/img/ngsi-ld-entities.png)

-   納屋などの建物 (building) は、現実世界のレンガとモルタルの構造物です。**Building** エンティティには、
    次のようなプロパティがあります:
    -   建物の名前。例: "大きな赤い納屋"
    -   建物のカテゴリ。例: "納屋"
    -   住所 "Friedrichstraße 44, 10969 Kreuzberg, Berlin"
    -   物理的な場所 (例: _52.5075 N, 13.3903 E_)
    -   充填レベル (filling level) - 建物が満杯である程度
    -   温度 - 例: _21 °C_
    -   建物の所有者 (実在の人物) との関連 (association)
-   **TemperatureSensors** や **FillingLevelSensors** などのスマート・デバイスは、一般的な **Device** データモデルを
    拡張します。各 **Device** エンティティには、次のようなプロパティがあります:
    -   デバイスの説明
    -   デバイスのカテゴリ (例: _sensor_, _actuator_, _both_)
    -   デバイスが測定しているプロパティの名前 (例: _temperature_)
    -   デバイスが測定している資産 (asset) (例: building)との関連 (association)
-   **person** は、農家や農業労働者を代表するエンティティです。各 **Person** エンティティには、次のようなプロパティが
    あります:
    -   その人の名前。例: "Mr. Jones"
    -   役職
    -   彼らが所有する農場の建物との関連 (association)
-   私たちが農場でやっていることのタスク。これは概念的なエンティティであり、労働者、農産物、場所を関連付けるために
    使用されます。**Task** エンティティには次のようなプロパティがあります。
    -   タスクの名前 (例: _Spray Fertilizer XXX on field Y_)
    -   タスクのステータス (例: _scheduled_, _in progress_, _completed_)
    -   タスクを実行するワーカー (つまり、**Person** エンティティ) への関連付け
    -   使用する製品 (例: 肥料 **Fertilizer** エンティティ) への関連付け
    -   使用する場所 (例:  **PartField** エンティティなど) への関連付け

ご覧のとおり、上記で定義された各エンティティには、静的データと動的データが混在しています。一部のプロパティは変更される
可能性があります。魚の加水分解物などの有機肥料 **Fertilizer** は、その `formula` (配合) を変更したり、
干し草を販売したり、納屋の `fillingLevel` (充填レベル) を減らしたりすることができます。

> **注** このチュートリアルでは、次のタイポグラフィ・スタイルを使用しています :
>
> -   エンティティ・タイプは、**bold text** (太字) になっています
> -   データ属性は `monospace text` (モノスペース・テキスト) で書かれています
> -   実世界のアイテムはプレーン・テキストを使用します
>
> したがって、実世界の人物は、コンテキスト・データでは **Person** エンティティによって表され、人物が所有する実世界の
> 納屋 (barn) は、`owner` 属性を持つ **Building** エンティティによってコンテキスト・データで表されます。

<a name="architecture"/>

# アーキテクチャ

デモ FMIS アプリケーションは、Context Broker に準拠した NGSI-LD 呼び出しを送受信します。標準化された NGSI-LD
インターフェースは複数の Context Broker で利用できるため、1つだけ選択する必要があります。たとえば、
[Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/) です。
したがって、アプリケーションは1つの FIWARE コンポーネントのみを使用します。

現在、Orion Context Broker は、保持しているコンテキスト・データの永続性を維持するためにオープンソースの
[MongoDB](https://www.mongodb.com/) テクノロジに依存しています。

データ交換の相互運用性を促進するために、NGSI-LD context brokers は
[JSON-LD `@context` ファイル](https://json-ld.org/spec/latest/json-ld/#the-context)を明示的に公開して、コンテキスト・
エンティティ内に保持されるデータを定義します。これにより、すべてのエンティティ・タイプとすべての属性に一意の URI
が定義され、NGSI ドメイン外の他のサービスがデータ構造の名前を選択できるようになります。すべての @context ファイル
がネットワーク上で利用可能である必要があります。このチュートリアルでは、チュートリアル・アプリケーションを使用して、
一連の静的ファイルをホストします。

したがって、アーキテクチャは次の3つの要素で構成されます:

-   [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/) は、
    [NGSI-LD](https://forge.etsi.org/swagger/ui/?url=https://forge.etsi.org/rep/NGSI-LD/NGSI-LD/raw/master/spec/updated/generated/full_api.json)
    を使用してリクエストを受信します
-   基礎となる [MongoDB](https://www.mongodb.com/) データベース:
    -   データ・エンティティ、サブスクリプション、レジストレーションなどのコンテキスト・データ情報を保持するために
        Orion Context Broker によって使用されます
-   **チュートリアル・アプリケーション**は次のことを行います:

2つの要素間のすべての対話は HTTP リクエストによって開始されるため、要素をコンテナ化して、公開されたポートから
実行できます。

![](https://fiware.github.io/tutorials.Entity-Relationships/img/architecture-ld.png)

必要な構成情報は、関連する `docker-compose.yml` ファイルの services セクションにあります。
[以前のチュートリアル](https://github.com/FIWARE/tutorials.Getting-Started/tree/NGSI-LD)で説明しています。

<a name="prerequisites"/>

# 前提条件

<a name="docker-and-docker-compose"/>

## Docker と Docker Compose

シンプルにするために、すべてのコンポーネントは [Docker](https://www.docker.com) を使用して実行されます。 **Docker**
は、それぞれの環境に分離されたさまざまなコンポーネントを可能にするコンテナ・テクノロジです。

-   Windows に Docker をインストールするには、[こちら](https://docs.docker.com/docker-for-windows/)の指示に従ってください
-   Mac に Docker をインストールするには、[こちら](https://docs.docker.com/docker-for-mac/)の指示に従ってください
-   Linux に Docker をインストールするには、[こちら](https://docs.docker.com/install/)の手順に従ってください

**Docker Compose** は、マルチコンテナ Docker アプリケーションを定義して実行するためのツールです。
[YAMLファイル](https://raw.githubusercontent.com/Fiware/tutorials.Entity-Relationships/master/docker-compose.yml)
を使用して、アプリケーションに必要なサービスを設定します。これは、すべてのコンテナ・サービスを単一のコマンドで起動
できることを意味します。Docker Compose は、Docker for Windows および Docker for Mac の一部としてデフォルトでインストール
されますが、Linux ユーザは[こちら](https://docs.docker.com/compose/install/)にある手順に従う必要があります。

次のコマンドを使用して、現在の **Docker** および **DockerCompose** のバージョンを確認できます:

```console
docker-compose -v
docker version
```

Docker version 24.0.x 以降および Docker Compose 2.24.x 以降を使用していることを確認し、
必要に応じてアップグレードしてください。

## WSL

簡単な bash スクリプトを使ってサービスを開始します。Windows ユーザは、Windows 上の Linux ディストリビューションに
似たコマンドライン機能を提供するために [を使用して Windows に Linux をインストールする方法](https://learn.microsoft.com/ja-jp/windows/wsl/install) をダウンロードするべきです。

<a name="start-up"/>

# 起動

すべてのサービスは、リポジトリ内で提供される
[services](https://github.com/FIWARE/tutorials.Entity-Relationships/blob/NGSI-LD/services) Bash スクリプトを実行して、
コマンドラインから初期化できます。以下のようにコマンドを実行して、リポジトリのクローンを作成して必要なイメージを
作成してください :

```console
git clone https://github.com/FIWARE/tutorials.Entity-Relationships.git
cd tutorials.Entity-Relationships
git checkout NGSI-LD

./services [orion|scorpio|stellio]
```

このコマンドは、起動時にシード・データ (**Building**, **Person**, **TemperatureSensor**, **FillingLevelSensor**,
**Herbicide**, **PartField**) をインポートします。

> :information_source: **注:** クリーンアップして最初からやり直す場合は、次のコマンドで実行できます:
>
> ```console
> ./services stop
> ```

<a name="creating-and-associating-data-entities"/>

# データ・エンティティの作成と関連付け

<a name="creating-several-entities-at-once"/>

## 一度に複数のエンティティを作成

以前のチュートリアルでは、各エンティティを個別に作成しました。

同時に複数のセンサを作成しましょう。このリクエストは、コンビニエンス・バッチ処理エンドポイントを使用して5つの
エンティティを作成します。バッチ処理では `/ngsi-ld/v1/entityOperations/` エンドポイントが使用されます。
`upsert` エンドポイントは、エンティティ存在しない場合は新しいエンティティを作成し、存在する場合は既存の
エンティティを上書きすることを意味します。

さまざまなデバイスを区別するために、各温度センサには `type=TemperatureSensor` が割り当てられています。`category`
などの実際のプロパティは、各デバイスにプロパティとして追加されています。

#### 1️⃣ リクエスト:

```console
curl -X POST 'http://localhost:1026/ngsi-ld/v1/entityOperations/upsert' \
-H 'Content-Type: application/json' \
-H 'Link: <http://context/user-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"' \
-H 'Accept: application/ld+json' \
--data-raw '[
    {
      "id": "urn:ngsi-ld:TemperatureSensor:001",
      "type": "TemperatureSensor",
      "description" : {"type": "Property", "value": "Temperature Gauge 1"},
      "category": {"type": "Property", "value": "sensor"},
      "controlledProperty" : {"type": "Property", "value": "temperature"},
      "temperature": {"type": "Property", "value": 20, "unitCode": "CEL"}
    },
    {
      "id": "urn:ngsi-ld:TemperatureSensor:002",
      "type": "TemperatureSensor",
      "description" : {"type": "Property", "value": "Temperature Gauge 2"},
      "category": {"type": "Property", "value": "sensor"},
      "controlledProperty" : {"type": "Property", "value": "temperature"},
      "temperature": {"type": "Property", "value": 21, "unitCode": "CEL"}
    },
    {
      "id": "urn:ngsi-ld:TemperatureSensor:003",
      "type": "TemperatureSensor",
      "description" : {"type": "Property", "value": "Temperature Gauge 3"},
      "category": {"type": "Property", "value": "sensor"},
      "controlledProperty" : {"type": "Property", "value": "temperature"},
      "temperature": {"type": "Property", "value": 27, "unitCode": "CEL"}
    }
]'
```

同様に、`type=FillingLevelSensor` を使用して一連の **FillingLevelSensors** エンティティを作成できます。

#### 2️⃣ リクエスト:

```console
curl -iX POST 'http://localhost:1026/ngsi-ld/v1/entityOperations/upsert' \
-H 'Link: <http://context/user-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"' \
-H 'Content-Type: application/json' \
--data-raw '[
    {
      "id": "urn:ngsi-ld:FillingLevelSensor:001",
      "type": "FillingLevelSensor",
      "description" : {"type": "Property", "value": "Filling Level Sensor 1"},
      "category": {"type": "Property", "value": "sensor"},
      "controlledProperty" : {"type": "Property", "value": "fillingLevel"},
      "fillingLevel": {"type": "Property", "value": 1, "unitCode": "C62"}
    },
    {
      "id": "urn:ngsi-ld:FillingLevelSensor:002",
      "type": "FillingLevelSensor",
      "description" : {"type": "Property", "value": "Filling Level Sensor 2"},
      "category": {"type": "Property", "value": "sensor"},
      "controlledProperty" : {"type": "Property", "value": "fillingLevel"},
      "fillingLevel": {"type": "Property", "value": 0.9, "unitCode": "C62"}
    },
    {
      "id": "urn:ngsi-ld:FillingLevelSensor:003",
      "type": "FillingLevelSensor",
      "description" : {"type": "Property", "value": "Filling Gauge 3"},
      "category": {"type": "Property", "value": "sensor"},
      "controlledProperty" : {"type": "Property", "value": "fillingLevel"},
      "fillingLevel": {"type": "Property", "value": 0.8, "unitCode": "C62"}
    }
]'
```

どちらの場合も、NGSI-LD [仕様](https://www.etsi.org/deliver/etsi_gs/CIM/001_099/009/01.08.01_60/gs_cim009v010801p.pdf)
に従って各エンティティ `id` をエンコードしました - プロポーザルは、各 `id` が標準形式に従う URN であるというものです:
`urn:ngsi-ld:<entity-type>:<entity-id>`。 これは、システム内のすべての `id` が一意になることを意味します。

デバイス情報は、`/ngsi-ld/v1/entities` エンドポイントで GET リクエストを行うことでリクエストできます。たとえば、
デバイスのコンテキスト・データを返すためです。

#### 3️⃣ リクエスト:

```console
curl -X GET 'http://localhost:1026/ngsi-ld/v1/entities/?type=TemperatureSensor,FillingLevelSensor&options=keyValues' \
-H 'Link: <http://context/user-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"'
```

#### レスポンス:

```json
[
    {
        "id": "urn:ngsi-ld:TemperatureSensor:001",
        "type": "TemperatureSensor",
        "description": "Temperature Gauge 1",
        "category": "sensor",
        "controlledProperty": "temperature",
        "temperature": 20
    },
    {
        "id": "urn:ngsi-ld:TemperatureSensor:002",
        "type": "TemperatureSensor",
        "description": "Temperature Gauge 2",
        "category": "sensor",
        "controlledProperty": "temperature",
        "temperature": 21
    },
    ... etc
]
```

ご覧のように、現在、`description`, `category`, `controledProperty` の3つの追加プロパティ属性があります。

<a name="creating-one-to-one-or-one-to-many-relationships"/>

## 1対1または1対多のリレーションシップの作成

データベースでは、外部キー (foreign keys) は1対1または1対多のリレーションシップ (one-to-one or one-to-many relationship)
を指定するためによく使用されます。たとえば、1つの建物に多数のデバイスを収容できます。この情報を記憶するには、
外部キーと同様の関連付けリレーションシップ (association relationship) を追加する必要があります。バッチ処理を再度使用して、
既存の **TemperatureSensor** エンティティと **FillingLevelSensor** エンティティを修正し、デバイスによって制御される
各建物との1対1のリレーションシップを保持する `controlledAsset` 属性を追加できます。Smart Data Model によると、
[Device](https://swagger.lab.fiware.org/?url=https://smart-data-models.github.io/dataModel.Device/Device/swagger.yaml)
定義 `https://uri.fiware.org/ns/dataModels#controlledAsset` はこのリレーションシップに使用される URI の長い名前であり、
`controlledAsset` 属性の値は **Building** エンティティ自体に関連付けられた URN に対応します。

URN は、標準形式に従います: `urn:ngsi-ld:<entity-type>:<entity-id>`

#### 4️⃣ リクエスト:

次のリクエストは、6つのデバイスを `urn:ngsi-ld:Building:farm001`, `urn:ngsi-ld:Building:barn002` および
`urn:ngsi-ld:Building:farm002` に6つのデバイスを関連付けます。

```console
curl -G -iX POST 'http://localhost:1026/ngsi-ld/v1/entityOperations/upsert' \
-d 'options=update' \
-H 'Content-Type: application/json' \
-H 'Link: <http://context/user-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"' \
--data-raw '[
    {
        "id": "urn:ngsi-ld:TemperatureSensor:001",
        "type": "TemperatureSensor",
        "controlledAsset": {"type": "Relationship", "object": "urn:ngsi-ld:Building:farm001"}
    },
    {
        "id": "urn:ngsi-ld:TemperatureSensor:002",
        "type": "TemperatureSensor",
        "controlledAsset": {"type": "Relationship", "object": "urn:ngsi-ld:Building:barn002"}
    },
    {
        "id": "urn:ngsi-ld:FillingLevelSensor:003",
        "type": "FillingLevelSensor",
        "controlledAsset": {"type": "Relationship", "object": "urn:ngsi-ld:Building:farm002"}
    },
    {
        "id": "urn:ngsi-ld:FillingLevelSensor:001",
        "type": "FillingLevelSensor",
        "controlledAsset": {"type": "Relationship", "object": "urn:ngsi-ld:Building:farm001"}
    },
    {
        "id": "urn:ngsi-ld:FillingLevelSensor:002",
        "type": "FillingLevelSensor",
        "controlledAsset": {"type": "Relationship", "object": "urn:ngsi-ld:Building:barn002"}
    },
    {
        "id": "urn:ngsi-ld:TemperatureSensor:003",
        "type": "TemperatureSensor",
        "controlledAsset": {"type": "Relationship", "object": "urn:ngsi-ld:Building:farm002"}
    }
]'
```

これで、デバイス情報が再度リクエストされると、レスポンスが変更され、前の手順で追加された新しいプロパティ
`controlledAsset`が含まれます。

#### 5️⃣ リクエスト:

```console
curl -G -iX GET 'http://localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:TemperatureSensor001' \
-d 'options=keyValues' \
-H 'Link: <http://context/user-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"'
```

#### レスポンス:

`controlledAsset` 属性を含む更新されたレスポンスを以下に示します:

```json
{
    "@context": [
            "http://context/user-context.jsonld",
            "https://uri.etsi.org/ngsi-ld/v1/ngsi-ld-core-context-v1.8.jsonld"
        ],
    "id": "urn:ngsi-ld:TemperatureSensor:001",
    "type": "TemperatureSensor",
    "description": "Temperature Gauge 1",
    "category": "sensor",
    "controlledProperty": "temperature",
    "temperature": 20,
    "controlledAsset": "urn:ngsi-ld:Building:farm001"
}
```

<a name="reading-a-foreign-key-relationship"/>

## 外部キー・リレーションシップ (Foreign Key Relationship) の読み取り

<a name="reading-from-child-entity-to-parent-entity"/>

### 子エンティティから親エンティティへの読み取り

`options=keyValues` 設定を使用して、既知の **Device** エンティティから属性リレーションシップ情報を取得するように
リクエストすることもできます。

#### 6️⃣ リクエスト:

```console
curl -G -iX GET 'http://localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:TemperatureSensor:001' \
-d 'options=keyValues' \
-d 'attrs=controlledAsset' \
-H 'Link: <http://context/user-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"' \
-H 'Accept: application/json'
```

#### レスポンス:

```json
{
    "id": "urn:ngsi-ld:TemperatureSensor:001",
    "type": "TemperatureSensor",
    "controlledAsset": "urn:ngsi-ld:Building:farm001"
}
```

これは、"`id=urn:ngsi-ld:Building:farm001` を使用して **Building** エンティティ内でセンサの読み取りを行っている"
と解釈できます。

<a name="reading-from-parent-entity-to-child-entity"/>

### 親エンティティから子エンティティへの読み取り

親から子への読み取りは、次のクエリを使用して実行できます:

#### 7️⃣ リクエスト:

```console
curl -G -iX GET 'http://localhost:1026/ngsi-ld/v1/entities' \
-d 'q=controlledAsset==%22urn:ngsi-ld:Building:farm001%22' \
-d 'attrs=controlledAsset' \
-d 'options=keyValues' \
-H 'Link: <http://context/user-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"'
```

このリクエストは、URN `urn:ngsi-ld:Building:farm001` に関連付けられている、すべての **Device** エンティティの `id` を
要求しています。レスポンスは、次に示されているように JSON 配列です。

#### レスポンス:

```json
[
    {
        "id": "urn:ngsi-ld:TemperatureSensor:001",
        "type": "TemperatureSensor",
    },
    {
        "id": "urn:ngsi-ld:FillingLevelSensor:001",
        "type": "FillingLevelSensor",
    }
]
```

平易な英語では、これは "`urn:ngsi-ld:Building:farm001` に2つのデバイスがあります" と解釈できます。
リクエストは、`count=true` を使用して変更でき、基準を満たすエンティティの数を返します。

#### 8️⃣ リクエスト:

```console
curl -G -iX GET 'http://localhost:1026/ngsi-ld/v1/entities' \
-d 'q=controlledAsset==%22urn:ngsi-ld:Building:farm001%22' \
-d 'attrs=controlledAsset' \
-d 'options=keyValues' \
-d 'count=true' \
-d 'limit=0' \
-H 'Link: <http://context/user-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"'
```

影響を受けるエンティティの数を示す HTTP ヘッダをレスポンスの一部として返します:

#### レスポンス:

```text
NGSILD-Results-Count: 2
```

<a name="creating-many-to-many-relationships"/>

## 多対多のリレーションシップの作成

ブリッジ・テーブルは、多対多のリレーションシップ (many-to-many relationships) を関連付けるためによく使用されます。
たとえば、FMIS 内のすべてのスプレー作業では、農場労働者、適用する製品、および処理を適用する場所 (**PartField** と呼ばれる)
を関連付ける必要があります。

"除草剤 (herbicide) をフィールドにスプレーするように作業者に指示する" というコンテキスト情報を保持するには、他のエンティティ
からのデータを関連付けるために存在する新しいデータ・エンティティ **Task** を作成する必要があります。**Person**, **Herbicide**
および **PartField** エンティティとの外部キー・リレーションシップがあるため、`field`, `herbicide` および `worker`
と呼ばれるリレーションシップ属性が必要です。

タスクの割り当ては、リレーションシップ情報とその他の追加プロパティ (`description` および `status` など) を保持する
エンティティを作成するだけで実行できます。

#### 9️⃣ リクエスト:

```console
curl -L -X POST 'http://localhost:1026/ngsi-ld/v1/entities/' \
-H 'Content-Type: application/json' \
-H 'Link: <http://context/user-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"' \
--data-raw '{
    "id": "urn:ngsi-ld:Task:001",
    "type": "Task",
    "worker": {"type": "Relationship", "object": "urn:ngsi-ld:Person:001"},
    "field": {"type": "Relationship", "object": "urn:ngsi-ld:PartField:002"},
    "product": {"type": "Relationship", "object": "urn:ngsi-ld:Herbicide:001"},
    "description": {"type": "Property", "value": "Spray the North Field with Agent Orange"},
    "status": {"type": "Property", "value": "scheduled"},
    "dueDate": {"type": "Property", "value": "2021-07-16"}
}'
```

<a name="reading-from-a-bridge-table"/>

## ブリッジ・テーブルからの読み取り

ブリッジ・テーブル・エンティティから読み取るときは、エンティティの `type` がわかっている必要があります。

少なくとも1つの **Task** エンティティを作成した後、次のリクエストを行うことによって、
"フィールド `urn:ngsi-ld:PartField:002` でアクティビティが割り当てられているワーカーはどれですか？"
というクエリを実行できます。

#### 1️⃣0️⃣ リクエスト:

```console
curl -G -iX GET 'http://localhost:1026/ngsi-ld/v1/entities' \
-d 'q=field==%22urn:ngsi-ld:PartField:002%22' \
-d 'options=keyValues' \
-d 'attrs=worker' \
-d 'type=Task' \
-H 'Link: <http://context/user-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"' \
-H 'Accept: application/json'
```

#### レスポンス:

```json
[
    {
        "id": "urn:ngsi-ld:Task:001",
        "type": "Task",
        "worker": "urn:ngsi-ld:Person:001"
    }
]
```

同様に、次に示されているようにリクエストを変更することによって、"どのフィールドが `urn:ngsi-ld:Herbicide:001`
を使用して処理されるか？” をリクエストできます。

#### 1️⃣1️⃣ リクエスト:

```console
curl -G -iX GET 'http://localhost:1026/ngsi-ld/v1/entities' \
-d 'q=product==%22urn:ngsi-ld:Herbicide:001%22' \
-d 'options=keyValues' \
-d 'attrs=field' \
-d 'type=Task' \
-H 'Link: <http://context/user-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"' \
-H 'Accept: application/json'
```

#### レスポンス:

```json
[
    {
        "id": "urn:ngsi-ld:Task:001",
        "type": "Task",
        "field": "urn:ngsi-ld:PartField:002"
    }
]
```

<a name="relationships-of-properties"/>

## プロパティのリレーションシップ

_Properties-of-Properties_ および _Relationships of Properties_ はメタデータです。これらのようなメタデータ・
エンティティをコンテキスト・データに追加すると、ユーザはエンティティのリレーションシップのグラフをナビゲートし、
システムの状態についてさらに洞察を得ることができます。

<a name="retriving-the-temperature-of-a-barn"/>

### 納屋の温度を取得

温度センサからの温度測定値については、すでに説明しました。このデータを別のエンティティに複製する必要がある場合も
あります。たとえば、納屋 (barn) のセンサの温度測定値は、納屋自体の温度測定値でもあります。ダミーの読み取り値は
すでに `urn:ngsi-ld:Building:farm001` エンティティに追加されており、GET リクエストで取得できます:

#### 1️⃣2️⃣ リクエスト:

```console
curl -G -iX GET 'http://localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:Building:farm001' \
-d 'attrs=temperature' \
-H 'Link: <http://context/user-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"' \
-H 'Accept: application/json'
```

#### レスポンス:

```json
{
    "id": "urn:ngsi-ld:Building:farm001",
    "type": "Building",
    "temperature": {
        "type": "Property",
        "value": 20,
        "unitCode": "CEL",
        "providedBy": {
            "type": "Relationship",
            "object": "urn:ngsi-ld:TemperatureSensor:001"
        }
    }
}
```

ご覧のように、`temperature` プロパティは、測定のプロバイダに関する追加情報を保持します。これにより、追加の推論を
行うことができます。

# 次のステップ

高度な機能を追加することで、アプリケーションに複雑さを加える方法を知りたいですか？ このシリーズの
[他のチュートリアル](https://www.letsfiware.jp/ngsi-ld-tutorials)を読むことで見つけることができます

---

## License

[MIT](LICENSE) © 2020-2025 FIWARE Foundation e.V.
