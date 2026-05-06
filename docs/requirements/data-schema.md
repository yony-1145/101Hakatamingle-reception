# データ設計

テーブル定義のみを記載。判定ロジック・運用ルールは [business-rules.md](./business-rules.md) を参照。

## ユーザー (users)

| フィールド | 内容 | 必須/任意 |
| --- | --- | --- |
| id | Supabase Auth UUID | 必須 |
| email | メールアドレス | 必須 |
| name | 名前 | 必須 |
| birthDate | 生年月日（年齢は算出で取得） | 任意 |
| attribute | 属性 enum: `student` / `worker` / `entrepreneur` / `other` | 任意 |
| nationality | 国籍（ISO 3166-1 alpha-2 コードの配列 / 複数選択可） | 任意 |
| residenceCountry | 居住国（ISO 3166-1 alpha-2 コード / 単一値） | 任意 |
| residenceCity | 居住都市・地域（自由記述テキスト / 例: `福岡市`, `Seoul`） | 任意 |
| preferredLanguage | 言語設定（`ja` / `en` / デフォルト `ja`） | 任意 |
| stripeCustomerId | Stripe Customer ID（`cus_xxxxx` / null = 未作成） | 任意（初回決済時に自動作成） |
| role | ロール（`participant` / `admin`） | 必須（デフォルト `participant`） |
| termsAgreedAt | 規約同意日時 | NULL許容（初回ログイン時の規約同意後に記録。同意前はNULL） |
| termsAgreedVersion | 同意した規約バージョン（例: `1.0`, `2.0`） | NULL許容（初回ログイン時の規約同意後に記録。同意前はNULL） |
| createdAt | 登録日時 | 必須 |

**各enumの値:**

- **attribute**: `student`（学生）/ `worker`（社会人）/ `entrepreneur`（起業家）/ `other`（その他）
  - 将来的に選択肢の追加が想定される
- **nationality**: ISO 3166-1 alpha-2 の国コード（例: `JP`, `US`, `CN`, `KR` 等）を**配列**で保持。複数国籍対応
- **residenceCountry**: ISO 3166-1 alpha-2 の国コード（単一値）

## イベント (events)

| フィールド | 内容 |
| --- | --- |
| id | イベントID |
| title | タイトル |
| startDateTime | 開催開始日時 |
| endDateTime | 開催終了日時 |
| location | 開催場所 |
| description | 説明文 |
| fee | 参加費 |
| capacity | 定員 |
| imageUrl | カバー画像URL |
| status | ステータス（draft / published / closed / cancelled） |
| registrationOpen | 受付中フラグ（運営が手動で停止可能） |
| paymentType | 決済方式（stripe_only / stripe_and_cash） |
| refundable | 返金可否フラグ（キャンセル時の自動返金を制御） |
| refundDeadline | 返金期限（デフォルト: `startDateTime - 24時間`）。この日時以降のキャンセルは返金対象外 |
| allowOverCapacity | 定員超過許可フラグ（運営の手動オーバーライド / デフォルトfalse） |
| checkinToken | チェックインQR検証用の署名済みトークン（HMAC-SHA256） |
| reminderHoursBefore | リマインドメール送信タイミング（開始時刻の何時間前 / デフォルト24） |
| reminderSubjectTemplate | リマインドメールの件名テンプレート（デフォルト: "{{eventTitle}}のリマインド"） |
| reminderBodyTemplate | リマインドメールの本文テンプレート（デフォルト: 固定テンプレート） |
| createdBy | 作成者のユーザーID（users.role = 'admin'） |
| createdAt | 作成日時 |

## 参加履歴 (registrations)

| フィールド | 内容 |
| --- | --- |
| id | 登録ID |
| userId | ユーザーID（ゲストの場合は null） |
| guestName | 自動生成ゲスト名（例: Guest123 / 会員の場合は null） |
| guestEmail | ゲストメールアドレス（会員の場合は null） |
| eventId | イベントID |
| paymentStatus | 支払いステータス（unpaid / paid / pending / refunded） |
| paymentMethod | 支払い方法（stripe / cash / free） |
| stripeSessionId | Stripe Checkout Session ID（定期同期Cron用 / null = 現金払い） |
| stripePaymentId | Stripe Payment Intent ID（返金処理用 / null = 現金払い） |
| feeAtRegistration | 登録時点の参加費（円 / events.fee をコピー）。料金変更後の返金額計算に使用 |
| refundStatus | 返金ステータス（null = 返金対象外 / pending / succeeded / failed） |
| refundedAt | 返金日時（null = 未返金） |
| refundedAmount | 返金金額（円 / null = 未返金）。MVPでは全額返金のみだが、将来のキャンセルタイミングによる返金率変動・部分返金対応と監査のため明示的に記録 |
| checkedIn | チェックイン済みフラグ |
| checkedInAt | チェックイン日時 |
| lastNotifiedAt | 直近の受付待ち通知日時（QRスキャンまたはマイページボタンで更新 / nullの場合は未通知）。5分以内の重複通知をdedupするために使用 |
| reminderSentAt | リマインドメール送信日時（null = 未送信）。Cron 二重送信防止に使用。ゲスト登録は対象外（常にnull） |
| cancelledAt | キャンセル日時（null = 有効） |
| termsAgreedAt | 規約同意日時（ゲスト登録時に記録 / 会員は users.termsAgreedAt を参照） |
| termsAgreedVersion | 同意した規約バージョン（例: `1.0` / ゲスト登録時に記録） |
| createdAt | 登録日時 |

**制約:** `userId IS NOT NULL` または `(guestName IS NOT NULL AND guestEmail IS NOT NULL)` のいずれかを満たすこと

## 運営操作の監査ログ (admin_audit_logs)

重要な運営操作の証跡を記録するテーブル。

| フィールド | 内容 |
| --- | --- |
| id | ログID |
| adminUserId | 操作を行った運営のユーザーID（`users.id`） |
| action | アクション種別（下記enum） |
| targetType | 対象エンティティ種別（`registration` / `event` / `user`  等） |
| targetId | 対象エンティティのID |
| details | 変更内容のJSON（例: `{"before": "unpaid", "after": "paid"}`） |
| ipAddress | 操作時のIPアドレス（HTTPリクエストヘッダから取得） |
| createdAt | 操作日時 |

**actionのenum値:**

- `self_checkin`: QRセルフチェックイン（自動記録）
- `manual_checkin`: スタッフによる手動チェックイン
- `payment_status_update`: 支払いステータス手動変更（現金受領等）
- `cash_refund_record`: 現金返金の手動記録
- `capacity_override`: 定員超過オーバーライドの切替
- `registration_toggle`: 受付停止/再開
- `registration_cancel_by_admin`: 運営による参加キャンセル
- `checkin_token_regenerated`: QRコード再発行
- `event_cancel`: イベントキャンセル
- （将来的に追加予定）
- ※ `admin_role_change`（運営ロール付与/剥奪）はMVPではSupabaseダッシュボードから直接操作するためアプリ経由での記録不可。フェーズ2でアプリ内ロール管理UIを実装する際に追加予定

**記録例:**

```json
{
  "adminUserId": "admin-uuid-123",
  "action": "payment_status_update",
  "targetType": "registration",
  "targetId": "reg-uuid-456",
  "details": {
    "before": { "paymentStatus": "unpaid" },
    "after": { "paymentStatus": "paid", "paymentMethod": "cash" },
    "note": "当日現金受領"
  },
  "ipAddress": "203.0.113.1",
  "createdAt": "2026-04-25T19:35:12Z"
}
```

**運用方針:**

- MVPでは専用の閲覧UIを提供せず、Supabaseダッシュボードから直接クエリで確認
- フェーズ2以降で管理画面に「監査ログ閲覧」画面を追加予定（フィルタ・検索機能付き）
- 保持期間は**無期限**（小規模コミュニティではデータ量は月数十件程度を想定。データ量が増えた場合は自動クリーンアップを検討）

## 運営スタッフ割当 (event_staff)

各イベントに担当運営スタッフを割り当てる中間テーブル。

| フィールド | 内容 |
| --- | --- |
| id | 割当ID |
| eventId | イベントID |
| userId | スタッフのユーザーID（users.role = 'admin' のみ） |
| assignedBy | 割り当てを行った管理者のユーザーID（users.role = 'admin'） |
| assignedAt | 割当日時 |

**制約:**
- `(eventId, userId)` の組み合わせはユニーク
- `userId` / `assignedBy` は `users.role = 'admin'` であることをTriggerで検証

**運用想定:**

- 1イベントあたり3名程度の運営スタッフを割り当て
- 割り当てられたスタッフは管理画面・受付専用画面で確認可能
- 当日の運営メンバーを事前に明示化できる
