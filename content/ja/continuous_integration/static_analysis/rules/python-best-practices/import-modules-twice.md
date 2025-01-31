---
dependencies: []
disable_edit: true
kind: documentation
title: モジュールが 2 度インポートされた
---
## メタデータ
**ID:** `python-best-practices/import-modules-twice`

**言語:** Python

**重大度:** 警告

**カテゴリー:** ベストプラクティス

## 説明
モジュールの定義は常に一度だけにしてください。モジュールを何度もインポートしたり、異なるメソッドを使ってインポートしたりしないでください。コードがわかりにくくなります。1 つのインポートメカニズムだけで、モジュールを一度インポートします。

## 非準拠コードの例
```python
import logging
import logging  # 同じモジュールを再度インポートしないでください。

```

```python
import logging
from logging import foo # 同じモジュールを再度インポートしないでください。

```

## 準拠コードの例
```python
import logging
```