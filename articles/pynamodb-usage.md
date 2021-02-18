---
title: "pythonでdynamo dbをORM風に扱えるpynamodbの使い方"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Python, DynamoDB, AWS]
published: true
---

dynamoをテーブルクラスを経由して叩けるライブラリ(ORMっぽい)

## 基本的な使い方
1. テーブルを表すクラスを作成する
2. テーブルクラスに対してdynamo操作を行う事ができる

-> [Usage](https://pynamodb.readthedocs.io/en/latest/quickstart.html)

## テーブルクラスとdynamoの設定
- クラスがどのテーブルに当たるかなどの設定はテーブルクラス直下に`Meta`というクラスを作成して、そのクラスに設定項目を書いていく

-> [Model Setting](https://pynamodb.readthedocs.io/en/latest/tutorial.html#model-settings)

```python
from pynamodb.models import Model
from pynamodb.attributes import UnicodeAttribute


class Thread(Model):
    class Meta:
        table_name = 'Thread'
        # Specifies the region
        region = 'us-west-1'
        # Optional: Specify the hostname only if it needs to be changed from the default AWS setting
        host = 'http://localhost'
        # Specifies the write capacity
        write_capacity_units = 10
        # Specifies the read capacity
        read_capacity_units = 10
    forum_name = UnicodeAttribute(hash_key=True)
```

### AWS接続情報の設定
- boto3のようにAWSアクセスのためにkeyやsecretが必要
- デフォルトでは`AWS_ACCESS_KEY_ID`や`AWS_SECRET_ACCESS_KEY`という名前の環境変数から読まれる
- その他`botocore`で設定できる項目はすべてpynamodbでもworkする

-> [AWS Access](https://pynamodb.readthedocs.io/en/latest/awsaccess.html#aws-access)

## テーブルクラスとdynamoのカラムのマッピング指定
- テーブルクラス直下にカラムの定義を書いていく
- カラムのデータ型を指定することができ、それに対応したattributeオブジェクトが用意されている
- 以下は`forum_name`というunicode型のプライマリハッシュキーカラムを持つ`Thread`というテーブルをクラスにマッピングする例

-> [Defining Model Attributes](https://pynamodb.readthedocs.io/en/latest/tutorial.html#defining-model-attributes)

```python
from pynamodb.models import Model
from pynamodb.attributes import UnicodeAttribute

def my_default_value():
    return 'My default value'

class Thread(Model):
    class Meta:
        table_name = 'Thread'
    forum_name = UnicodeAttribute(hash_key=True, default=my_default_value)
```

- カラムの型は様々用意されている

> UnicodeAttribute
UnicodeSetAttribute
NumberAttribute
NumberSetAttribute
BinaryAttribute
BinarySetAttribute
UTCDateTimeAttribute
BooleanAttribute
JSONAttribute
MapAttribute

## Attribute定義オブジェクトの使い方
### マッピング対象のdynamoのカラム名を指定する
- `attr_name`という引数を指定してattributeを作成するとその名前にマッピングされる

```python
NumberAttribute(attr_name="startAt") # => "startAt"というカラムにマップされる数値型データが定義される
```

## Listなどの複雑なデータを持つカラムの定義
- dynamoのカラムにはListやMapのような複雑なデータも入る
- そのようなデータを定義するために`ListAttribute`や`MapAttribute`というattributeオブジェクトが用意されている

-> [List Attributes](https://pynamodb.readthedocs.io/en/latest/attributes.html#list-attributes)
-> [Map Attributes](https://pynamodb.readthedocs.io/en/latest/attributes.html#map-attributes)

### List型カラムの定義
- 例えば以下では`groceries`というListが格納されるカラムを持つテーブルが作成される

```python
from pynamodb.attributes import ListAttribute, NumberAttribute, UnicodeAttribute

class GroceryList(Model):
    class Meta:
        table_name = 'GroceryListModel'

    store_name = UnicodeAttribute(hash_key=True)
    groceries = ListAttribute()

# Example usage:

GroceryList(store_name='Haight Street Market',
            groceries=['bread', 1, 'butter', 6, 'milk', 1]) # 入る値は何でもいい
```

### Map型カラムの定義
- `MapAttribute`を継承させたカスタムカラム型を実装して定義できる

```python
from pynamodb.attributes import MapAttribute, UnicodeAttribute

class CarInfoMap(MapAttribute):
    make = UnicodeAttribute(null=False)
    model = UnicodeAttribute(null=True)

class Car(Model):
    class Meta:
        table_name = "Car"
    info = CarIndoMap()
```

### 内部データ型まで定義されたList型カラムの定義
- ただListを定義するだけだと中身が何でもありになってしまう
- Mapと`of`という引数で組み合わせると中身の型まで指定できる

```python
from pynamodb.attributes import ListAttribute, NumberAttribute


class OfficeEmployeeMap(MapAttribute):
    office_employee_id = NumberAttribute()
    person = UnicodeAttribute()


class Office(Model):
    class Meta:
        table_name = 'OfficeModel'
    office_id = NumberAttribute(hash_key=True)
    employees = ListAttribute(of=OfficeEmployeeMap)

# Example usage:

emp1 = OfficeEmployeeMap(
    office_employee_id=123,
    person='justin'
)
emp2 = OfficeEmployeeMap(
    office_employee_id=125,
    person='lita'
)
emp4 = OfficeEmployeeMap(
    office_employee_id=126,
    person='garrett'
)

Office(
    office_id=3,
    employees=[emp1, emp2, emp3]
).save()  # persists

Office(
    office_id=3,
    employees=['justin', 'lita', 'garrett']
).save()  # raises ValueError
```

## Index
### 定義
- Indexクラスを継承させIndexを表すクラスを作成する
- カラムの様にモデルクラス直下に定義するとIndexとして使用できるようになる

-> [Global Secondary Indexes](https://pynamodb.readthedocs.io/en/latest/indexes.html#global-secondary-indexes)

```python
from pynamodb.indexes import GlobalSecondaryIndex, AllProjection
from pynamodb.attributes import NumberAttribute


class ViewIndex(GlobalSecondaryIndex):
    """
    This class represents a global secondary index
    """
    class Meta:
        # index_name is optional, but can be provided to override the default name
        index_name = 'foo-index'
        read_capacity_units = 2
        write_capacity_units = 1
        # All attributes are projected
        projection = AllProjection()

    # This attribute is the hash key for the index
    # Note that this attribute must also exist
    # in the model
    view = NumberAttribute(default=0, hash_key=True)

from pynamodb.models import Model
from pynamodb.attributes import UnicodeAttribute


class TestModel(Model):
    """
    A test model that uses a global secondary index
    """
    class Meta:
        table_name = 'TestModel'
    forum = UnicodeAttribute(hash_key=True)
    thread = UnicodeAttribute(range_key=True)
    view_index = ViewIndex()
    view = NumberAttribute(default=0)
```

### IndexのQuery
- indexを指定して、検索対象の値を入れ込むとクエリが飛ばせる

```python
for item in TestModel.view_index.query(1):
    print("Item queried from index: {0}".format(item))
```

## dynamoの操作
### Query
-> [Querying](https://pynamodb.readthedocs.io/en/latest/quickstart.html#querying)

- テーブルクラスに対して`.query`を呼び出すと定義されたプライマリキーで検索がかかる

```python
class UserModel(Model):
        """
        A DynamoDB User
        """
        class Meta:
            table_name = 'dynamodb-user'
        email = UnicodeAttribute()
        first_name = UnicodeAttribute(range_key=True)
        last_name = UnicodeAttribute(hash_key=True)

# "Smith"という名前のデータでfirst nameがJで始まるものが取れる
for user in UserModel.query('Smith', UserModel.first_name.startswith('J')):
    print(user.first_name)
```

#### Pagination
- dynamoは一括で取れるデータの容量が制限されている
- pynamodbではforで回すと中身のiteratorが再帰的に全部のデータを取ってきてくれる
- この挙動はIndexでのQueryなどでも同じ

### 操作の条件指定
- dynamoでサポートされている条件指定はすべて使用できる
- attributeオブジェクトに対してメソッドを呼ぶことで条件オブジェクトが作成できる
- 以下の対応表にすべての条件指定と例がまとまっているのでとても有用

-> [Condition Expressions](https://pynamodb.readthedocs.io/en/latest/conditional.html#condition-expressions)
