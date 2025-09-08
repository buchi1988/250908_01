# 指定したドキュメントに紐づく「ファイルのメタデータ」を取得するAPI

# エンドポイント

**GET /resources/v1/modeler/documents/{docId}/files**
与えた `docId`（検索API等で得たドキュメントID）の**添付ファイル一覧と各ファイルのメタ情報**を返します。
※返すのは**バイナリ本体ではなくメタデータ**です（ダウンロードは別エンドポイント）。

# パラメータ解説

* **docId（必須／path）**
  対象ドキュメントの識別子。検索APIの結果に出てくるIDを渡します。

* **\$include（任意／query）**
  レスポンスに含める**サブリソース**をカンマ区切りで指定します。
  例：`ownerInfo, lockerInfo, versions`
  画像の `[]ownerInfo / []lockerInfo / []versions` という表記は、**配列要素（＝各ファイル）ごとに**その情報を展開することを示す記法です。

  * `ownerInfo` … 作成者／所有者情報
  * `lockerInfo` … チェックアウト／ロック情報
  * `versions` … ファイルの版情報（過去版の一覧など）

* **\$fields（任意／query）**
  返却フィールドの絞り込み。`all` / `none` のほか、必要なペイロード項目だけを指定できます（正確なフィールド名はスキーマに従ってください）。
  レスポンスを軽量にしたいときに使います。

* **SecurityContext（任意／header）**
  実行時のセキュリティ／コラボラティブスペース等のコンテキスト。権限や見える範囲に影響します。

* **Accept-Language（任意／header）**
  ロケール指定（例：`ja-JP`, `en-US`）。

# 使い方の例

**1) 基本：ファイル一覧＋バージョン情報も欲しい**

```
GET /resources/v1/modeler/documents/1234567890ABCDEF/files
  ?$include=versions
```

**2) ファイル一覧と所有者・ロック情報を含め、返却項目を最小化**

```
GET /resources/v1/modeler/documents/1234567890ABCDEF/files
  ?$include=ownerInfo,lockerInfo
  &$fields=files,ownerInfo,lockerInfo
```

（`$fields` の具体的な項目名はスキーマに合わせて列挙してください）

**3) ヘッダーも付ける例**

```
Accept-Language: ja-JP
SecurityContext: <あなたのコンテキスト文字列>
```

# よくある流れ

1. `/documents/search` で目的のドキュメントを見つけて `docId` を取得
2. 本エンドポイント `/documents/{docId}/files` で**添付ファイルのメタデータ**（名前、タイプ、版、所有者など）を取得
3. 実ファイルの**ダウンロード**が必要なら、別の「ファイル本体取得」系エンドポイントに、ここで得たファイルIDを渡す

# コツ

* **\$include と \$fields を併用**して、必要な情報だけ返すと高速・省容量。
* 版管理やチェックアウト状況をUIに出したい場合は、`versions` / `lockerInfo` を `$include` に加える。
* ページング用の `$top/$skip` はこの画面には無い＝**1ドキュメント内のファイル一覧をまとめて返す**想定です。