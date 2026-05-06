# 技術構成

## 技術スタック

| レイヤー | 技術 |
| --- | --- |
| フロントエンド | React + Vite（SPA / PWA） |
| ホスティング（フロント） | Cloudflare Pages |
| バックエンド API | Hono（Cloudflare Workers 上で実行） |
| 認証 | Supabase Auth（メール/パスワード / OTP / TOTP MFA） |
| データベース | Supabase（PostgreSQL + RLS） |
| ファイルストレージ | Supabase Storage（イベントカバー画像等） |
| 決済 | Stripe Checkout / Refund API |
| メール送信 | 外部サービス（Resend / SendGrid 等から選定予定） |
| リアルタイム | Supabase Realtime（受付専用画面・参加状況表示用） |
| 自動処理（SQL系） | Supabase Cron（pg_cron）|
| 自動処理（API系） | Cloudflare Workers Cron Triggers |

## アーキテクチャ概略

```
[Cloudflare Pages] (React SPA)
       ↓ API call
[Cloudflare Workers] (Hono API)
       ↓
[Supabase] (DB + Auth + Realtime + Storage)
[Stripe] (決済 / 返金)
[Cloudflare Workers Cron] (リマインドメール、Stripe 状態同期)
[Supabase Cron] (イベント自動クローズ等のSQL処理)
```

## 技術方針

- MVP → 検証 → 改善のサイクル
- AI駆動開発 + 重要ロジックにはテスト実装
- 可用性重視（落ちないことが最優先）
- 疎結合: 各サービスは HTTP/SDK 経由で連携、ロックイン回避

## PWA 方針

軽量 PWA 構成（インストール可能 + 静的アセットキャッシュのみ）。フルオフライン非対応。

### 採用ライブラリ

- `vite-plugin-pwa`（Workbox ベース）

### キャッシュ戦略

- **静的アセット（HTML / CSS / JS / アイコン / フォント）**: Cache-first
  - 起動高速化、Workbox で precaching
- **API 通信**: Network-only（キャッシュなし）
  - リアルタイム性が必要なため（残り枠数・チェックイン状態等）
- **画像（Supabase Storage）**: Stale-while-revalidate
  - 表示は即時、裏で更新

### マニフェスト

- `manifest.json` 設置: アプリ名 / アイコン / テーマカラー / 表示モード `standalone`
- インストールプロンプト: 参加者のマイページアクセス時にバナー表示（リピート利用促進）

### 非対応範囲

- オフライン操作（チェックイン・登録等）
- バックグラウンド同期
- プッシュ通知（フェーズ2で検討）

### 受付スタッフ向け受付画面

- オンライン前提（Supabase Realtime 依存）
- 会場 Wi-Fi の安定性が運用条件

## 自動処理の振り分け方針

- **Supabase Cron（pg_cron）**: 純SQL で完結する処理
  - イベント自動クローズ（`endDateTime` 経過後 `status='closed'`）
- **Cloudflare Workers Cron Triggers**: 外部API呼び出しを伴う処理
  - リマインドメール送信
  - Stripe 決済ステータス同期（毎時）
  - メール本文の多言語切替・テンプレート展開を TS で実装

## テスト対象（重要ロジック）

- 支払いステータス管理（Webhook + Cron 同期）
- チェックイン処理（QRセルフ + スタッフ手動）
- QRコード生成・認証
- 返金処理（Stripe連携）
- 受付可否判定ロジック（定員・キャンセル枠・オーバーライド）
- キャンセル・返金期限の判定ロジック
- 監査ログの記録（重要操作が漏れなく記録されるか）
- リマインドメール送信の二重送信防止（`reminderSentAt` 制御）

## 外部サービス連携

- **Stripe**: Checkout（決済）/ Customer（会員のみ）/ Refund API（返金処理）/ Webhook（`checkout.session.completed`）
- **Supabase Auth**: メール/パスワード認証 / OTP（ゲスト用）/ TOTP MFA（admin用）/ JWT セッション管理（ゲストは3時間）
- **Supabase Realtime**: 受付専用画面の参加者リスト・通知のリアルタイム配信
- **Supabase Storage**: イベントカバー画像の保存・配信（RLS で権限制御）
- **メール配信サービス**: 未決定（Resend / SendGrid 等から選定予定）
