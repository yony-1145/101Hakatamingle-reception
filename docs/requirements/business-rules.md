# ビジネスルール・判定ロジック

検証ルール・判定ロジック・運用ルールを集約。テーブル定義は [data-schema.md](./data-schema.md)、機能の能力一覧は [features.md](./features.md) を参照。

## QRコード仕様

### 生成・配布

- 方式: 静的QRコード + サーバー側での時間窓検証
- URL構造: `https://app.example.com/checkin/{eventId}?token={署名済みトークン}`
- 署名済みトークン: `HMAC-SHA256(eventId + secret)` で生成
- イベント作成時に1回生成し、`events.checkinToken`に保存
- 会場にA4などで印刷して掲示（運営の端末不要）

### QR再発行（流出時の対応）

- 管理画面から実行
- 新しい`checkinToken`を生成し`events.checkinToken`を上書き
- 古いQRはそれ以降無効化
- 監査ログに`checkin_token_regenerated`として記録

## チェックイン検証ルール

### 前提条件

- ログイン済みであること（userId存在）
- 未ログイン（新規・既存問わず）: ログイン画面へリダイレクト（元QR URLを保持）。ログイン / ゲストとして参加 / 新規会員登録 の3リンクを表示。「ゲストとして参加」は会場QR経由（リダイレクトに `?from=qr` 等のフラグ）の場合のみ表示

### 検証ルール（すべて満たす必要あり）

1. `token`がHMAC検証で正しい
2. 現在時刻が **開催開始30分前 〜 開催終了時刻** の範囲内
3. ユーザーが該当イベントに登録済み（`registrations`に有効なレコードあり）
4. `paymentStatus = 'paid'`
5. `paymentMethod != 'cash'`（現金払いはセルフチェックイン不可。MVP以降で追加される事前決済手段にも自動対応）
6. `checkedIn = false`（二重チェックイン防止）

**チェックイン可能時間窓:** `startDateTime - 30分` 〜 `endDateTime`

### 検証通過時のアクション

- `checkedIn = true`, `checkedInAt = 現在時刻` を記録
- 完了画面を表示（名前・イベント名・日程・Paid済み）
- スタッフ受付画面に「〇〇さんがチェックインしました」通知
- 監査ログに`self_checkin`として記録

### 検証非通過時の挙動

| 非通過理由 | 挙動 |
| --- | --- |
| 未ログイン | ログイン画面へリダイレクト（元QR URLを保持）。ログイン / ゲストとして参加（会場QR経由時のみ） / 新規会員登録 の3リンク表示 |
| 時間窓外 | 「チェックイン可能時間外です」 |
| 未決済または現金払い（ルール4 or 5 非通過） | 時間窓内: **自動でスタッフ受付画面に「〇〇さんが受付待ちです」通知**を発火（`registrations.lastNotifiedAt`更新）。参加者画面は「受付待ち中... スタッフが対応します」を表示。`lastNotifiedAt`が5分以内なら通知をスキップ（dedup）。時間窓外: 「チェックイン可能時間外です」 |
| 未登録 | 「このイベントにはまだ登録されていません」＋「このイベントに登録する」ボタン → 登録フローへ |
| 二重チェックイン | 「既にチェックイン済みです」 |

## チェックイン方式（ユーザー種別別）

### QRセルフチェックイン（事前オンライン決済会員）

QRスキャン → サーバー検証通過後、即座に`checkedIn=true`, `checkedInAt=現在時刻`を記録 → 完了画面表示 → スタッフ受付画面に「〇〇さんがチェックインしました」通知（ボタン操作不要） → 監査ログに`self_checkin`として記録

### ゲスト・現金払いユーザー

QRスキャン（会員） / マイページの「チェックインする」ボタン（会員） / ゲスト当日登録完了直後の自動発火（ゲスト） → サーバー検証で現金/unpaid判定 → `lastNotifiedAt`を更新（5分dedup） → スタッフ受付画面に「〇〇さんが受付待ちです」通知 → スタッフが確認 → 現金受取（必要な場合） → 「チェックイン完了」ボタン → `checkedIn=true`, `checkedInAt=現在時刻`を記録 → 監査ログに`manual_checkin`として記録

## 参加状況の表示（マイページ）

会員マイページのみで表示。ゲストはマイページ非対応のため対象外。

| 表示ステータス | 条件 |
| --- | --- |
| 参加予定 | checkedIn=false + チェックイン時間窓前 + stripe/paid + refundStatus IS NULL |
| 参加予定（当日現金払い） | checkedIn=false + 時間窓前 + cash |
| チェックイン待ち（QRスキャン） | checkedIn=false + 時間窓内 + stripe/paid + refundStatus IS NULL |
| チェックインする（ボタン） | checkedIn=false + 時間窓内 + cash + lastNotifiedAt未設定 or 5分以上前 |
| 受付待ち中... | checkedIn=false + 時間窓内 + cash + lastNotifiedAt 5分以内 |
| チェックイン済み | checkedIn=true |
| キャンセル済み | cancelledAt IS NOT NULL |
| 返金処理中 | cancelledAt IS NOT NULL + paymentStatus='paid' + refundStatus='pending' |
| 返金失敗 | cancelledAt IS NOT NULL + paymentStatus='paid' + refundStatus='failed'。「返金手続きに遅延が生じています。○○（運営メアド）までお問い合わせください」と表示 |
| 返金済み | paymentStatus='refunded' |

## 受付可否判定ロジック

```
有効な登録数 = registrations.count(WHERE eventId = X AND cancelledAt IS NULL)

受付可能 = (
  events.registrationOpen = true AND
  events.status = 'published' AND
  (有効な登録数 < events.capacity OR events.allowOverCapacity = true) AND
  現在時刻 < events.endDateTime
)
```

- `registrationOpen = false`（運営が手動停止）: 「受付を締め切りました」と表示
- `有効な登録数 >= capacity` かつ `allowOverCapacity = false`: 「満員のため受付を締め切りました」と表示
- キャンセルで有効な登録数が減少した場合は自動的に再受付となる

## キャンセル・返金判定ロジック

### 参加者キャンセル

```
キャンセル可否:
  現在時刻 < startDateTime かつ cancelledAt IS NULL
  → キャンセル可能

キャンセル成立時の返金可否:
  events.refundable = true AND
  現在時刻 < events.refundDeadline AND
  paymentStatus = 'paid'
  → 返金対象

返金実行:
  paymentMethod = 'stripe': refundStatus = 'pending' → Stripe APIで自動返金 → 成功: paymentStatus = 'refunded'、refundStatus = 'succeeded'、refundedAt / refundedAmount を記録。失敗: refundStatus = 'failed'（管理画面で手動対応）
  paymentMethod = 'cash': 運営が管理画面から手動で返金記録を入力 → paymentStatus を 'refunded' に更新、refundedAt / refundedAmount に記録
```

- ゲストはマイページがないため自身でのキャンセル不可。当日受付スタッフへ口頭申し出 → 運営が管理画面から `registration_cancel_by_admin` で削除

## ゲスト参加管理

### ゲスト名の自動生成

- **形式**: `Guest-{イベント内シーケンス}` （例: `Guest-1`, `Guest-2`, ...）
  - イベント単位でカウンタを管理。1からスタート
  - 同一イベント内での重複なし
- **受付画面での識別**:
  - 参加者リスト: `Guest-1` と一緒に**メアド一部表示**（例: `h*** @example.com`）で識別
  - 手動チェックイン検索: 名前またはメアドで検索可能。メアドマッチで一意に特定

### ゲスト登録削除（メアド誤入力時等）

- **シナリオ**: ゲストがメアド誤入力 → OTP届かず → セッション切れ → 会場で困る
- **対応**: 管理画面から当該ゲスト登録を削除（`registration_cancel_by_admin` で記録）
- **ゲスト再開**: 会場で再度 QR スキャン → 新規ゲスト登録フロー開始（メアド再入力可）
- **制約**: ゲストはメアド修正等自力対応が無いため、運営側で削除のみサポート

### イベント運営都合キャンセル

- `events.status = 'cancelled'` に更新
- **チェックイン済み参加者（`checkedIn = true`）**:
  - `cancelledAt` は更新しない（既に参加した事実は変わらない）
  - 返金対象外（参加済み）
  - 通知メール対象外（既に会場にいる想定）
- **チェックイン未済参加者（`checkedIn = false`）**:
  - `cancelledAt = now` に更新
  - **Stripe/paid参加者**: Stripe APIで自動全額返金。`refundDeadline` / `refundable`ポリシーは無視（運営都合キャンセルのため）
    - 処理順: `refundStatus = 'pending'` → Stripe API実行 → 成功時: `refundStatus = 'succeeded'` + `refundedAt` / `refundedAmount`を記録。失敗時: `refundStatus = 'failed'`のみ更新
  - **現金/unpaid参加者**: 未払いのため返金処理なし。登録キャンセルのみ
- **参加者への通知（即時送信）**:
  - チェックイン未済参加者のみに通知送信（既チェックイン者は除外）
  - Stripe/paid → 成功: 「キャンセルのため全額返金予定。○日以内にお戻りします」
  - Stripe/paid → 返金失敗: 「お支払い済みですが返金処理に遅延が生じています。お手数ですが○○（運営メアド）までお問い合わせください」
  - 現金/unpaid → 「参加費はお支払い不要です」
- **運営への通知**:
  - イベントキャンセル確定時に運営共有アドレスへメール送信
  - 件名: `【事務連絡】イベント「○○」キャンセル完了`
  - 本文: 返金処理の集計結果（成功数 / 失敗数 / 対象外数）+ 失敗者リスト（メアド / 金額）+ 既チェックイン数 + Stripeダッシュボードリンク
- **返金失敗時の対応**: 管理画面に返金失敗者のリストを表示。返金失敗者はStripeダッシュボードで手動返金後、管理画面から「手動返金済み」としてマーク可能（`refundStatus = 'succeeded'`に更新 + `refundedAt` / `refundedAmount`を記録）
- 監査ログに`event_cancel`として記録

## 受付待ち通知の dedup ルール

- QRスキャン / マイページボタン / ゲスト当日登録完了 で通知発火時に `registrations.lastNotifiedAt` を更新
- `lastNotifiedAt` が5分以内の場合は新規通知発火をスキップ（dedup）
- スタッフ側は受付専用画面の参加者リストでチェックイン済み・未済み両方を表示。通知は単にリスト内の該当参加者へのクイックジャンプ手段

## Stripe決済確認フロー

### Webhook-driven決済確認

- **参加者フロー:**
  1. Stripe Checkout 完了 → リダイレクト → `paymentStatus='pending'` のまま
  2. Webhook `checkout.session.completed` 受信 → `paymentStatus='paid'` 更新
  3. Supabase Realtime で確認画面をリアルタイム更新

- **Webhook実装:**
  - エンドポイント: `POST /api/webhooks/stripe`
  - 署名検証: `Stripe-Signature` ヘッダで HMAC-SHA256 検証
  - 冪等性: `sessionId` をユニークキーに upsert （同一イベント複数受信対応）
  - タイムアウト: 3秒以内に 200 OK 応答（失敗時は Stripe が自動リトライ）

- **サーバーダウン対策（定期同期Cron）:**
  - **実行**: 毎時 0 分（Vercel Cron Jobs / Supabase Edge Functions Scheduled）
  - **処理**: 
    - `paymentStatus='pending' AND createdAt < 1時間前` のレコードを抽出
    - 各 `stripePaymentId` に対し Stripe API で session 状態確認
    - `status='complete'` なら `paymentStatus='paid'` に更新
    - `status='expired'` なら `paymentStatus='unpaid'` に更新（参加者へ再支払い案内メール送信）
  - **ログ**: 更新件数を `admin_audit_logs` に `stripe_payment_sync` として記録

### Stripe Customer 運用ルール

- **会員のみ** Stripe Customer を作成し `users.stripeCustomerId` に保持（ゲストは作成しない）
- 会員の**初回決済時**に Stripe Customer を自動作成
  - `email`, `name` を渡して Customer を作成
  - `users.stripeCustomerId` に保存
  - Stripe Checkout Session に `customer` パラメータを指定
- **2回目以降の決済**は保存済みの `stripeCustomerId` を使用（決済履歴がStripe側で集約される）
- **ゲスト決済**: MVPではゲストは当日現金払いのみのためStripe Customer は不使用
- **メールアドレス変更時**: `stripeCustomerId` が存在すれば Stripe API で Customer の email も同期更新する

## 無料イベント（fee=0）の扱い

- **対象**: `events.fee = 0` のイベント
- **決済フロー**: Stripe Checkout をスキップ
  - 参加登録時に自動で `paymentStatus='paid'` / `paymentMethod='free'` を設定
  - `stripeSessionId` / `stripePaymentId` は NULL（Stripe連携なし）
  - `feeAtRegistration = 0`
- **QRセルフチェックイン**: 可能
  - 事前決済済み扱い（paymentStatus='paid'）なため、QRスキャンで自動チェックイン
  - チェックイン検証ルール L34-36 「paymentStatus='paid' + paymentMethod != 'cash'」を満たす
- **返金ロジック**: 不要
  - キャンセル時も返金処理なし。`refundStatus` は NULL のまま
  - 参加登録キャンセル処理のみ実施
- **マイページ表示**: 「支払い済み」（実際は支払いなし）

## 規約同意ポリシー

### 同意の記録

- **会員**: 初回ログイン時に表示（写真撮影同意を含む / アプリ内固定テキスト）。`users.termsAgreedAt` と `termsAgreedVersion` に記録
- **ゲスト**: 参加登録フォームで必須チェックボックスとして表示。`registrations.termsAgreedAt` と `termsAgreedVersion` に記録

### 規約テキスト

- **日英2バージョン**を用意し、ユーザーの言語設定に応じて表示（法的明確化のため）

### バージョン管理

- **規約バージョン更新時**: `termsAgreedVersion < 現行バージョン` のユーザーに対し再同意画面を表示。合意後に新バージョンを記録
- **強制リダイレクトルール（会員）**: ログイン後、`users.termsAgreedAt IS NULL` または `users.termsAgreedVersion < 現行バージョン` の場合、規約同意画面へ強制リダイレクト。同意完了まで他ページへのアクセス不可
- `users.termsAgreedAt` / `users.termsAgreedVersion` はDB上NULL許容。新規登録直後はNULL、初回同意後に記録される

## 多言語対応ルール

### UI

- ボタン・メニュー・システムメッセージ等は i18n で日英切替（next-i18next 等を想定）
- 言語選択: `users.preferredLanguage` に保存。未ログイン時はブラウザ設定を自動検出

### イベント情報

- `title` / `description` / `location` は単一フィールドで**日英混在運用**を想定（現コミュニティ運用の継続）

### 日時表示

- 言語設定に連動（`2026/04/11` / `Apr 11, 2026`）

### 参加費（fee）の変更ポリシー

- **変更可**: 既存登録者がいる場合でも fee 変更可能
- **理由**: `feeAtRegistration` で既登録者の請求額は保護済み。新規登録者のみ新fee適用
- **UI警告**: 既登録者がいる場合、変更前に警告表示「〇名の既登録者がいます。既登録者の請求額は変わりません」
- **効果タイミング**: 変更後の登録者から新 fee が適用

### メール通知

- システム部分の文言: ユーザーの言語設定に応じて日英切替
- イベント情報部分: DB上の混在テキストをそのまま埋め込み

## メール通知ルール

### 参加登録確認

- 参加確定時に自動送信

### リマインド

- 送信タイミング: `events.reminderHoursBefore` で指定（デフォルト24時間前）
- 件名・本文テンプレート: イベントごとに `events.reminderSubjectTemplate` / `reminderBodyTemplate` でカスタマイズ可能
- 変数置換: `{{eventTitle}}`、`{{eventDate}}`、`{{eventStartTime}}`、`{{participantName}}`
- **対象**: 会員のみ（`registrations.userId IS NOT NULL`）。ゲストは当日会場登録のため対象外
- **二重送信防止**: `registrations.reminderSentAt` が null のレコードのみ送信、送信成功後にタイムスタンプを記録

### 運営共有アドレスへの通知

- 参加申込時に運営共有アドレスへ通知（送信先アドレスは環境変数で設定）

### 未払いリマインド

- 管理画面から未払い参加者への一括リマインド送信

## 自動処理（Cron）の振り分け

実装基盤を用途で分割。詳細は [tech.md - 自動処理の振り分け方針](./tech.md#自動処理の振り分け方針) を参照。

### Supabase Cron（pg_cron）で実行

純SQLで完結する処理。

- **イベント自動クローズ**: 毎時実行
  - `endDateTime < NOW()` かつ `status = 'published'` のイベントを `status = 'closed'` に更新

### Cloudflare Workers Cron Triggers で実行

外部API呼び出し・テンプレート展開・多言語対応が必要な処理。

- **リマインドメール送信**: 毎時実行
  - 対象: `events.startDateTime BETWEEN NOW() + reminderHoursBefore AND NOW() + reminderHoursBefore + 1h`
  - 条件: `registrations.userId IS NOT NULL` AND `cancelledAt IS NULL` AND `reminderSentAt IS NULL`
  - 送信成功後 `reminderSentAt = NOW()` を記録（二重送信防止）
  - メール本文は `users.preferredLanguage` に応じて日英切替
- **Stripe 決済ステータス同期**: 毎時実行（[Stripe決済確認フロー](#stripe決済確認フロー) 参照）
  - 対象: `paymentStatus = 'pending'` で 1時間以上経過したレコード
  - Stripe API で再確認 → ステータス更新
