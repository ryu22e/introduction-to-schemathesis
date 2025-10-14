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

## コード例紹介

Django(+ Django REST framework）製のAPIとSchemathesisを組み合わせてみよう

私が普段プロダクト開発に使っているDjango(+ Django REST framework）でAPIを作り、Schemathesisでテストを行うサンプルコードを作りました（このサンプルコードはSchemathesisの公式ドキュメントには載っていない、私が独自に調査して作ったものです）。
このコードの内容を交えて、設定内容やハマりどころについて説明します。

## 「それで、結局Schemathesisがあれば人間はテストコードを書かなくてもよくなるの？」

Schemathesisはテストコードを自動生成してくれるため、人間がテストコードを書かなくてもテストが機能するのでは？ と考える人がいるかもしれません。しかし、現実は甘くありません……。Schemathesisのテストでは十分テストしきれないケースも存在します。Schemathesisは、あくまで人間が書くテストコードがカバーしきれない部分を保管する、補助的なツールと考えたほうがいいでしょう。
ここでは、テストコードを書かずにSchemathesisのテストだけで済ませた場合、どんな不都合が起こり得るのかを説明します。
