# ドキュメント検索API」のエンドポイント仕様

* **GET /resources/v1/modeler/documents/search**
  文字列（`searchStr`）を入力に、**ドキュメントのメタデータ**と**添付ファイル本文**（インデックス済み）を全文検索します。

# 主なパラメータ

* **`searchStr`（必須）**: 2文字以上。複数条件は `+` で **AND**。
  例: `TitleOfDocument+(current:Frozen)`（タイトルと状態=Frozen を両方満たす）
* **`$top`**: 返す件数。デフォルト **50**、最大 **1000** と記載。ただし画面上部の注記に **「1回の呼び出しで最大50件」** とあるため、実際は50件で打ち切られる可能性があります。
* **`$skip`**: 先頭から何件スキップするか（ページング用）。`$top` と組み合わせます。
* **`$include`**: 一緒に返す**サブリソース**をカンマ区切りで指定。`none` / `all` のほか、例として
  `ownerInfo, originatorInfo, parents, files, sovaccess, relOwnerInfo, lockerInfo, versions` など。
* **`$fields`**: 返却フィールドを絞り込み（`none` / `all` または個別フィールド名）。
* **ヘッダー**

  * **`SecurityContext`**: 実行コンテキスト（ロール/コラボスペースなど）を指定。
  * **`Accept-Language`**: レスポンスのロケール。

# 使い方の例

1. **条件検索＋ページング＋付加情報**

```
GET /resources/v1/modeler/documents/search
  ?searchStr=TitleOfDocument+(current:Frozen)
  &$top=30&$skip=10
  &$include=files,versions
```

→ タイトル一致かつ状態 Frozen の結果を、11件目から30件取得。ファイル情報と版情報も返す。

2. **必要フィールドだけ返す（軽量化）**

```
...?searchStr=pump
  &$top=50&$skip=0
  &$include=ownerInfo,lockerInfo
  &$fields=physicalId,title,current
```

3. **ページング**

* 1ページ目: `?$top=50&$skip=0`
* 2ページ目: `?$top=50&$skip=50`

# コツ・注意点

* 検索対象は**メタデータ＋本文**（インデックス済みフィールド）。
* 大量件数は\*\*`$top` と `$skip` でページング\*\*。注記上限（50件/呼び出し）に注意。
* `+` は AND 条件。OR/NOT の表記はこの画面だけでは不明。
* **`$include` と `$fields` を併用**すると、必要最小限の情報だけを返せて効率的です。