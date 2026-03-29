# 横浜市立南高校 野球部OB会 公式サイト

<img src="icon.png" alt="南高校章" width="64">

マスターズ甲子園への挑戦、OB会の活動予定・試合結果、現役野球部の情報を掲載する公式サイトの技術解説です。

> **本番サイト**: https://minami-baseball-ob.vercel.app/

---

## 技術スタック

| カテゴリ | 技術 |
|---------|------|
| フロントエンド | Next.js 15 (App Router) + TypeScript + Tailwind CSS 4 |
| DB / 認証 / ストレージ | Supabase（PostgreSQL + Auth + Storage） |
| ホスティング | Vercel（git push で自動デプロイ） |
| CI/CD | GitHub Actions（権限同期 + 会員申請PR自動化 + keep-alive） |

---

## 機能一覧

### 公開ページ
- **トップページ** — ヒーロー画像 + 活動紹介パネル + 最新情報
- **マスターズ甲子園** — トーナメント表（画像） + 対戦結果
- **OB会の戦績** — 試合結果一覧 + 詳細ページ（写真・動画付き）
- **歴代戦績** — 現役野球部の歴代戦績（1955〜2024年）。大会種別フィルタ付き
- **今後の予定** — 試合・練習・懇親会など（種類別バッジ表示）
- **お知らせ** — 写真付きの告知
- **現役野球部** — 最新情報
- **ギャラリー** — 紐付け先別フォルダ表示（折りたたみ式）、YouTube埋め込み再生、写真クリックで別タブに原寸表示
- **メンバー** — 卒業年度別の構成（名前は非公開）

### 会員専用ページ（member権限以上）
- 住所録・連絡先など、会員向けの個人情報を掲載（対外非公開）

### 編集機能（editor権限以上）
- 予定・試合結果・お知らせ・現役情報の追加・編集・削除
- 写真アップロード（複数枚同時、長辺1200px自動リサイズ）
- YouTube動画の埋め込み（タイトル・紐付け先の編集可能）
- 写真の複数選択・一括削除
- **写真のゴミ箱**: 削除した写真を2日間保持 → 復元 or 完全削除。2日後に自動パージ
- ストレージ使用量のリアルタイム表示
- マスターズ甲子園トーナメント表のアップロード

### 管理機能（admin権限のみ）
- メンバー管理・権限設定（viewer/member/editor の切替）

---

## 設計方針

### 誰でも使いやすい UI
- **スマホファースト**: 全タッチターゲット 44px 以上、モバイル基本フォント 18px・行間 1.7
- **ダークモード対応**: ヘッダーの切替ボタンで明暗切替
- **でかいボタン、確認ダイアログ**: 誤操作防止
- **白ベース + 深紺アクセント**: 落ち着いた配色

### 5段階アクセスレベル

| レベル | ロール | できること |
|--------|--------|-----------|
| 1 | 未ログイン | 公開ページの閲覧 |
| 2 | `viewer`（デフォルト） | 公開ページの閲覧（ログイン済みだが未承認） |
| 3 | `member`（会員） | 公開ページ + 会員専用ページ（住所録等） |
| 4 | `editor`（編集者） | 会員ページ + コンテンツの追加・編集 |
| 5 | `admin`（管理者） | 全権限（メンバー管理・権限変更・データ削除） |

### 会員申請フロー（Googleフォーム → PR → 自動同期）

```
OBメンバー → Googleフォーム送信
    （卒業期 / 氏名 / Googleメール / 編集権限 要or不要）
         ↓
Google Apps Script → GitHub API (repository_dispatch)
         ↓
member-request.yml → PR自動作成
    「会員申請: 田中太郎（30期）→ member」
         ↓
管理者 → PRを確認してマージ
         ↓
sync-roles.yml → Supabase権限自動更新
```

Googleフォームの申請内容がそのままPRになるので、管理者はマージするだけ。YAML手動編集も不要。

### ログイン方式
- **Google OAuth のみ** — ワンタップでログイン。シンプルで安全

---

## DB設計

### テーブル

| テーブル | 内容 |
|---------|------|
| `members` | メンバー情報（管理用） |
| `user_roles` | 権限（admin/editor/member/viewer）+ 表示名 + 卒業期 |
| `schedule` | 予定（試合・練習・飲み会・懇親会・その他） |
| `results` | 試合結果（マスターズ甲子園予選/本選・練習試合・その他） |
| `announcements` | お知らせ |
| `photos` | 写真メタデータ（Storage連携、FK紐付け、`deleted_at`でソフトデリート） |
| `videos` | 動画（YouTube埋め込みURL） |
| `current_team_posts` | 現役野球部の情報 |

### RLS（行レベルセキュリティ）

全テーブルにRLSを有効化。ヘルパー関数で権限チェック:

| 関数 | 判定 |
|------|------|
| `is_admin()` | admin か |
| `is_editor_or_above()` | editor 以上か |
| `is_member_or_above()` | member 以上か |

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
| **Member Request PR** | `repository_dispatch`（Apps Script経由） | Googleフォーム申請からPR自動作成 |
| **Keep Supabase Alive** | 毎週日曜 UTC 0:00 | Supabase 無料枠の一時停止を防止するpingリクエスト |
| **Purge Deleted Photos** | 毎日 UTC 19:00（JST 4:00） | ゴミ箱の写真を2日後に自動完全削除（Storage + DB） |

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
/history             歴代戦績（公開）
/members             メンバー（公開）
/login               ログイン（公開）
/account             アカウント設定（ログイン済み）
/members-only/*      会員専用ページ（member以上）
/edit/*              編集画面 7画面（editor以上）
/admin/*             管理画面（adminのみ）
```

---

## 写真管理の工夫

- **紐付け管理**: 写真は `result_id`, `schedule_id`, `announcement_id`, `current_team_post_id` のFKで各テーブルに紐付け
- **フォルダ表示**: ギャラリー・編集画面では紐付け先ごとにフォルダ分け（折りたたみ式）
- **一括操作**: 選択モードで複数写真をチェック → 一括削除（ゴミ箱へ）
- **ゴミ箱**: ソフトデリート（`deleted_at`カラム）で2日間保持。復元 or 完全削除。GitHub Actionsで自動パージ
- **容量表示**: ストレージ使用量をプログレスバーで可視化（写真/動画の内訳付き）
- **自動リサイズ**: クライアント側で長辺1200px・JPEG品質85%にリサイズしてからアップロード
- **写真クリック拡大**: 全公開ページの写真をクリックで別タブに原寸表示（ピンチズーム対応）

---

## FC2元サイトからの移管

旧公式サイト（FC2）から段階的に情報を移管中。

- **移管済み**: OB会概要・設立趣旨、活動内容、應援團（校歌・応援歌）、関連リンク、注意書き、OB会規約（全15条+付則+振込先）
- **データ化済み**: 現役野球部の歴代戦績（1955〜2024年、429試合）→ `data/senseki.json`
- **データ化済み**: マスターズ甲子園予選 過去戦績20試合（2011〜2024年）→ `results` テーブル
- **今後**: 会長挨拶、役員一覧、写真の移管を段階的に検討

---

## SEO / メタデータ

| ファイル | 用途 | サイズ |
|---------|------|--------|
| `app/icon.png` | ブラウザタブ ファビコン（南高校章） | 32x32 |
| `app/apple-icon.png` | iPhone / iPad ホーム画面 | 180x180 |
| `public/icon-192.png` | Android PWA アイコン | 192x192 |
| `public/icon-512.png` | Android スプラッシュ / PWA | 512x512 |
| `app/manifest.ts` | Web App Manifest（Android「ホームに追加」） | - |
| `app/opengraph-image.tsx` | OGP画像（Facebook / LINE等） | 1200x630 |
| `app/twitter-image.tsx` | X(Twitter)カード画像 | 1200x600 |
| `app/robots.ts` | クローラー制御 | - |
| `app/sitemap.ts` | サイトマップ（静的 + 動的） | - |

### 設計

- **ファビコン・アプリアイコンは南高校章**の静的PNG（sharpで元画像から各サイズ生成）
- **OGP / Twitter画像は SVG → PNG 動的生成**（`next/og` の `ImageResponse`）
- **manifest.ts**: `maskable` purpose 指定で Android のアダプティブアイコン（丸型/角丸型）に対応
- **robots.ts**: `/admin/*`, `/edit/*`, `/api/*`, `/login` を `disallow`。公開ページのみインデックス
- **sitemap.ts**: 静的10ページ（優先度・更新頻度を個別設定）+ `results/[id]` を Supabase から動的生成（`updated_at` で `lastModified` を設定）
- **sitemap の DB アクセス**: `@supabase/supabase-js` 直接クライアント使用（`cookies()` 非依存でビルド時・クローラーアクセス時も安定）

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
