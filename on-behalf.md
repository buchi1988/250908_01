# 「On-Behalf 3DEXPERIENCE Authentication」の要点と実装手順（詳説）

## これは何を解決する仕組み？

3DEXPERIENCE（3DX）のデータへアクセスする際、本来はユーザー本人のID/パスワードで3DPassportにログインが必要です。
**On-Behalf（代理）認証**は、\*\*「バッチサービス」\*\*として登録したクライアントが、**ユーザーの資格情報を知らなくても、そのユーザーになり代わって** CAS セッションを確立し、3DX各サービス（例：3DSpace、3DSwym）にログインできる仕組みです。トークンは短命（通常1分）で一回限りの使用に限定されます。&#x20;

---

## 用語ミニ辞典

* **バッチサービス**：管理者が3DPassport Control Centerで登録する“機械アカウント”。登録時に\*\*サービス秘密鍵（service secret key）\*\*が1度だけ表示されます。
* **Transient token（トランジエントトークン）**：ユーザーID（ユーザー名またはメール）とサービス名＋秘密鍵で取得する**一次トークン**。**1回だけ使用可・短時間で失効**。
* **CAS / CASTGC / TGT / ST**：CASは中央認証。ログインするとブラウザ（またはHTTPクライアント）が**CASTGC**クッキー（TGTのハンドル）を保持し、各サービスごとに\*\*ST（Service Ticket）\*\*を発行してもらってログインします。&#x20;

---

## 事前準備（管理UI）

1. 3DPassport Control Center → **Integration** → **Batch Services**。
2. **Service Name** を作成すると**Service Secret**が**1回だけ**表示されるので安全に保管（忘れたら**新しい鍵を再発行**）。
3. 必要に応じて**Service authentication**と個別の**Active**トグルをONに。**OFFだとトランジエントトークンは発行されません**。管理者は個別/全体で無効化できます。&#x20;

---

## APIフロー（ページ2のシーケンス図の読み解き）

### ① トランジエントトークンの取得（Get On-Behalf Transient Token）

* **HTTP**：`GET <3DP_URL>/api/v2/batch/ticket?identifier=<USER>|<EMAIL>[&service=<3DEXP_SERVICE_URL>]`
* **ヘッダー**：

  * `DS-SERVICE-NAME: <バッチサービス名>`
  * `DS-SERVICE-SECRET: <サービス秘密鍵>`
* **レスポンス**：JSON（`Accept: application/json`を付ける）で

  * `transient_token`
  * 後続のCASログインに使う**完全なURL**（`/api/login/cas/transient?tgt=...`）
  * **有効秒数（通常≈60秒）**
* **注意**：`identifier` と `service` は**URLエンコード必須**。トークンは**使い捨て**。&#x20;

### ② CASへ「トランジエントトークン」でログイン

* **サービスURLを渡さない**場合：
  `GET <3DP_URL>/api/login/cas/transient?tgt=<transient_token>`

  * 成功するとJSON（`Accept: application/json`あり）または`302`＋「my-profile」へ。
  * \*\*CASTGC（TGT）\*\*が発行され、以後はこのSSOセッションを使えます。
* **最初からサービスURLを渡す**場合：
  `GET <3DP_URL>/api/login/cas/transient?tgt=<transient_token>&service=<3DEXP_SERVICE_URL>`

  * `302`で `<SERVICE_URL>?ticket=ST-xxx` にリダイレクト。サービス側がSTを検証してログイン成立。&#x20;

### ③ 追加サービスへのログイン（ページ3の図）

SSO（CASTGC）を持っていれば、**各サービスごとにSTを取りに行く**だけです：
`GET <3DP_URL>/login?service=<3DEXP_LOGIN_URL>` → `302`で `?ticket=ST-xxx` が付いたサービスURLへ。サービスがST検証後にログイン完了。&#x20;

---

## 有効期限と制限

* **トランジエントトークン**：**≈1分**・**1回限り使用可**。消費したら再利用不可。
* **セッション存続期間**：CASに入った後は**CASTGCの有効期間**に従う。
* **提供形態**：**オンプレ／プライベートクラウドのみ**。
* **R2022x FD04以降**：**管理者ロールの人に対してもトークン発行が可能**になり、\*\*すべてのメンバー（管理者か否かを問わず）\*\*が代理認証の対象になりました。取り扱いに十分注意してください。&#x20;

---

## 典型的なHTTP例（cURL）

**1) トランジエントトークン取得**

```bash
curl -G "$3DP_URL/api/v2/batch/ticket" \
  --data-urlencode "identifier=$USER_OR_EMAIL" \
  --data-urlencode "service=$SERVICE_URL" \
  -H "DS-SERVICE-NAME: $BATCH_SERVICE_NAME" \
  -H "DS-SERVICE-SECRET: $BATCH_SERVICE_SECRET" \
  -H "Accept: application/json"
```

**2) CASへログイン（サービス同時指定）**

```bash
curl -i "$3DP_URL/api/login/cas/transient?tgt=$TRANSIENT_TOKEN&service=$SERVICE_URL"
# 302 -> $SERVICE_URL?ticket=ST-xxx（サービス側でST検証）
```

**3) 追加サービス**

```bash
curl -i "$3DP_URL/login?service=$ANOTHER_SERVICE_URL"
# 302 -> ?ticket=ST-yyy
```

（ヘッダー名は文書中で`DS-SERVICE-NAME/SECRET`と表記。HTTPヘッダーは大文字小文字非依存ですが、文書表記に合わせるのが無難です。）&#x20;

---

## 実装時の落とし穴と対策

* **トグルがOFF**（Service authentication / Batch ServiceのActive）→ トークンが発行されない。
* **URLエンコード漏れ**（identifier・service）→ 400/無効URL。
* **Acceptヘッダー未指定** → 期待と異なる`302`/HTML応答に。
* **トークン期限切れ**（≈60秒）→ 401/無効。**取得→即ログイン**の実装に。
* **秘密鍵の管理**：初回しか見えません。**保管・ローテーション**の運用設計を。
* **セッション管理**：CASTGCは**長め**。クライアント側で**Cookie保護**・**不要時は明示ログアウト**。&#x20;

---

## セキュリティ運用の勘所

* **最小権限**：バッチサービスに与える権限・到達先ネットワークを最小化。
* **監査**：誰を代理認証したかのログを残す。
* **鍵ローテーション／失効**：漏えい疑い時は**即Deactivate**→**鍵再発行**。
* **管理者アカウント代理認証の扱い**：R2022xFD04以降は特に厳重に。&#x20;

---

## まとめ

1. 管理UIで**バッチサービス名＋秘密鍵**を用意 → 2) `identifier`（ユーザー名/メール）で**トランジエントトークンを取得** → 3) `/api/login/cas/transient`で**CASセッション（CASTGC）確立** → 4) 各サービスへ**STでログイン**。
   図（**ページ2・3**）は上記の\*\*2パターン（サービス同時指定/後から指定）\*\*を丁寧に示しています。実装は上のcURL例を骨子にすればスムーズです。&#x20;
