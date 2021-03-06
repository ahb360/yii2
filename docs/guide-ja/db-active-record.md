アクティブレコード
==================

> Note|注意: この節はまだ執筆中です。

[アクティブレコード](http://ja.wikipedia.org/wiki/Active_Record) は、データベースに保存されているデータにアクセスするために、オブジェクト指向のインタフェイスを提供するものです。
アクティブレコードクラスはデータベーステーブルと関連付けられます。
アクティブレコードのインスタンスはそのテーブルの行に対応し、アクティブレコードのインスタンスの属性がその行のカラムの値を表現します。
データベーステーブルに保存されたデータにアクセスしたり、データを操作したりするために、生の SQL 文を書くのではなく、アクティブレコードの属性にアクセスしたり、アクティブレコードのメソッドを使ったりするのです。

例えば、`Customer` が `customer` テーブルに関連付けられたアクティブレコードクラスであり、`name` が `customer` テーブルのカラムであると仮定しましょう。
`customer` テーブルに新しい行を挿入するために次のコードを書くことが出来ます。

```php
$customer = new Customer();
$customer->name = 'Qiang';
$customer->save();
```

上記のコードは、MySQL では、次のように生の SQL 文を使うのと等価なものです。
しかし、生の SQL 文の方は、直感的でなく、間違いも生じやすく、また、別の種類のデータベースを使う場合には、互換性の問題も生じ得ます。

```php
$db->createCommand('INSERT INTO `customer` (`name`) VALUES (:name)', [
    ':name' => 'Qiang',
])->execute();
```

Yii は次のリレーショナルデータベースに対して、アクティブレコードのサポートを提供しています。

* MySQL 4.1 以降: [[yii\db\ActiveRecord]] による。
* PostgreSQL 7.3 以降: [[yii\db\ActiveRecord]] による。
* SQLite 2 および 3: [[yii\db\ActiveRecord]] による。
* Microsoft SQL Server 2008 以降: [[yii\db\ActiveRecord]] による。
* Oracle: [[yii\db\ActiveRecord]] による。
* CUBRID 9.3 以降: [[yii\db\ActiveRecord]] による。(cubrid PDO 拡張の [バグ](http://jira.cubrid.org/browse/APIS-658)
  のために、値を引用符で囲む機能が動作しません。そのため、サーバだけでなくクライアントも CUBRID 9.3 が必要になります)
* Sphinx: [[yii\sphinx\ActiveRecord]] による。`yii2-sphinx` エクステンションが必要。
* ElasticSearch: [[yii\elasticsearch\ActiveRecord]] による。`yii2-elasticsearch` エクステンションが必要。

これらに加えて、Yii は次の NoSQL データベースに対しても、アクティブレコードの使用をサポートしています。

* Redis 2.6.12 以降: [[yii\redis\ActiveRecord]] による。`yii2-redis` エクステンションが必要。
* MongoDB 1.3.0 以降: [[yii\mongodb\ActiveRecord]] による。`yii2-mongodb` エクステンションが必要。

このチュートリアルでは、主としてリレーショナルデータベースのためのアクティブレコードの使用方法を説明します。
しかし、ここで説明するほとんどの内容は NoSQL データベースのためのアクティブレコードにも適用することが出来るものです。


アクティブレコードクラスを宣言する
----------------------------------

まずは、[[yii\db\ActiveRecord]] を拡張してアクティブレコードクラスを宣言するところから始めましょう。
すべてのアクティブレコードクラスはデータベーステーブルと関連付けられますので、クラスの中で [[yii\db\ActiveRecord::tableName()|tableName()]] メソッドをオーバーライドして、どのテーブルが関連付けられるかを指定しなければなりません。

次の例では、`customer` というデータベーステーブルのための `Customer` という名前のアクティブレコードクラスを宣言しています。

```php
namespace app\models;

use yii\db\ActiveRecord;

class Customer extends ActiveRecord
{
    const STATUS_INACTIVE = 0;
    const STATUS_ACTIVE = 1;
    
    /**
     * @return string このアクティブレコードクラスと関連付けられるテーブルの名前
     */
    public static function tableName()
    {
        return 'customer';
    }
}
```

アクティブレコードのインスタンスは [モデル](structure-models.md) であると見なされます。
この理由により、私たちは通常 `app\models` 名前空間 (あるいはモデルクラスを保管するための他の名前空間) の下にアクティブレコードクラスを置きます。

[[yii\db\ActiveRecord]] は [[yii\base\Model]] から拡張していますので、属性、検証規則、データのシリアライゼーションなど、[モデル](structure-models.md) が持つ *全ての* 機能を継承しています。


## データベースに接続する <span id="db-connection"></span>

デフォルトでは、アクティブレコードは、`db` [アプリケーションコンポーネント](structure-application-components.md) を [[yii\db\Connection|DB 接続]] として使用して、データベースのデータにアクセスしたり操作したりします。
[データベースアクセスオブジェクト](db-dao.md) で説明したように、次のようにして、アプリケーションの構成情報ファイルの中で `db` コンポーネントを構成することが出来ます。

```php
return [
    'components' => [
        'db' => [
            'class' => 'yii\db\Connection',
            'dsn' => 'mysql:host=localhost;dbname=testdb',
            'username' => 'demo',
            'password' => 'demo',
        ],
    ],
];
```

アクティブレコードクラスのために `db` とは異なるデータベース接続を使いたい場合は、[[yii\db\ActiveRecord::getDb()|getDb()]] メソッドをオーバーライドしなければなりません。

```php
class Customer extends ActiveRecord
{
    // ...

    public static function getDb()
    {
        // "db2" アプリケーションコンポーネントを使用
        return \Yii::$app->db2;
    }
}
```

## データをクエリする <span id="querying-data"></span>

アクティブレコードクラスを宣言した後、それを使って対応するデータベーステーブルからデータをクエリすることが出来ます。
このプロセスは通常次の三つのステップを踏みます。

1. [[yii\db\ActiveRecord::find()]] メソッドを呼んで、新しいクエリオブジェクトを作成する。
2. [クエリ構築メソッド](db-query-builder.md#building-queries) を呼んで、クエリオブジェクトを構築する。
3. [クエリメソッド](db-query-builder.md#query-methods) を呼んで、アクティブレコードのインスタンスの形でデータを取得する。

ご覧のように、このプロセスは [クエリビルダ](db-query-builder.md) による手続きと非常によく似ています。
唯一の違いは、`new` 演算子を使ってクエリオブジェクトを生成する代りに、[[yii\db\ActiveRecord::find()]] を呼んで  [[yii\db\ActiveQuery]] クラスであるクエリオブジェクトを返すという点です。

以下の例は、アクティブクエリを使ってデータをクエリする方法を示すものです。

```php
// ID が 123 である一人の顧客を返す
// SELECT * FROM `customer` WHERE `id` = 123
$customer = Customer::find()
    ->where(['id' => 123])
    ->one();

// アクティブな全ての顧客を返して、ID によって並べる
// SELECT * FROM `customer` WHERE `status` = 1 ORDER BY `id`
$customers = Customer::find()
    ->where(['status' => Customer::STATUS_ACTIVE])
    ->orderBy('id')
    ->all();

// アクティブな顧客の数を返す
// SELECT COUNT(*) FROM `customer` WHERE `status` = 1
$count = Customer::find()
    ->where(['status' => Customer::STATUS_ACTIVE])
    ->count();

// アクティブな全ての顧客を顧客IDによってインデックスされた配列として返す
// SELECT * FROM `customer`
$customers = Customer::find()
    ->indexBy('id')
    ->all();
```

上記において、`$customer` は `Customer` オブジェクトであり、`$customers` は `Customer` オブジェクトの配列です。
これらは全て `customer` テーブルから取得されたデータを投入されます。

> Info|情報: [[yii\db\ActiveQuery]] は [[yii\db\Query]] から拡張しているため、[クエリビルダ](db-query-builder.md) の節で説明されたクエリ構築メソッドとクエリメソッドの *全て* を使うことが出来ます。

プライマリキーの値や一群のカラムの値でクエリをすることはよく行われる仕事ですので、Yii はこの目的のために、二つのショートカットメソッドを提供しています。

- [[yii\db\ActiveRecord::findOne()]]: クエリ結果の最初の行を一つのアクティブレコードインスタンスに投入して返す。
- [[yii\db\ActiveRecord::findAll()]]: *全ての* クエリ結果をアクティブレコードインスタンスの配列に投入して返す。

どちらのメソッドも、次のパラメータ形式のどれかを取ることが出来ます。

- スカラ値: 値は検索時に求められるプライマリキーの値として扱われます。
  Yii は、データベースのスキーマ情報を読んで、どのカラムがプライマリキーのカラムであるかを自動的に判断します。
- スカラ値の配列: 配列は検索時に求められるプライマリキーの値の配列として扱われます。
- 連想配列: キーはカラム名であり、値は検索時に求められる対応するカラムの値です。
  詳細については、[ハッシュ形式](db-query-builder.md#hash-format) を参照してください。

次のコードは、これらのメソッドの使用方法を示すものです。

```php
// ID が 123 である一人の顧客を返す
// SELECT * FROM `customer` WHERE `id` = 123
$customer = Customer::findOne(123);

// ID が 100, 101, 123, 124 のどれかである顧客を全て返す
// SELECT * FROM `customer` WHERE `id` IN (100, 101, 123, 124)
$customers = Customer::findAll([100, 101, 123, 124]);

// ID が 123 であるアクティブな顧客を返す
// SELECT * FROM `customer` WHERE `id` = 123 AND `status` = 1
$customer = Customer::findOne([
    'id' => 123,
    'status' => Customer::STATUS_ACTIVE,
]);

// アクティブでない全ての顧客を返す
// SELECT * FROM `customer` WHERE `status` = 0
$customer = Customer::findAll([
    'status' => Customer::STATUS_INACTIVE,
]);
```

> Note|注意: [[yii\db\ActiveRecord::findOne()]] も [[yii\db\ActiveQuery::one()]] も、生成される SQL 文に `LIMIT 1` を追加しません。
あなたのクエリが多数のデータ行を返すかもしれない場合は、パフォーマンスを向上させるために、例えば `Customer::find()->limit(1)->one()` のように、`limit(1)` を明示的に呼ぶべきです。
。

クエリ構築メソッドを使う以外に、生の SQL を書いてデータをクエリして結果をアクティブレコードオブジェクトに投入することも出来ます。
[[yii\db\ActiveRecord::queryBySql()]] メソッドを呼ぶことによってそうすることが出来ます。

```php
// アクティブでない全ての顧客を返す
$sql = 'SELECT * FROM customer WHERE status=:status';
$customers = Customer::findBySql($sql, [':status' => Customer::STATUS_INACTIVE])->all();
```
[[yii\db\ActiveRecord::queryBySql()|queryBySql()]] を呼んだ後では、無視されますので、クエリ構築メソッドを追加で呼び出してはいけません。


## データにアクセスする <span id="accessing-data"></span>

既に述べたように、データベースから取得されたデータはアクティブレコードのインスタンスに投入されます。
そして、クエリ結果の各行がアクティブレコードの一つのインスタンスに対応します。
アクティブレコードインスタンスの属性にアクセスすることによって、カラムの値にアクセスすることが出来ます。
例えば、

```php
// "id" と "email" は "customer" テーブルのカラム名
$customer = Customer::findOne(123);
$id = $customer->id;
$email = $customer->email;
```

> Note|注意: アクティブレコードの属性の名前は、関連付けられたテーブルのカラムの名前に従って、大文字と小文字を区別して名付けられます。
  Yii は、関連付けられたテーブルの全てのカラムに対して、アクティブレコードの属性を自動的に定義します。
  これらの属性は、すべて、再宣言してはいけません。

アクティブレコードの属性はテーブルのカラムに従って命名されるため、テーブルのカラム名がアンダースコアで単語を分ける方法で命名されている場合は、`$customer->first_name` のような属性名を使って PHP コードを書くことになります。
コードスタイルの一貫性が気になるのであれば、テーブルのカラム名を (例えば camelCase を使う名前に) 変更しなければなりません。


### データ変換 <span id="data-transformation"></span>

入力または表示されるデータの形式が、データベースにデータを保存するときに使われるものと異なる場合がよくあります。
例えば、データベースでは顧客の誕生日を UNIX タイムスタンプで保存している (まあ、あまり良い設計ではありませんが) けれども、ほとんどの場合において誕生日を `'YYYY/MM/DD'` という形式の文字列として操作したい、というような場合です。
この目的を達するために、次のように、`Customer` アクティブレコードクラスにおいてデータ変換メソッドを定義することが出来ます。

```php
class Customer extends ActiveRecord
{
    // ...

    public function getBirthdayText()
    {
        return date('Y/m/d', $this->birthday);
    }
    
    public function setBirthdayText($value)
    {
        $this->birthday = strtotime($value);
    }
}
```

このようにすれば、PHP コードにおいて、`$customer->birthday` にアクセスする代りに、`$customer->birthdayText` にアクセスすれば、顧客の誕生日を `'YYYY/MM/DD'` の形式で入力および表示することが出来ます。


### データを配列に取得する <span id="data-in-arrays"></span>

データをアクティブレコードオブジェクトの形で取得するのは便利であり柔軟ですが、大きなメモリ使用量を要するために、大量のデータを取得しなければならない場合は、必ずしも望ましい方法ではありません。
そういう場合は、クエリメソッドを実行する前に [[yii\db\ActiveQuery::asArray()|asArray()]] を呼ぶことによって、PHP 配列を使ってデータを取得することが出来ます。

```php
// すべての顧客を返す
// 各顧客は連想配列として返される
$customers = Customer::find()
    ->asArray()
    ->all();
```

> Note|注意: このメソッドはメモリを節約してパフォーマンスを向上させますが、低レベルの DB 抽象レイヤに近いものであり、あなたはアクティブレコードの機能をほとんど失うことになります。
  非常に重要な違いがカラムの値のデータタイプにあります。
  アクティブレコードインスタンスとしてデータを返す場合、カラムの値は実際のカラムの型に従って自動的に型キャストされます。
  一方、配列としてデータを返す場合は、実際のカラムの型に関係なく、カラムの値は文字列になります。
  なぜなら、何も処理をしない場合の PDO の結果は文字列だからです。


### データをバッチモードで取得する <span id="data-in-batches"></span>

[クエリビルダ](db-query-builder.md) において、大量のデータをデータベースから検索する場合に、メモリ使用量を最小化するために *バッチクエリ* を使うことが出来るということを説明しました。
おなじテクニックをアクティブレコードでも使うことが出来ます。
例えば、

```php
// 一度に 10 人の顧客を読み出す
foreach (Customer::find()->batch(10) as $customers) {
    // $customers は 10 以下の Customer オブジェクトの配列
}
// 一度に 10 人の顧客を読み出して、一人ずつ反復する
foreach (Customer::find()->each(10) as $customer) {
    // $customer は Customer オブジェクト
}
// イーガーローディングをするバッチクエリ
foreach (Customer::find()->with('orders')->each() as $customer) {
}
```


## データを保存する <span id="inserting-updating-data"></span>

アクティブレコードを使えば、次のステップを踏んで簡単にデータをデータベースに保存することが出来ます。

1. アクティブレコードのインスタンスを準備する
2. アクティブレコードの属性に新しい値を割り当てる
3. [[yii\db\ActiveRecord::save()]] を呼んでデータをデータベースに保存する

例えば、

// 新しいデータ行を挿入する
$customer = new Customer();
$customer->name = 'James';
$customer->email = 'james@example.com';
$customer->save();

// 既存のデータ行を更新する
$customer = Customer::findOne(123);
$customer->email = 'james@newexample.com';
$customer->save();
```

[[yii\db\ActiveRecord::save()|save()]] メソッドは、アクティブレコードインスタンスの状態に従って、データ行を挿入するか、または、更新することが出来ます。
インスタンスが `new` 演算子によって新しく作成されたものである場合は、[[yii\db\ActiveRecord::save()|save()]] を呼び出すと、新しい行が挿入されます。
インスタンスがクエリメソッドの結果である場合は、[[yii\db\ActiveRecord::save()|save()]] を呼び出すと、そのインスタンスと関連付けられた行が更新されます。

アクティブレコードインスタンスの二つの状態は、その [[yii\db\ActiveRecord::isNewRecord|isNewRecord]] プロパティの値をチェックすることによって区別することが出来ます。
下記のように、このプロパティは [[yii\db\ActiveRecord::save()|save()]] によっても内部的に使用されています。

```php
public function save($runValidation = true, $attributeNames = null)
{
    if ($this->getIsNewRecord()) {
        return $this->insert($runValidation, $attributeNames);
    } else {
        return $this->update($runValidation, $attributeNames) !== false;
    }
}
```

> Tip|ヒント: [[yii\db\ActiveRecord::insert()|insert()]] または [[yii\db\ActiveRecord::update()|update()]] を直接に呼んでも、行を挿入または更新することが出来ます。


### データの検証 <span id="data-validation"></span>

[[yii\db\ActiveRecord]] は [[yii\base\Model]] を拡張したものですので、同じ [データ検証](input-validation.md) 機能を共有しています。
例えば、[[yii\base\Model::rules()|rules()]] メソッドをオーバーライドして検証規則を宣言することが出来ます。
[[yii\db\ActiveRecord::rules()|rules()]] メソッドをオーバーライドすることによって検証規則を宣言し、m[[yii\db\ActiveRecord::validate()|validate()]] メソッドを呼不ことによってテータの検証を実行することが出来ます。

[[yii\db\ActiveRecord::save()|save()]] を呼ぶと、デフォルトでは [[yii\db\ActiveRecord::validate()|validate()]] を自動的に呼びます。
検証が通った時だけ、実際にデータが保存されます。
検証が通らなかった時は単に false が返され、[[yii\db\ActiveRecord::errors|errors]] プロパティをチェックして検証エラーメッセージを取得することが出来ます。

> Tip|情報: データが検証を必要としないことが確実である場合 (例えば、データが信頼できるソースに由来するものである場合) は、検証をスキップするために `save(false)` を呼ぶことが出来ます。


### 一括代入 <span id="massive-assignment"></span>

通常の [モデル](structure-models.md) と同じように、アクティブレコードのインスタンスも  [一括代入機能](structure-models.md#massive-assignment) を享受することが出来ます。
この機能を使うと、下記で示されているように、一つの PHP 文で、アクティブレコードインスタンスの複数の属性に値を割り当てることが出来ます。
ただし、[安全な属性](structure-models.md#safe-attributes) だけが一括代入が可能であることを記憶しておいてください。

```php
$values = [
    'name' => 'James',
    'email' => 'james@example.com',
];

$customer = new Customer();

$customer->attributes = $values;
$customer->save();
```


### カウンタを更新する <span id="updating-counters"></span>

データベーステーブルのあるカラムの値を増加・減少させるのは、よくある仕事です。
私たちはそのようなカラムをカウンタカラムと呼んでいます。
[[yii\db\ActiveRecord::updateCounters()|updateCounters()]] を使って一つまたは複数のカウンタカラムを更新することが出来ます。
例えば、

```php
$post = Post::findOne(100);

// UPDATE `post` SET `view_count` = `view_count` + 1 WHERE `id` = 100
$post->updateCounters(['view_count' => 1]);
```

> Note|注意: カウンタカラムを更新するのに [[yii\db\ActiveRecord::save()]] を使うと、不正確な結果になってしまう場合があります。
というのは、同じカウンタの値を読み書きする複数のリクエストによって、同一のカウンタが保存される可能性があるからです。


### ダーティな属性 <span id="dirty-attributes"></span>

[[yii\db\ActiveRecord::save()|save()]] を呼んでアクティブレコードインスタンスを保存すると、*ダーティな属性* だけが保存されます。
属性は、DB からロードされた後、または、最後に保存された後にその値が変更されると、*ダーティ* であると見なされます。
ただし、データ検証は、アクティブレコードインスタンスがダーティな属性を持っているかどうかに関係なく実施されることに注意してください。

アクティブレコードはダーティな属性のリストを自動的に保守します。
そうするために、一つ前のバージョンの属性値を保持して、最新のバージョンと比較します。
[[yii\db\ActiveRecord::getDirtyAttributes()]] を呼ぶと、現在ダーティである属性を取得することが出来ます。
また、[[yii\db\ActiveRecord::markAttributeDirty()]] を呼んで、ある属性をダーティであると明示的にマークすることも出来ます。

最新の修正を受ける前の属性値を知りたい場合は、[[yii\db\ActiveRecord::getOldAttributes()|getOldAttributes()]] または [[yii\db\ActiveRecord::getOldAttribute()|getOldAttribute()]] を呼ぶことが出来ます。


### デフォルト属性値 <span id="default-attribute-values"></span>

あなたのテーブルのカラムの中には、データベースでデフォルト値が定義されているものもあるかも知れません。
そして、場合によっては、アクティブレコードインスタンスのウェブフォームに、そういうデフォルト値をあらかじめ投入したいことがあるでしょう。
同じデフォルト値を繰り返して書くことを避けるために、[[yii\db\ActiveRecord::loadDefaultValues()|loadDefaultValues()]] を呼んで、DB で定義されたデフォルト値を対応するアクティブレコードの属性に投入することが出来ます。

```php
$customer = new Customer();
$customer->loadDefaultValues();
// $customer->xyz には、"xyz" カラムを定義するときに宣言されたデフォルト値が割り当てられる
```


### 複数の行を更新する <span id="updating-multiple-rows"></span>

上述のメソッドは、すべて、個別のアクティブレコードインスタンスに対して作用し、個別のテーブル行を挿入したり更新したりするものです。
複数の行を同時に更新するためには、代りに、スタティックなメソッドである [[yii\db\ActiveRecord::updateAll()|updateAll()]] を呼ばなければなりません。

```php
// UPDATE `customer` SET `status` = 1 WHERE `email` LIKE `%@example.com`
Customer::updateAll(['status' => Customer::STATUS_ACTIVE], ['like', 'email', '@example.com']);
```

同様に、[[yii\db\ActiveRecord::updateAllCounters()|updateAllCounters()]] を呼んで、複数の行のカウンタカラムを同時に更新することが出来ます。

```php
// UPDATE `customer` SET `age` = `age` + 1
Customer::updateAllCounters(['age' => 1]);
```


## データを削除する <span id="deleting-data"></span>

一行のデータを削除するためには、最初にその行に対応するアクティブレコードインスタンスを取得して、次に [[yii\db\ActiveRecord::delete()]] メソッドを呼びます。

```php
$customer = Customer::findOne(123);
$customer->delete();
```

[[yii\db\ActiveRecord::deleteAll()]] を呼んで、複数またはすべてのデータ行を削除することが出来ます。例えば、

```php
Customer::deleteAll(['status' => Customer::STATUS_INACTIVE]);
```

> Note|注意: [[yii\db\ActiveRecord::deleteAll()|deleteAll()]] を呼ぶときは、十分に注意深くしてください。
  なぜなら、条件の指定を間違うと、あなたのテーブルからすべてのデータを完全に消し去ってしまうことになるからです。


## アクティブレコードのライフサイクル <span id="ar-life-cycles"></span>

アクティブレコードがさまざまな目的で使用される場合のそれぞれのライフサイクルを理解しておくことは重要なことです。
それぞれのライフサイクルにおいては、特定の一続きのメソッドが呼び出されます。
そして、これらのメソッドをオーバーライドして、ライフサイクルをカスタマイズするチャンスを得ることが出来ます。
また、ライフサイクルの中でトリガされる特定のアクティブレコードイベントに反応して、あなたのカスタムコードを挿入することも出来ます。
これらのイベントが特に役に立つのは、アクティブレコードのライフサイクルをカスタマイズする必要のあるアクティブレコード [ビヘイビア](concept-behaviors.md) を開発する際です。

次に、さまざまなアクティブレコードのライフサイクルと、そのライフサイクルに含まれるメソッドやイベントを要約します。


### 新しいインスタンスのライフサイクル <span id="new-instance-life-cycle"></span>

`new` 演算子によって新しいアクティブレコードインスタンスを作成する場合は、次のライフサイクルを経ます。

1. クラスのコンストラクタ。
2. [[yii\db\ActiveRecord::init()|init()]]: [[yii\db\ActiveRecord::EVENT_INIT|EVENT_INIT]] イベントをトリガ。


### データをクエリする際のライフサイクル <span id="querying-data-life-cycle"></span>


[クエリメソッド](#querying-data) のどれか一つによってデータをクエリする場合は、新しくデータを投入されるアクティブレコードは次のライフサイクルを経ます。

1. クラスのコンストラクタ。
2. [[yii\db\ActiveRecord::init()|init()]]: [[yii\db\ActiveRecord::EVENT_INIT|EVENT_INIT]] イベントをトリガ。
3. [[yii\db\ActiveRecord::afterFind()|afterFind()]]: [[yii\db\ActiveRecord::EVENT_AFTER_FIND|EVENT_AFTER_FIND]] イベントをトリガ。


### データを保存する際のライフサイクル <span id="saving-data-life-cycle"></span>

[[yii\db\ActiveRecord::save()|save()]] を呼んでアクティブレコードインスタンスを挿入または更新する場合は、次のライフサイクルを経ます。

1. [[yii\db\ActiveRecord::beforeValidate()|beforeValidate()]]: [[yii\db\ActiveRecord::EVENT_BEFORE_VALIDATE|EVENT_BEFORE_VALIDATE]] イベントをトリガ。
   このメソッドが false を返すか、[[yii\base\ModelEvent::isValid]] が false であった場合、残りのステップはスキップされる。
2. データ検証を実行。データ検証が失敗した場合、3 以降のステップはスキップされる。
3. [[yii\db\ActiveRecord::afterValidate()|afterValidate()]]: [[yii\db\ActiveRecord::EVENT_AFTER_VALIDATE|EVENT_AFTER_VALIDATE]] イベントをトリガ。
4. [[yii\db\ActiveRecord::beforeSave()|beforeSave()]]: [[yii\db\ActiveRecord::EVENT_BEFORE_INSERT|EVENT_BEFORE_INSERT]] または [[yii\db\ActiveRecord::EVENT_BEFORE_UPDATE|EVENT_BEFORE_UPDATE]] イベントをトリガ。
   このメソッドが false を返すか、[[yii\base\ModelEvent::isValid]] が false であった場合、残りのステップはスキップされる。
5. 実際のデータの挿入または更新を実行する。
6. [[yii\db\ActiveRecord::afterSave()|afterSave()]]: [[yii\db\ActiveRecord::EVENT_AFTER_INSERT|EVENT_AFTER_INSERT]] または [[yii\db\ActiveRecord::EVENT_AFTER_UPDATE|EVENT_AFTER_UPDATE]] イベントをトリガ。
   

### データを削除する際のライフサイクル <span id="deleting-data-life-cycle"></span>

[[yii\db\ActiveRecord::delete()|delete()]] を呼んでアクティブレコードインスタンスを削除する際は、次のライフサイクルを経ます。

1. [[yii\db\ActiveRecord::beforeDelete()|beforeDelete()]]: [[yii\db\ActiveRecord::EVENT_BEFORE_DELETE|EVENT_BEFORE_DELETE]] イベントをトリガ。
   このメソッドが false を返すか、[[yii\base\ModelEvent::isValid]] が false であった場合は、残りのステップはスキップされる。
2. 実際のデータの削除を実行する。
3. [[yii\db\ActiveRecord::afterDelete()|afterDelete()]]: [[yii\db\ActiveRecord::EVENT_AFTER_DELETE|EVENT_AFTER_DELETE]] イベントをトリガ。


> Note|注意: 次のメソッドは、どれを呼んでも、上記のライフサイクルを開始させません。
>
> - [[yii\db\ActiveRecord::updateAll()]] 
> - [[yii\db\ActiveRecord::deleteAll()]]
> - [[yii\db\ActiveRecord::updateCounters()]] 
> - [[yii\db\ActiveRecord::updateAllCounters()]] 


## トランザクション操作 <span id="transactional-operations"></span>

アクティブレコードを扱う際には、二つの方法でトランザクション操作を処理することができます。
最初の方法は、"[データベースの基礎](db-dao.md)" の「トランザクション」の項で説明したように、全てを手作業でやる方法です。
もう一つの方法として、`transactions` メソッドを実装して、モデルのシナリオごとに、どの操作をトランザクションで囲むかを指定することが出来ます。

```php
class Post extends \yii\db\ActiveRecord
{
    public function transactions()
    {
        return [
            'admin' => self::OP_INSERT,
            'api' => self::OP_INSERT | self::OP_UPDATE | self::OP_DELETE,
            // 上は次と等価
            // 'api' => self::OP_ALL,
        ];
    }
}
```

上記において、`admin` と `api` はモデルのシナリオであり、`OP_` で始まる定数は、これらのシナリオについてトランザクションで囲まれるべき操作を示しています。
サポートされている操作は、`OP_INSERT`、`OP_UPDATE`、そして、`OP_DELETE` です。
`OP_ALL` は三つ全てを示します。

このような自動的なトランザクションは、`beforeSave`、`afterSave`、`beforeDelete`、`afterDelete` によってデータベースに追加の変更を加えており、本体の変更と追加の変更の両方が成功した場合にだけデータベースにコミットしたい、というときに取り分けて有用です。


## 楽観的ロック <span id="optimistic-locks"></span>

楽観的ロックは、一つのデータ行が複数のユーザによって更新されるときに発生しうる衝突を回避するための方法です。
例えば、ユーザ A と ユーザ B が 同時に同じ wiki 記事を編集しており、ユーザ A が自分の編集結果を保存した後に、ユーザ B も自分の編集結果を保存しようとして「保存」ボタンをクリックする場合を考えてください。
ユーザ B は、実際には古くなったバージョンの記事に対する操作をしようとしていますので、彼が記事を保存するのを防止して、彼に何らかのヒントメッセージを表示する方策を取ることが望まれます。

楽観的ロックは、あるカラムを使って各行のバージョン番号を記録するという方法によって、上記の問題を解決します。
行が古くなったバージョン番号とともに保存されようとすると、[[yii\db\StaleObjectException]] 例外が投げられて、行が保存されるのが防止されます。
楽観的ロックは、 [[yii\db\ActiveRecord::update()]] または [[yii\db\ActiveRecord::delete()]] メソッドを使って既存の行を更新または削除しようとする場合にだけサポートされます。

楽観的ロックを使用するためには、次のようにします。

1. 各行のバージョン番号を保存するカラムを作成します。カラムのタイプは `BIGINT DEFAULT 0` でなければなりません。
   `optimisticLock()` メソッドをオーバーライドして、このカラムの名前を返すようにします。
2. ユーザ入力を収集するウェブフォームに、更新されるレコードのロックバージョンを保持する隠しフィールドを追加します。
3. データ更新を行うコントローラアクションにおいて、[[\yii\db\StaleObjectException]] 例外を捕捉して、衝突を解決するために必要なビジネスロジック (例えば、変更をマージしたり、データの陳腐化を知らせたり) を実装します。


## リレーショナルデータを扱う

テーブルのリレーショナルデータもアクティブレコードを使ってクエリすることが出来ます
(すなわち、テーブル A のデータを選択すると、テーブル B の関連付けられたデータも一緒に取り込むことが出来ます)。
アクティブレコードのおかげで、返されるリレーショナルデータは、プライマリテーブルと関連付けられたアクティブレコードオブジェクトのプロパティのようにアクセスすることが出来ます。

例えば、適切なリレーションが宣言されていれば、`$customer->orders` にアクセスすることによって、指定された顧客が発行した注文を表す `Order` オブジェクトの配列を取得することが出来ます。

リレーションを宣言するためには、[[yii\db\ActiveQuery]] オブジェクトを返すゲッターメソッドを定義します。そして、その [[yii\db\ActiveQuery]] オブジェクトに、リレーションのコンテキストに関する情報を持たせ、従って関連するレコードだけをクエリさせます。
例えば、

```php
class Customer extends \yii\db\ActiveRecord
{
    public function getOrders()
    {
        // Customer は Order.customer_id -> id によって、複数の Order を持つ
        return $this->hasMany(Order::className(), ['customer_id' => 'id']);
    }
}

class Order extends \yii\db\ActiveRecord
{
    public function getCustomer()
    {
        // Order は Customer.id -> customer_id によって、一つの Customer を持つ
        return $this->hasOne(Customer::className(), ['id' => 'customer_id']);
    }
}
```

上記の例で使用されている [[yii\db\ActiveRecord::hasMany()]] と [[yii\db\ActiveRecord::hasOne()]] のメソッドは、リレーショナルデータベースにおける多対一と一対一の関係を表現するために使われます。
例えば、顧客 (customer) は複数の注文 (order) を持ち、注文 (order) は一つの顧客 (customer)を持つ、という関係です。
これらのメソッドはともに二つのパラメータを取り、[[yii\db\ActiveQuery]] オブジェクトを返します。

 - `$class`: 関連するモデルのクラス名。これは完全修飾のクラス名でなければなりません。
 - `$link`: 二つのテーブルに属するカラム間の関係。これは配列として与えられなければなりません。
   配列のキーは、`$class` と関連付けられるテーブルにあるカラムの名前であり、配列の値はリレーションを宣言しているクラスのテーブルにあるカラムの名前です。
   リレーションをテーブルの外部キーに基づいて定義するのが望ましいプラクティスです。

リレーションを宣言した後は、リレーショナルデータを取得することは、対応するゲッターメソッドで定義されているコンポーネントのプロパティを取得するのと同じように、とても簡単なことになります。

```php
// 顧客の注文を取得する
$customer = Customer::findOne(1);
$orders = $customer->orders;  // $orders は Order オブジェクトの配列
```

舞台裏では、上記のコードは、各行について一つずつ、次の二つの SQL クエリを実行します。

```sql
SELECT * FROM customer WHERE id=1;
SELECT * FROM order WHERE customer_id=1;
```

> Tip|情報: `$customer->orders` という式に再びアクセスした場合は、第二の SQL クエリはもう実行されません。
  第二の SQL クエリは、この式が最初にアクセスされた時だけ実行されます。
  二度目以降のアクセスでは、内部的にキャッシュされている以前に読み出した結果が返されるだけです。
  リレーショナルデータを再クエリしたい場合は、単純に、まず既存の式を未設定状態に戻して (`unset($customer->orders);`) から、再度、`$customer->orders` にアクセスします。

場合によっては、リレーショナルクエリにパラメータを渡したいことがあります。
例えば、顧客の注文を全て返す代りに、小計が指定した金額を超える大きな注文だけを返したいことがあるでしょう。
そうするためには、次のようなゲッターメソッドで `bigOrders` リレーションを宣言します。

```php
class Customer extends \yii\db\ActiveRecord
{
    public function getBigOrders($threshold = 100)
    {
        return $this->hasMany(Order::className(), ['customer_id' => 'id'])
            ->where('subtotal > :threshold', [':threshold' => $threshold])
            ->orderBy('id');
    }
}
```

`hasMany()` が 返す [[yii\db\ActiveQuery]] は、[[yii\db\ActiveQuery]] のメソッドを呼ぶことでクエリをカスタマイズ出来るものであることを覚えておいてください。

上記の宣言によって、`$customer->bigOrders` にアクセスした場合は、小計が 100 以上である注文だけが返されることになります。
異なる閾値を指定するためには、次のコードを使用します。

```php
$orders = $customer->getBigOrders(200)->all();
```

> Note|注意: リレーションメソッドは [[yii\db\ActiveQuery]] のインスタンスを返します。
リレーションを属性 (すなわち、クラスのプロパティ) としてアクセスした場合は、返り値はリレーションのクエリ結果となります。
クエリ結果は、リレーションが複数のレコードを返すものか否かに応じて、[[yii\db\ActiveRecord]] の一つのインスタンス、またはその配列、または null となります。
例えば、`$customer->getOrders()` は `ActiveQuery` のインスタンスを返し、`$customer->orders` は `Order` オブジェクトの配列 (またはクエリ結果が無い場合は空の配列) を返します。


### 中間テーブルを使うリレーション

場合によっては、二つのテーブルが [中間テーブル][] と呼ばれる中間的なテーブルによって関連付けられていることがあります。
そのようなリレーションを宣言するために、[[yii\db\ActiveQuery::via()|via()]] または [[yii\db\ActiveQuery::viaTable()|viaTable()]] メソッドを呼んで、[[yii\db\ActiveQuery]] オブジェクトをカスタマイズすることが出来ます。

例えば、テーブル `order` とテーブル `item` が中間テーブル `order_item` によって関連付けられている場合、`Order` クラスにおいて `items` リレーションを次のように宣言することが出来ます。

```php
class Order extends \yii\db\ActiveRecord
{
    public function getItems()
    {
        return $this->hasMany(Item::className(), ['id' => 'item_id'])
            ->viaTable('order_item', ['order_id' => 'id']);
    }
}
```

[[yii\db\ActiveQuery::via()|via()]] メソッドは、最初のパラメータとして、結合テーブルの名前ではなく、アクティブレコードクラスで宣言されているリレーションの名前を取ること以外は、[[yii\db\ActiveQuery::viaTable()|viaTable()]] と同じです。
例えば、上記の `items` リレーションは次のように宣言しても等価です。

```php
class Order extends \yii\db\ActiveRecord
{
    public function getOrderItems()
    {
        return $this->hasMany(OrderItem::className(), ['order_id' => 'id']);
    }

    public function getItems()
    {
        return $this->hasMany(Item::className(), ['id' => 'item_id'])
            ->via('orderItems');
    }
}
```

[中間テーブル]: https://en.wikipedia.org/wiki/Junction_table "Junction table on Wikipedia"


### レイジーローディングとイーガーローディング

前に述べたように、関連オブジェクトに最初にアクセスしたときに、アクティブレコードは DB クエリを実行して関連データを読み出し、それを関連オブジェクトに投入します。
同じ関連オブジェクトに再度アクセスしても、クエリは実行されません。
これを *レイジーローディング* と呼びます。
例えば、

```php
// 実行される SQL: SELECT * FROM customer WHERE id=1
$customer = Customer::findOne(1);
// 実行される SQL: SELECT * FROM order WHERE customer_id=1
$orders = $customer->orders;
// SQL は実行されない
$orders2 = $customer->orders;
```

レイジーローディングは非常に使い勝手が良いものです。しかし、次のシナリオでは、パフォーマンスの問題を生じ得ます。

```php
// 実行される SQL: SELECT * FROM customer WHERE id=1
$customers = Customer::find()->limit(100)->all();

foreach ($customers as $customer) {
    // 実行される SQL: SELECT * FROM order WHERE customer_id=...
    $orders = $customer->orders;
    // ... $orders を処理 ...
}
```

データベースに 100 人以上の顧客が登録されていると仮定した場合、上記のコードで何個の SQL クエリが実行されるでしようか?
101 です。最初の SQL クエリが 100 人の顧客を返します。
次に、100 人の顧客全てについて、それぞれ、顧客の注文を返すための SQL クエリが実行されます。

上記のパフォーマンスの問題を解決するためには、[[yii\db\ActiveQuery::with()]] を呼んでいわゆる *イーガーローディング* を使うことが出来ます。

```php
// 実行される SQL: SELECT * FROM customer LIMIT 100;
//                 SELECT * FROM orders WHERE customer_id IN (1,2,...)
$customers = Customer::find()->limit(100)
    ->with('orders')->all();

foreach ($customers as $customer) {
    // SQL は実行されない
    $orders = $customer->orders;
    // ... $orders を処理 ...
}
```

ご覧のように、同じ仕事をするのに必要な SQL クエリがたった二つになります。

> Info|情報: 一般化して言うと、`N` 個のリレーションのうち `M` 個のリレーションが `via()` または `viaTable()` によって定義されている場合、この `N` 個のリレーションをイーガーロードしようとすると、合計で `1+M+N` 個の SQL クエリが実行されます。
> 主たるテーブルの行を返すために一つ、`via()` または `viaTable()` の呼び出しに対応する `M` 個の中間テーブルのそれぞれに対して一つずつ、そして、`N` 個の関連テーブルのそれぞれに対して一つずつ、という訳です。

> Note|注意: イーガーローディングで `select()` をカスタマイズしようとする場合は、関連モデルにリンクするカラムを必ず含めてください。
> そうしないと、関連モデルは読み出されません。例えば、

```php
$orders = Order::find()->select(['id', 'amount'])->with('customer')->all();
// $orders[0]->customer は常に null になる。この問題を解決するためには、次のようにしなければならない。
$orders = Order::find()->select(['id', 'amount', 'customer_id'])->with('customer')->all();
```

場合によっては、リレーショナルクエリをその場でカスタマイズしたいことがあるでしょう。
これは、レイジーローディングでもイーガーローディングでも、可能です。例えば、

```php
$customer = Customer::findOne(1);
// レイジーローディング: SELECT * FROM order WHERE customer_id=1 AND subtotal>100
$orders = $customer->getOrders()->where('subtotal>100')->all();

// イーガーローディング: SELECT * FROM customer LIMIT 100
//                       SELECT * FROM order WHERE customer_id IN (1,2,...) AND subtotal>100
$customers = Customer::find()->limit(100)->with([
    'orders' => function($query) {
        $query->andWhere('subtotal>100');
    },
])->all();
```


### 逆リレーション

リレーションは、たいていの場合、ペアで定義することが出来ます。
例えば、`Customer` が `orders` という名前のリレーションを持ち、`Order` が `customer` という名前のリレーションを持つ、ということがあります。

```php
class Customer extends ActiveRecord
{
    ....
    public function getOrders()
    {
        return $this->hasMany(Order::className(), ['customer_id' => 'id']);
    }
}

class Order extends ActiveRecord
{
    ....
    public function getCustomer()
    {
        return $this->hasOne(Customer::className(), ['id' => 'customer_id']);
    }
}
```

次に例示するクエリを実行すると、注文 (order) のリレーションとして取得した顧客 (customer) が、最初にその注文をリレーションとして取得した顧客とは別の Customer オブジェクトになってしまうことに気付くでしょう。
また、`customer->orders` にアクセスすると一個の SQL が実行され、`order->customer` にアクセスするともう一つ別の SQL が実行されるということにも気付くでしょう。

```php
// SELECT * FROM customer WHERE id=1
$customer = Customer::findOne(1);
// "等しくない" がエコーされる
// SELECT * FROM order WHERE customer_id=1
// SELECT * FROM customer WHERE id=1
if ($customer->orders[0]->customer === $customer) {
    echo '等しい';
} else {
    echo '等しくない';
}
```

冗長な最後の SQL 文の実行を避けるためには、次のように、[[yii\db\ActiveQuery::inverseOf()|inverseOf()]] メソッドを呼んで、`customer` と `oerders` のリレーションに対して逆リレーションを宣言することが出来ます。

```php
class Customer extends ActiveRecord
{
    ....
    public function getOrders()
    {
        return $this->hasMany(Order::className(), ['customer_id' => 'id'])->inverseOf('customer');
    }
}
```

こうすると、上記と同じクエリを実行したときに、次の結果を得ることが出来ます。

```php
// SELECT * FROM customer WHERE id=1
$customer = Customer::findOne(1);
// "等しい" がエコーされる
// SELECT * FROM order WHERE customer_id=1
if ($customer->orders[0]->customer === $customer) {
    echo '等しい';
} else {
    echo '等しくない';
}
```

上記では、レイジーローディングにおいて逆リレーションを使う方法を示しました。
逆リレーションはイーガーローディングにも適用されます。

```php
// SELECT * FROM customer
// SELECT * FROM order WHERE customer_id IN (1, 2, ...)
$customers = Customer::find()->with('orders')->all();
// "等しい" がエコーされる
if ($customers[0]->orders[0]->customer === $customers[0]) {
    echo '等しい';
} else {
    echo '等しくない';
}
```

> Note|注意: 逆リレーションはピボットテーブルを含むリレーションに対しては定義することが出来ません。
> つまり、リレーションが [[yii\db\ActiveQuery::via()|via()]] または [[yii\db\ActiveQuery::viaTable()|viaTable()]] によって定義されている場合は、[[yii\db\ActiveQuery::inverseOf()]] を追加で呼ぶことは出来ません。


### リレーションを使ってテーブルを結合する <a name="joining-with-relations">

リレーショナルデータベースを扱う場合、複数のテーブルを結合して、JOIN SQL 文にさまざまなクエリ条件とパラメータを指定することは、ごく当り前の仕事です。
その目的を達するために、[[yii\db\ActiveQuery::join()]] を明示的に呼んで JOIN クエリを構築する代りに、既存のリレーション定義を再利用して [[yii\db\ActiveQuery::joinWith()]] を呼ぶことが出来ます。
例えば、

```php
// 全ての注文を検索して、注文を顧客 ID と注文 ID でソートする。同時に "customer" をイーガーロードする。
$orders = Order::find()->joinWith('customer')->orderBy('customer.id, order.id')->all();
// 書籍を含む全ての注文を検索し、"books" をイーガーロードする。
$orders = Order::find()->innerJoinWith('books')->all();
```

上記において、[[yii\db\ActiveQuery::innerJoinWith()|innerJoinWith()]] メソッドは、結合タイプを `INNER JOIN` とする [[yii\db\ActiveQuery::joinWith()|joinWith()]] へのショートカットです。

一個または複数のリレーションを結合することが出来ます。リレーションにクエリ条件をその場で適用することも出来ます。
また、サブリレーションを結合することも出来ます。例えば、

```php
// 複数のリレーションを結合
// 書籍を含む注文で、過去 24 時間以内に登録した顧客によって発行された注文を検索する
$orders = Order::find()->innerJoinWith([
    'books',
    'customer' => function ($query) {
        $query->where('customer.created_at > ' . (time() - 24 * 3600));
    }
])->all();
// サブリレーションとの結合: 書籍および書籍の著者を結合
$orders = Order::find()->joinWith('books.author')->all();
```

舞台裏では、Yii は最初に JOIN SQL 文を実行して、その JOIN SQL に適用された条件を満たす主たるモデルを取得します。
そして、次にリレーションごとのクエリを実行して、対応する関連レコードを投入します。

[[yii\db\ActiveQuery::joinWith()|joinWith()]] と [[yii\db\ActiveQuery::with()|with()]] の違いは、前者が主たるモデルクラスのテーブルと関連モデルクラスのテーブルを結合して主たるモデルを読み出すのに対して、後者は主たるモデルクラスのテーブルに対してだけクエリを実行して主たるモデルを読み出す、という点にあります。

この違いによって、[[yii\db\ActiveQuery::joinWith()|joinWith()]] では、JOIN SQL 文だけに指定できるクエリ条件を適用することが出来ます。
例えば、上記の例のように、関連モデルに対する条件によって主たるモデルをフィルタすることが出来ます。
主たるモデルを関連テーブルのカラムを使って並び替えることも出来ます。

[[yii\db\ActiveQuery::joinWith()|joinWith()]] を使うときは、カラム名の曖昧さを解決することについて、あなたが責任を負わなければなりません。
上記の例では、order テーブルと item テーブルがともに `id` という名前のカラムを持っているため、`item.id` と `order.id` を使って、`id` カラムの参照の曖昧さを解決しています。

デフォルトでは、リレーションを結合すると、リレーションがイーガーロードされることにもなります。
このデフォルトの動作は、指定されたリレーションをイーガーロードするかどうかを規定する `$eagerLoading` パラメータを渡して、変更することが出来ます。

また、デフォルトでは、[[yii\db\ActiveQuery::joinWith()|joinWith()]] は関連テーブルを結合するのに `LEFT JOIN` を使います。
結合タイプをカスタマイズするために `$joinType` パラメータを渡すことが出来ます。
`INNER JOIN` タイプのためのショートカットとして、[[yii\db\ActiveQuery::innerJoinWith()|innerJoinWith()]] を使うことが出来ます。

下記に、いくつかの例を追加します。

```php
// 書籍を含む注文を全て検索するが、"books" はイーガーロードしない。
$orders = Order::find()->innerJoinWith('books', false)->all();
// これも上と等価
$orders = Order::find()->joinWith('books', false, 'INNER JOIN')->all();
```

二つのテーブルを結合するとき、場合によっては、JOIN クエリの ON の部分で何らかの追加条件を指定する必要があります。
これは、次のように、[[yii\db\ActiveQuery::onCondition()]] メソッドを呼んで実現することが出来ます。

```php
class User extends ActiveRecord
{
    public function getBooks()
    {
        return $this->hasMany(Item::className(), ['owner_id' => 'id'])->onCondition(['category_id' => 1]);
    }
}
```

上記においては、[[yii\db\ActiveRecord::hasMany()|hasMany()]] メソッドが [[yii\db\ActiveQuery]] のインスタンスを返しています。
そして、それに対して [[yii\db\ActiveQuery::onCondition()|onCondition()]] が呼ばれて、`category_id` が 1 である品目だけが返されるべきことを指定しています。

[[yii\db\ActiveQuery::joinWith()|joinWith()]] を使ってクエリを実行すると、指定された ON 条件が対応する JOIN クエリの ON の部分に挿入されます。
例えば、

```php
// SELECT user.* FROM user LEFT JOIN item ON item.owner_id=user.id AND category_id=1
// SELECT * FROM item WHERE owner_id IN (...) AND category_id=1
$users = User::find()->joinWith('books')->all();
```

[[yii\db\ActiveQuery::with()]] を使ってイーガーロードする場合や、レイジーロードする場合には、JOIN クエリは使われないため、ON 条件が対応する SQL 文の WHERE の部分に挿入されることに注意してください。
例えば、

```php
// SELECT * FROM user WHERE id=10
$user = User::findOne(10);
// SELECT * FROM item WHERE owner_id=10 AND category_id=1
$books = $user->books;
```


関連付けを扱う
--------------

アクティブレコードは、二つのアクティブレコードオブジェクト間の関連付けを確立および破棄するために、次の二つのメソッドを提供しています。

- [[yii\db\ActiveRecord::link()|link()]]
- [[yii\db\ActiveRecord::unlink()|unlink()]]

例えば、顧客と新しい注文があると仮定したとき、次のコードを使って、その注文をその顧客のものとすることが出来ます。

```php
$customer = Customer::findOne(1);
$order = new Order();
$order->subtotal = 100;
$customer->link('orders', $order);
```

上記の [[yii\db\ActiveRecord::link()|link()]] の呼び出しは、注文の `customer_id` に `$customer` のプライマリキーの値を設定し、[[yii\db\ActiveRecord::save()|save()]] を呼んで注文をデータベースに保存します。


DBMS 間のリレーション
---------------------

アクティブレコードは、異なる DBMS に属するエンティティ間、例えば、リレーショナルデータベースのテーブルと MongoDB のコレクションの間に、リレーションを確立することを可能にしています。
そのようなリレーションでも、何も特別なコードは必要ありません。

```php
// リレーショナルデータベースのアクティブレコード
class Customer extends \yii\db\ActiveRecord
{
    public static function tableName()
    {
        return 'customer';
    }

    public function getComments()
    {
        // リレーショナルデータベースに保存されている Customer は、MongoDB コレクションに保存されている複数の Comment を持つ
        return $this->hasMany(Comment::className(), ['customer_id' => 'id']);
    }
}

// MongoDb のアクティブレコード
class Comment extends \yii\mongodb\ActiveRecord
{
    public static function collectionName()
    {
        return 'comment';
    }

    public function getCustomer()
    {
        // MongoDB コレクションに保存されている Comment は、リレーショナルデータベースに保存されている一つの Customer を持つ
        return $this->hasOne(Customer::className(), ['id' => 'customer_id']);
    }
}
```

アクティブレコードの全ての機能、例えば、イーガーローディングやレイジーローディング、関連付けの確立や破棄などが、DBMS 間のリレーションでも利用可能です。

> Note|注意: DBMS ごとのアクティブレコードの実装には、DBMS 固有のメソッドや機能が含まれる場合があり、そういうものは DBMS 間のリレーションには適用できないということを忘れないでください。
  例えば、[[yii\db\ActiveQuery::joinWith()]] の使用が MongoDB コレクションに対するリレーションでは動作しないことは明白です。


スコープ
--------

[[yii\db\ActiveRecord::find()|find()]] または [[yii\db\ActiveRecord::findBySql()|findBySql()]] を呼ぶと、[[yii\db\ActiveQuery|ActiveQuery]] のインスタンスが返されます。
そして、追加のクエリメソッド、例えば、[[yii\db\ActiveQuery::where()|where()]] や [[yii\db\ActiveQuery::orderBy()|orderBy()]] を呼んで、クエリ条件をさらに指定することが出来ます。

別々の場所で同じ一連のクエリメソッドを呼びたいということがあり得ます。
そのような場合には、いわゆる *スコープ* を定義することを検討すべきです。
スコープは、本質的には、カスタムクエリクラスの中で定義されたメソッドであり、クエリオブジェクトを修正する一連のメソッドを呼ぶものです。
スコープを定義しておくと、通常のクエリメソッドを呼ぶ代りに、スコープを使うことが出来るようになります。

スコープを定義するためには二つのステップが必要です。
最初に、モデルのためのカスタムクエリクラスを作成して、このクラスの中に必要なスコープメソッドを定義します。
例えば、`Comment` モデルのために `CommentQuery` クラスを作成して、次のように、`active()` というスコープメソッドを定義します。

```php
namespace app\models;

use yii\db\ActiveQuery;

class CommentQuery extends ActiveQuery
{
    public function active($state = true)
    {
        $this->andWhere(['active' => $state]);
        return $this;
    }
}
```

重要な点は、以下の通りです。

1. クラスは `yii\db\ActiveQuery` (または、`yii\mongodb\ActiveQuery` などの、その他の `ActiveQuery`) を拡張したものにしなければなりません。
2. メソッドは `public` で、メソッドチェーンが出来るように `$this` を返さなければなりません。メソッドはパラメータを取ることが出来ます。
3. クエリ条件を修正する方法については、[[yii\db\ActiveQuery]] のメソッド群を参照するのが非常に役に立ちます。

次に、[[yii\db\ActiveRecord::find()]] をオーバーライドして、通常の [[yii\db\ActiveQuery|ActiveQuery]] の代りに、カスタムクエリクラスを使うようにします。
上記の例のためには、次のコードを書く必要があります。

```php
namespace app\models;

use yii\db\ActiveRecord;

class Comment extends ActiveRecord
{
    /**
     * @inheritdoc
     * @return CommentQuery
     */
    public static function find()
    {
        return new CommentQuery(get_called_class());
    }
}
```

以上です。これで、カスタムスコープメソッドを使用することが出来ます。

```php
$comments = Comment::find()->active()->all();
$inactiveComments = Comment::find()->active(false)->all();
```

リレーションを定義するときにもスコープを使用することが出来ます。例えば、

```php
class Post extends \yii\db\ActiveRecord
{
    public function getActiveComments()
    {
        return $this->hasMany(Comment::className(), ['post_id' => 'id'])->active();

    }
}
```

または、リレーショナルクエリを実行するときに、その場でスコープを使うことも出来ます。

```php
$posts = Post::find()->with([
    'comments' => function($q) {
        $q->active();
    }
])->all();
```

### デフォルトスコープ

あなたが Yii 1.1 を前に使ったことがあれば、*デフォルトスコープ* と呼ばれる概念を知っているかも知れません。
デフォルトスコープは、全てのクエリに適用されるスコープです。
デフォルトスコープは、[[yii\db\ActiveRecord::find()]] をオーバライドすることによって、簡単に定義することが出来ます。
例えば、

```php
public static function find()
{
    return parent::find()->where(['deleted' => false]);
}
```

ただし、すべてのクエリにおいて、デフォルトの条件を上書きしないために、[[yii\db\ActiveQuery::where()|where()]] を使わず、[[yii\db\ActiveQuery::andWhere()|andWhere()]] または [[yii\db\ActiveQuery::orWhere()|orWhere()]] を使うべきであることに注意してください。

