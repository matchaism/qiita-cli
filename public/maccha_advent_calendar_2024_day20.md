---
title: Pythonにおけるオーバーロードのためにmultipledispatchを使う / maccha Advent Calendar 2024
tags:
  - Python
  - オーバーロード
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
# Pythonにおけるオーバーロードのためにmultipledispatchを使う

[maccha Advent Calendar 2024](https://adventar.org/calendars/10199)の20日目の記事です．

Pythonでプログラミングをしていると，同じ名前の関数やメソッドを引数の型や個数に応じて実装したい場面があります．そうした「オーバーロード」のための手法と注意点について紹介します．

## オーバーロードとは

オーバーロードとは，同じ名前の関数やメソッドを引数の型や個数に応じて異なる実装をすることです．C++など一部のプログラミング言語では，標準機能としてサポートされています．

## Pythonでのオーバーロードの手法

Pythonでオーバーロードを実現する方法は主に以下の3つです．それぞれの特徴と注意点を見ていきます．

### 静的型チェックのみの`@overload`

Pythonの`typing`モジュールが提供する`@overload`デコレータは，型ヒントによる静的型チェックを補助しますが，実際の処理分岐は行いません．

```python
from typing import overload

@overload
def hoge(x: int) -> str: pass

@overload
def hoge(x: float) -> str: pass

def hoge(x: int | float) -> str:
  if isinstance(x, int):
    return "This is int."
  elif isinstance(x, float):
    return "This is float."
  return "What is this?"
```

上記のコードでは`hoge`関数の3つの実装のうち，実際に実行されるのは最後のものだけです．上2つは`@overload`による型チェックのための実装です．

また同じ名前の関数やメソッドを実装するとしても，引数の型や個数に応じて異なる実装をしたいことがあります．

### 第1引数の型に応じて処理を分岐する`@singledispatchmethod`

`functools`モジュールの`@singledispatchmethod`を使うと，第1引数の型に応じて処理を切り替えることができます．

```python
from functools import singledispatchmethod

class Ceiling:
  @singledispatchmethod
  def __call__(self, x):
    raise TypeError(f"Unsupported type: {type(x)}")
  
  @__call__.register
  def _(self, x: int):
    return x
    
  @__call__.register
  def _(self, x: float):
    return int(x) if x==int(x) else int(x)+1
```

この例では，ceilingメソッドが引数の型に応じた処理を提供しています．ただし`@singledispatchmethod`は**第1引数のみ**を基に動作するため，複数の引数の型に応じた処理分岐には対応していません．

### 複数の引数の型に応じて処理を分岐してくれる`@singledispatchmethod`

`multipledispatch`ライブラリを使うと，**複数の引数の型**に応じて処理を切り替えることができます．

```python
from multipledispatch import dispatch

class MyClass:
  @dispatch(int, int)
  def get_type_of_sum(self, x, y):
    return "int"

  @dispatch(float, float)
  def get_type_of_sum(self, x, y):
    return "float"

  @dispatch(int, float)
  def get_type_of_sum(self, x, y):
    return "float"

  @dispatch(float, int)
  def get_type_of_sum(self, x, y):
    return "float"
```

この方法では引数の型や個数に応じた柔軟な処理分岐が可能です．

ただし，リストやタプルなどの型に対しては中身の要素の型までチェックされません．`list[int]`や`tuple[int, float]`のような構造的な型に対するサポートはありません．

## まとめ

以上の点をまとめます．

|手法|使い道|
|:---|:---|
|`typing.overload`|引数の型チェックのみ|
|`functools.singledispatchmethod`|第1引数の型に応じて処理を分岐|
|`multipledispatch.dispatch`|複数の引数の型に応じて処理を分岐|

## 参考

- [typing.overloadのdocumentation](https://docs.python.org/ja/3/library/typing.html#typing.overload)
- [functools.singledispatchのdocumentation](https://docs.python.org/ja/3/library/functools.html#functools.singledispatch)
- [Multiple Dispatchのdocumentation](https://multiple-dispatch.readthedocs.io/en/latest/)
