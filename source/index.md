# APIのテストデータを自動生成できるSchemathesisの紹介

PyCon mini 東海 2025資料

```{raw} html

<a rel="license" href="http://creativecommons.org/licenses/by/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by/4.0/88x31.png" /></a>
<small>This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/4.0/">Creative Commons Attribution 4.0 International License</a>.</small>
```

## はじめに

### 自己紹介

* Ryuji Tsutsui@ryu22e
* さくらインターネット株式会社所属
* Python歴は14年くらい（主にDjango）
* Python Boot Camp、Shonan.py、GCPUG Shonanなどコミュニティ活動もしています
* 著書（共著）：『[Python実践レシピ](https://gihyo.jp/book/2022/978-4-297-12576-9)』

### 今​日話すこと

* Schemathesisの概要
* Schemathesisの使い方
* Django + Schemathesisの例

### 今日話さないこと

* Pythonの文法
* Pythonのユニットテストの書き方

### この​トークの​対象者

* Pythonのユニットテストを書いたことがある人

### この​トークで​得られる​こと

* Schemathesisの概要、使い方
* Hypothesisの概要
* 「プロパティベーステスト」の概要
* Schemathesisがどのような場面で役立つのか

### トークの構成

* Schemathesisの概要
* Hypothesisの概要
* コード例紹介

## Schemathesisの概要

### Schemathesisとは何か

Schemathesisとは、OpenAPIまたはGraphQLで書かれたAPI仕様を元にテストデータを自動生成し、実際にそれらのテストデータを使ってAPIを呼び出すことで、仕様通りの挙動になっているか検証できるツール。

### どんなとき役立つか

* 人間が書いたテストコードでは気付けないエッジケースを見つけられる
* 仕様書と実装の乖離に気づくことができる

### Schemathesisの簡単な使い方

使い方は2通り：

1. pytest経由で使う
2. コマンドラインインターフェースで使う

## Hypothesisの概要

Hypothesisとは、Schemathesisの内部で使われている「プロパティベーステスト」を行うためのテストフレームワークです。
ここでは、以下について説明します。

* Hypothesisの簡単な使い方
* 「プロパティベーステスト」がどのような考え方のテスト手法なのか

### テスト対象のアプリケーション

以下のFastAPIアプリケーションを用意する。

```{revealjs-code-block} python
import fastapi
from pydantic import BaseModel, StrictInt

app = fastapi.FastAPI()

class Values(BaseModel):
    a: StrictInt
    b: StrictInt

@app.post("/div")
async def div(values: Values):
    """2つの整数を受け取り、その商を返すAPIエンドポイント"""
    # 0で除算するケースを考慮していないのは「バグ」。
    return {"result": values.a / values.b}
```

### テストコード

```{revealjs-code-block} python
import schemathesis

# (1)OpenAPI仕様書のURLを指定してスキーマを生成
schema = schemathesis.openapi.from_url("http://127.0.0.1:8000/openapi.json")

# (2)スキーマと戦略の定義に基づいてテストケースを生成
@schema.parametrize()
def test_api(case):
    """pytestでAPIのテストを実行する関数"""
    # (3)Schemathesisが生成したテストケースを使用してAPIを呼び出し、
    # 検証を行う
    case.call_and_validate()
```

### テスト実行結果

```{revealjs-code-block} bash
$ pytest test_main.py -v
============================= test session starts ==============================
（省略）
test_main.py::test_api[POST /div] FAILED                                 [100%]

=================================== FAILURES ===================================
_____________________________ test_api[POST /div] ______________________________
+ Exception Group Traceback (most recent call last):
（省略）
  | - Server error
  | 
  | - Undocumented HTTP status code
  | 
  |     Received: 500
  |     Documented: 200, 422
  | 
  | [500] Internal Server Error:
  | 
  |     `Internal Server Error`
  | 
  | Reproduce with: 
  | 
  |     curl -X POST -H 'Content-Type: application/json' -d '{"a": 0, "b": 0}' http://127.0.0.1:8000/div
  | 
  |  (2 sub-exceptions)
  +-+---------------- 1 ----------------
    | schemathesis.core.failures.ServerError: Server error
    +---------------- 2 ----------------
    | schemathesis.openapi.checks.UndefinedStatusCode: Undocumented HTTP status code
    | 
    | Received: 500
    | Documented: 200, 422
    +------------------------------------
=========================== short test summary info ============================
FAILED test_main.py::test_api[POST /div]
============================== 1 failed in 0.38s ===============================
```

### アプリケーションを修正

`b`が0のときバリデーションエラーになるよう修正。

```{revealjs-code-block} python
import fastapi
from pydantic import BaseModel, Field, StrictInt  # (1)Fieldをインポート

app = fastapi.FastAPI()

class Values(BaseModel):
    a: StrictInt
    # (2)Fieldを使用してOpenAPIの拡張を追加
    b: StrictInt = Field(
        ...,
        description="0以外の整数",
        json_schema_extra={
            "not": {"const": 0}  # OpenAPI拡張：b != 0 の意味
        }
    )

@app.post("/div")
async def div(values: Values):
    """2つの整数を受け取り、その商を返すAPIエンドポイント"""
    return {"result": values.a / values.b}
```

### OpenAPIスキーマの内容

TODO スクショを貼る

## コード例紹介

Django(+ Django REST framework）製のAPIとSchemathesisを組み合わせてみる。

### Djangoアプリケーションを用意する

以下のライブラリを使用。

* Django REST framework
* drf-spectacular（OpenAPIスキーマを生成する）

### GitHubリポジトリの紹介

以下のリポジトリを元に説明する。

TODO URLを貼る

### なぜテストが通らないのか？

drf-spectacularが生成したOpenAPIスキーマのせい。

### drf-spectacularの何が問題か

* drf-spectacularは異常終了を考慮しない
* SchemathesisはOpenAPIスキーマが絶対正しいという前提
* 「そうか、どんな入力値でも絶対に200 OKなんだな！」と考える
* でも、実際にはそうではないのでコケる

### どうすればいいか

drf-standardized-errorsを導入する。

## 「それで、結局Schemathesisがあれば人間はテストコードを書かなくてもよくなるの？」

Schemathesisはテストコードを自動生成してくれる。
つまり、人間がテストコードを書かなくてもテストが機能するのでは？

### 結論から言うと

そんなことはない😇

### Schemathesisのテストの問題点

* どんなテストが実行されるのかコントロールできない
* テストコードからアプリケーションの仕様を読み取れなくなる

### Schemathesisの使い所

テストコードは従来通り書きつつ、最初に挙げた以下の2点の利点を活かすのが良い。

* 人間が書いたテストコードでは気付けないエッジケースを見つけられる
* 仕様書と実装の乖離に気づくことができる
