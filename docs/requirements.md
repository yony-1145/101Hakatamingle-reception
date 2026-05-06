# 101Hakata Mingle - 受付アプリ 要件定義

イベントコミュニティ「101Hakata Mingle」の受付業務を効率化するWebアプリケーション（PWA）の要件定義。各セクションは `requirements/` 配下にMECE構成で分割管理。

## 目次

| ファイル | 内容 | 観点 |
| --- | --- | --- |
| [overview.md](./requirements/overview.md) | プロジェクト概要・解決したい課題・アプローチ | Why |
| [users.md](./requirements/users.md) | ユーザー種別・権限・認証方式 | Who |
| [flows.md](./requirements/flows.md) | 利用フロー（参加者・ゲスト・運営） | When（順序） |
| [features.md](./requirements/features.md) | MVP機能要件（参加者向け / 運営向け） | What（能力） |
| [business-rules.md](./requirements/business-rules.md) | 検証ルール・判定ロジック・運用ルール（QR検証・受付可否・キャンセル・返金・通知dedup・規約同意・多言語・メール通知） | How（規則） |
| [data-schema.md](./requirements/data-schema.md) | データ設計（users / events / registrations / admin_audit_logs / event_staff） | データ |
| [tech.md](./requirements/tech.md) | 技術構成・テスト方針・外部サービス連携 | 技術 |
| [out-of-scope.md](./requirements/out-of-scope.md) | MVPスコープ外 / 未決定事項 | スコープ |

## MECE構成の方針

- **users.md**: 権限の主体（誰が）
- **flows.md**: ユースケース別の操作順序（いつ・どの順で）
- **features.md**: システムが提供する能力一覧（何ができる）
- **business-rules.md**: 機能の挙動を決める規則・ロジック（どう判断する）
- **data-schema.md**: テーブル定義のみ（何を保存する）
- **tech.md**: 採用技術と非機能の方針（どう作る）

各ファイルは独立した観点で記述され、同一情報は基本的に1箇所のみに記載。能力の能（features）と規則（business-rules）の関係は相互リンクで補完。
