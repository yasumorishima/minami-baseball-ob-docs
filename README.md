# 横浜市立南高校 野球部OB会 公式サイト

マスターズ甲子園への挑戦、OB会の活動予定・試合結果、現役野球部の情報を掲載する公式サイトの技術解説です。

> **本番サイト**: https://minami-baseball-ob.vercel.app/

---

## 技術スタック

| カテゴリ | 技術 |
|---------|------|
| フロントエンド | Next.js 15 (App Router) + TypeScript + Tailwind CSS 4 |
| DB / 認証 / ストレージ | Supabase（PostgreSQL + Auth + Storage） |
| ホスティング | Vercel（git push で自動デプロイ） |
| CI/CD | GitHub Actions（権限同期 + Supabase keep-alive） |

---

## 機能一覧

### 公開ページ
- **トップページ** — ヒーロー画像 + 活動紹介パネル + 最新情報
- **マスターズ甲子園** — トーナメント表（画像） + 対戦結果
- **OB会の戦績** — 試合結果一覧 + 詳細ページ（写真・動画付き）
- **今後の予定** — 試合・練習・懇親会など（種類別バッジ表示）
- **お知らせ** — 写真付きの告知
- **現役野球部** — 最新情報
- **ギャラリー** — 紐付け先別フォルダ表示（折りたたみ式）、YouTube埋め込み再生
- **メンバー** — 卒業年度別の構成（名前は非公開）

### 編集機能（editor権限以上）
- 予定・試合結果・お知らせ・現役情報の追加・編集・削除
- 写真アップロード（複数枚同時、長辺1200px自動リサイズ）
- YouTube動画の埋め込み（タイトル・紐付け先の編集可能）
- 写真の複数選択・一括削除
- ストレージ使用量のリアルタイム表示
- マスターズ甲子園トーナメント表のアップロード

### 管理機能（admin権限のみ）
- メンバー管理・権限設定

---

## 設計方針

### 誰でも使いやすい UI
- **スマホファースト**: 全タッチターゲット 44px 以上
- **ダークモード対応**: ヘッダーの切替ボタンで明暗切替
- **でかいボタン、確認ダイアログ**: 誤操作防止
- **白ベース + 深紺アクセント**: 落ち着いた配色

### 認証・権限管理（GitHub で完結）

```
誰でもサインアップ可能
    ↓
config/members.yml に未登録 → viewer（閲覧のみ）
config/members.yml に登録済み → editor or admin
```

**運用フロー:**
1. 新メンバーが Google アカウントでサイトにログイン
2. 管理者が `config/members.yml` にメールアドレスと権限を追記
3. `git push` → GitHub Actions が自動で Supabase の権限テーブルを更新

YAMLファイルを編集してpushするだけ。DB操作は不要。

### ログイン方式
- **Google OAuth（推奨）** — ワンタップでログイン
- パスワード認証 / マジックリンク — 補助的に提供

---

## DB設計

### テーブル

| テーブル | 内容 |
|---------|------|
| `members` | メンバー情報（管理用） |
| `user_roles` | 権限（admin/editor/viewer）+ 表示名 |
| `schedule` | 予定（試合・練習・飲み会・懇親会・その他） |
| `results` | 試合結果（マスターズ甲子園予選/本選・練習試合・その他） |
| `announcements` | お知らせ |
| `photos` | 写真メタデータ（Storage連携、各テーブルへのFK） |
| `videos` | 動画（YouTube埋め込みURL） |
| `current_team_posts` | 現役野球部の情報 |

### ビュー（投稿者名の結合）

各テーブルに `_with_author` ビューを作成し、`user_roles.display_name` をJOIN。
公開ページでは「更新: 〇〇」として投稿者名を表示。

```sql
CREATE VIEW results_with_author AS
SELECT r.*, ur.display_name AS author_name
FROM results r
LEFT JOIN user_roles ur ON r.created_by = ur.user_id;
```

### 試合の種類（game_type）

| 種類 | results | schedule |
|------|---------|----------|
| マスターズ甲子園予選 | o | o |
| マスターズ甲子園本選 | o | o |
| 練習試合 | o | o |
| 練習 | - | o |
| 飲み会・懇親会 | - | o |
| その他 | o | o |

マスターズ甲子園ページは `game_type LIKE 'マスターズ甲子園%'` でフィルタ。
トーナメント表は Storage に画像アップロード。

### Storage

| バケット | 用途 | 制限 |
|---------|------|------|
| `photos` | 写真・トーナメント表 | 1枚5MBまで、アップ時に長辺1200pxリサイズ |
| `videos` | （未使用 — YouTube埋め込みに統一） | - |

RLSポリシーで SELECT/INSERT/UPDATE/DELETE を制御。

---

## GitHub Actions

| ワークフロー | トリガー | 内容 |
|-------------|---------|------|
| **Sync Member Roles** | `config/members.yml` 変更時 | YAMLを読み取り、Supabase の `user_roles` テーブルを自動更新 |
| **Keep Supabase Alive** | 毎週日曜 UTC 0:00 | Supabase 無料枠の一時停止を防止するpingリクエスト |

---

## ページ構成

```
/                    トップページ（公開）
/about               OB会について（公開）
/masters             マスターズ甲子園（公開）
/results             試合結果一覧（公開）
/results/[id]        試合詳細（公開）
/schedule            今後の予定（公開）
/announcements       お知らせ（公開）
/current-team        現役野球部（公開）
/gallery             ギャラリー（公開）
/members             メンバー（公開）
/login               ログイン（公開）
/edit/*              編集画面 7画面 + アカウント設定（editor以上）
/admin/*             管理画面（adminのみ）
```

---

## 写真管理の工夫

- **紐付け管理**: 写真は `result_id`, `schedule_id`, `announcement_id`, `current_team_post_id` のFKで各テーブルに紐付け
- **フォルダ表示**: ギャラリー・編集画面では紐付け先ごとにフォルダ分け（折りたたみ式）
- **一括操作**: 選択モードで複数写真をチェック → 一括削除
- **容量表示**: ストレージ使用量をプログレスバーで可視化（写真/動画の内訳付き）
- **自動リサイズ**: クライアント側で長辺1200px・JPEG品質85%にリサイズしてからアップロード

---

## 開発のポイント

- **Tailwind CSS 4 の `@theme`**: CSS変数でライト/ダークモードの色を一元管理
- **ダークモード**: `color-scheme: dark` + CSS変数でフォームコントロールも完全対応
- **共通コンポーネント**: `PhotoSection`（写真アップロード）、`GameTypeBadge`（試合種類バッジ）等を再利用
- **Server Components**: 公開ページはサーバーコンポーネントで高速表示、編集ページはクライアントコンポーネント
- **Supabase CLI**: マイグレーションファイルで DB スキーマをバージョン管理

---

## ライセンス

このリポジトリはサイトの技術解説です。ソースコードは非公開（private）です。
