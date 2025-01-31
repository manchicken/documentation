---
dependencies: []
disable_edit: true
kind: documentation
title: 読み取り用のオープンフラグを定義しない
---
## メタデータ
**ID:** `python-best-practices/open-add-flag`

**言語:** Python

**重大度:** エラー

**カテゴリー:** ベストプラクティス

## 説明
フラグが読み取り専用に開かれている場合は、ファイルを開くフラグを付ける必要はありません。


#### 詳細はこちら

 - [Python ドキュメント: `open()`](https://docs.python.org/3/library/functions.html#open)

## 非準拠コードの例
```python
def print_foo():
  with open("myfile.txt", "r") as myfile:  # "r" フラグの指定は不要
    content = myfile.read()
```

## 準拠コードの例
```python
def print_foo():
  with open("myfile.txt") as myfile:
    content = myfile.read()
```