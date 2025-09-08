了解！Talend API Tester で「3DEXPERIENCE Web API にログイン → キーワードでドキュメント検索 → そのドキュメント配下のファイルID取得」までを、最短ルートでまとめます。

---

# 0) 事前に用意する変数（Talend の Environment 推奨）

* `3DP_URL`：3DPassport 基本URL（例: `https://3dpassport.example.com`）
* `SPACE_URL`：3DSpace 基本URL（例: `https://3dspace.example.com/3dspace`）
* `BATCH_SERVICE_NAME` / `BATCH_SERVICE_SECRET`：On-Behalf 用バッチサービス名とシークレット
* `IMPERSONATE_USER`：代理ログインするユーザー（ログイン名 or メール）
* `SECURITY_CONTEXT`：表示権限に影響（例：あなたの組織／コラボスペースを示す値）
* `LANG`：`ja-JP` など
  On-Behalf 認証の仕組み・注意点（トークンは≒1分の使い捨て等）はこちらの要点が参考になります。&#x20;

---

# 1) On-Behalf で SSO（最短2リクエスト）

**(1) トランジエントトークン取得**

```
GET  {{3DP_URL}}/api/v2/batch/ticket
  ?identifier={{IMPERSONATE_USER}}
  &service={{SPACE_URL}}
Headers:
  DS-SERVICE-NAME: {{BATCH_SERVICE_NAME}}
  DS-SERVICE-SECRET: {{BATCH_SERVICE_SECRET}}
  Accept: application/json
```

レスポンスの `transient_token` を抽出（Talend → **Assertions & extractions** で JSONPath `$.transient_token` など）。&#x20;

**(2) CAS にログイン（3DSpace へ遷移）**

```
GET  {{3DP_URL}}/api/login/cas/transient
  ?tgt={{transient_token}}
  &service={{SPACE_URL}}
```

Talend では「Follow redirects」「Preserve cookies」を ON。これで 3DSpace への SSO セッションが確立されます（以降の API が通るようになります）。&#x20;

> メモ：以降の GET は CSRF 不要ですが、レスポンスに CSRF が返る場合は変更系で使うため保持しておくと便利（サンプルに `csrf.name/value` が含まれます）。

---

# 2) キーワードでドキュメント検索（ファイルIDまで一気取り）

**(推奨) 検索で files を同梱して返させる**

```
GET  {{SPACE_URL}}/resources/v1/modeler/documents/search
  ?searchStr={{KEYWORD}}
  &$top=50&$skip=0
  &$include=files
  &$fields=identifier,title,relateddata.files
Headers:
  SecurityContext: {{SECURITY_CONTEXT}}
  Accept-Language: {{LANG}}
```

* `searchStr` は 2文字以上、`A+B` で AND 検索。`$top/$skip` でページング（最大50件に注意）。&#x20;
* レスポンス構造：`data[].identifier` が**ドキュメントID**、`data[].relateddata.files[].id` が**ファイルID**。Talend の抽出で

  * `docId` = `$.data[0].identifier`
  * `fileIds` = `$.data[0].relateddata.files[*].id`
    を変数に保存すれば、そのまま使えます。検索結果に `files` を含められる点と返却例はこの仕様／サンプルが示しています。

> 例：ヒットしたドキュメントの `relateddata.files` に `id`, `dataelements.title`（例「Fruits.docx」）等がぶら下がって返ります。&#x20;

---

# 3) 別法：docId が分かった後にファイル一覧APIで取得

検索で `files` を付けなかった場合や、詳細（版情報など）が欲しい場合は **docId → files API**：

```
GET  {{SPACE_URL}}/resources/v1/modeler/documents/{{docId}}/files
  ?$include=ownerInfo,versions
  &$fields=id,dataelements.title,relateddata.versions
Headers:
  SecurityContext: {{SECURITY_CONTEXT}}
  Accept-Language: {{LANG}}
```

* **ファイルID**は `$.data[*].id` で取得可能。
* `ownerInfo` / `lockerInfo` / `versions` を `$include` で拡張できます。使い方・パラメータの意味はこの説明がまとまっています。&#x20;
* 返却の実例（`title: "Fruits.docx"`, `fileChecksum`, `fileSize` などのメタ）はサンプルが参考になります。&#x20;

---

# 4) Talend API Tester での実装ヒント

* **Environment 変数**を上記で作成 → リクエストは変数置換で管理。
* **Extract**（JSONPath）で `transient_token` / `docId` / `fileIds[]` を保存。
* **Follow redirects / Preserve cookies** を ON（SSO 後続リクエストのため）。
* **ヘッダー**：`SecurityContext` と `Accept-Language` を都度付与。検索API/ファイルAPIとも対応しています。
* **CSRF**：GET では不要。PUT/POST 時に、レスポンスで得た `ENO_CSRF_TOKEN` を送り返す運用に。

---

# 5) つまずきやすいポイント

* **\$top は実質 50 件上限**の可能性があるため、`$skip` と併用してページングする。&#x20;
* **On-Behalf のトークンは ≒60秒・使い捨て**。取得→即ログイン。`identifier`/`service` は URL エンコード必須。&#x20;
* **SecurityContext** を誤ると見える結果が変わる／空になる。検索・ファイルAPIともヘッダーで渡せます。

---

この流れ（①On-Behalf SSO → ②`/documents/search` で `files` 同梱 → ③JSONPath 抽出）で、**1クエリでファイルIDまで**取れるはずです。もし「版の履歴も」「ダウンロードURLまで」などの要件があれば、続きのエンドポイント設計もすぐ書きます！
