# 横浜市立南高校 野球部OB会 公式サイト

<img src="icon.png" alt="南高校章" width="64">

マスターズ甲子園への挑戦、OB会の活動予定・試合結果、現役野球部の情報を掲載する公式サイトの技術解説です。

> **本番サイト**: https://minami-baseball-ob.vercel.app/
> **ソースコード**: https://github.com/minami-baseball-ob/minami-baseball-ob（public）

---

## 技術スタック

| カテゴリ | 技術 |
|---------|------|
| フロントエンド | Next.js 15 (App Router) + TypeScript + Tailwind CSS 4 |
| DB / 認証 / ストレージ | Supabase（PostgreSQL + Auth + Storage） |
| ホスティング | Vercel（git push で自動デプロイ） |
| アクセス解析 | Google Analytics 4 |
| CI/CD | GitHub Actions（権限同期 + 会員申請PR自動化 + keep-alive + ゴミ箱パージ） |

---

## 機能一覧

### 公開ページ
- **トップページ** — ヒーロー画像 + 活動紹介パネル + ランダム写真グリッド
- **マスターズ甲子園** — 大会概要 + 通算成績 + トーナメント表（年度別画像）+ 大会資料 + 関連リンク
- **試合結果** — 一覧 + 詳細ページ（写真・動画付き）。マスターズ/練習試合/その他のフィルタタブ
- **歴代戦績** — 現役野球部の戦績（1955〜2024年、680試合）。年別折りたたみ + 大会種別フィルタ + 写真付き試合はアイコン表示
- **今後の予定** — 試合・練習・懇親会など（種類別バッジ表示）
- **お知らせ** — 写真付きの告知
- **現役野球部** — 最新情報 + 歴代戦績への導線
- **ギャラリー** — 紐付け先別フォルダ表示（折りたたみ式）、YouTube埋め込み再生、写真クリックで別タブに原寸表示
- **プライバシーポリシー**
- **カスタム404ページ** — ドット絵マスコット「空振り！」

### 会員専用ページ（member権限以上）
- 登録メンバー一覧（卒業期・名前・権限、折りたたみ表示）
- OB会費納入状況（年度別折りたたみ、済/未トグル）

### 編集機能（editor権限以上）
- 予定・試合結果・お知らせ・現役情報の追加・編集・削除
- 写真アップロード（複数枚同時、長辺1200px自動リサイズ）
- YouTube動画の埋め込み（タイトル・紐付け先の編集可能）
- 写真の複数選択・一括削除
- 歴代戦績の写真管理（年・写真有無フィルタ付き）
- マスターズ甲子園トーナメント表・大会資料のアップロード
- OB会費の管理（メンバー選択 or 名前直接入力、年度別）
- **ゴミ箱**: 削除したコンテンツ・写真を7日間保持 → 復元 or 完全削除。7日後に自動パージ
- **変更履歴**: admin限定で過去バージョンを閲覧・復元可能（DBトリガーで自動保存）
- ストレージ使用量のリアルタイム表示

### 管理機能（admin権限のみ）
- メンバー管理・権限設定（viewer/member/editor の切替）
- ゴミ箱管理（`/admin/trash` — 全テーブルの削除済みレコードを一覧・復元・完全削除）

---

## 設計方針

### 誰でも使いやすい UI
- **スマホファースト**: 全タッチターゲット 44px 以上、モバイル基本フォント 18px・行間 1.7
- **ダークモード対応**: ヘッダーの切替ボタンで明暗切替
- **でかいボタン、確認ダイアログ**: 誤操作防止
- **白ベース + エンジ（チームカラー #7b2234）アクセント**: ヘッダー/ボトムナビにエンジのライン、フッターは暗いエンジ背景 + ドット絵マスコット3体
- **UXフィードバック**: ボタン/カードのactive:scale、ページ遷移プログレスバー、折りたたみアニメーション

### 5段階アクセスレベル

| レベル | ロール | できること |
|--------|--------|-----------|
| 1 | 未ログイン | 公開ページの閲覧 |
| 2 | `viewer`（デフォルト） | 公開ページの閲覧（ログイン済みだが未承認） |
| 3 | `member`（会員） | 公開ページ + 会員専用ページ（会費納入状況等） |
| 4 | `editor`（編集者） | 会員ページ + コンテンツの追加・編集 |
| 5 | `admin`（管理者） | 全権限（メンバー管理・権限変更・データ削除・履歴閲覧） |

### 会員申請・権限変更フロー（Googleフォーム → PR → 自動同期）

```
OBメンバー → サイトにGoogleログイン（viewerとして作成）
         ↓
アカウント設定ページ → 会員申請フォーム or 権限変更申請を送信
    （卒業期 / 氏名 / 編集権限 要or不要 ※メールは自動収集）
         ↓
Google Apps Script → GitHub API (repository_dispatch)
         ↓
member-request.yml → PR自動作成
    新規: 「会員申請: 田中太郎（30期）→ member」
    変更: 「権限変更: 田中太郎（30期）member → editor」
    ※PRにメールアドレスは含まれない（UID方式）
         ↓
管理者 → PRを確認してマージ
         ↓
sync-roles.yml → Supabase権限自動更新
```

### フィードバック・バグ報告（Googleフォーム → Issue自動起票）

OBメンバーがGitHub不要でフィードバックを送信できる仕組み。

```
OBメンバー → Googleフォーム送信（カテゴリ・内容・画像）
         ↓
Google Apps Script → GitHub Issues API
    画像はissue-imagesリポにContents APIでコミット → Issue本文にMarkdown埋め込み
         ↓
GitHub Issue 自動作成（スコア修正/バグ報告/機能要望/試合情報の提供）
```

### ログイン方式
- **Google OAuth のみ** — ワンタップでログイン。シンプルで安全
- 誰でもGoogleアカウントでログイン可能（viewer として作成）
- ログイン後、アカウント設定ページから会員申請フォームを送信 → 管理者承認で member 昇格

---

## DB設計

### テーブル

| テーブル | 内容 |
|---------|------|
| `members` | メンバー情報（管理用、admin限定） |
| `user_roles` | 権限（admin/editor/member/viewer）+ 表示名 + 卒業期 |
| `schedule` | 予定（試合・練習・飲み会・懇親会・その他）。ソフトデリート対応 |
| `results` | 試合結果（マスターズ甲子園予選/本選/練習試合/その他）。ソフトデリート対応 |
| `announcements` | お知らせ。ソフトデリート対応 |
| `photos` | 写真メタデータ（Storage連携、FK紐付け、`deleted_at`でソフトデリート、`history_id`で歴代戦績と紐付け） |
| `videos` | 動画（YouTube埋め込みURL） |
| `current_team_posts` | 現役野球部の情報。ソフトデリート対応 |
| `masters_documents` | マスターズ甲子園の大会資料メタデータ（年度・タイトル・URL・ファイル種別） |
| `dues_payments` | OB会費納入記録（年度別、アカウントあり/なし両対応。member閲覧/editor編集） |
| `*_history`（4テーブル） | 変更履歴（UPDATE/DELETE時にDBトリガーで旧データを自動保存。admin閲覧用） |

### RLS（行レベルセキュリティ）

全テーブルにRLSを有効化。ヘルパー関数で権限チェック:

| 関数 | 判定 |
|------|------|
| `is_admin()` | admin か |
| `is_editor_or_above()` | editor 以上か |
| `is_member_or_above()` | member 以上か |

`members` テーブルは admin のみ読み取り可（anon/authenticated 遮断）。

### ビュー（投稿者名の結合）

各テーブルに `_with_author` ビューを作成し、`user_roles.display_name` をJOIN。
`WHERE deleted_at IS NULL` フィルタ付きで、ソフトデリート済みレコードを自動除外。
公開ページでは「更新: 〇〇」として投稿者名を表示。

### 試合の種類（game_type）

| 種類 | results | schedule |
|------|---------|----------|
| マスターズ甲子園予選 | o | o |
| マスターズ甲子園本選 | o | o |
| 練習試合 | o | o |
| 練習 | - | o |
| 飲み会・懇親会 | - | o |
| その他 | o | o |
| ー（未分類） | o | - |

マスターズ甲子園の試合結果は `/results?filter=masters` でフィルタ表示。「ー」は2020以前のマスターズ戦績等、種別が不明な試合に使用。
トーナメント表は Storage（`brackets/{year}/`）に年度別で画像アップロード。
大会資料（PDF等）は [masters-docs](https://github.com/minami-baseball-ob/masters-docs) リポに年度別で保存（サイト上からアップロード → GitHub Contents API 経由）。

### Storage

| バケット | 用途 | 制限 |
|---------|------|------|
| `photos` | 写真・トーナメント表 | 1枚5MBまで、アップ時に長辺1200pxリサイズ |
| `videos` | （未使用 — YouTube埋め込みに統一） | - |

---

## GitHub Actions

| ワークフロー | トリガー | 内容 |
|-------------|---------|------|
| **Sync Member Roles** | `config/members.yml` 変更時 | YAMLを読み取り、Supabase の `user_roles` テーブルを自動更新 |
| **Member Request PR** | `repository_dispatch`（Apps Script経由） | Googleフォーム申請からPR自動作成（新規登録 + 権限変更の両対応） |
| **Keep Supabase Alive** | 毎週日曜 UTC 0:00 | Supabase 無料枠の一時停止を防止するpingリクエスト |
| **Purge Deleted Records** | 毎日 UTC 19:00（JST 4:00） | ゴミ箱のレコード・写真を7日後に自動完全削除（全4コンテンツテーブル + photos） |

---

## ページ構成

```
/                    トップページ（公開）
/about               OB会について（公開）
/masters             マスターズ甲子園（公開）
/results             試合結果一覧（公開、フィルタ対応）
/results/[id]        試合詳細（公開）
/schedule            今後の予定（公開）
/schedule/[id]       予定詳細（公開）
/announcements       お知らせ（公開）
/announcements/[id]  お知らせ詳細（公開）
/current-team        現役野球部（公開）
/current-team/[id]   現役情報詳細（公開）
/gallery             ギャラリー（公開）
/history             歴代戦績（公開）
/history/[id]        歴代戦績 試合詳細（公開）
/privacy             プライバシーポリシー（公開）
/login               ログイン（公開）
/account             アカウント設定（ログイン済み）
/members-only/*      会員専用ページ（member以上）
/edit/*              編集画面 8画面（editor以上）
/admin/*             管理画面（adminのみ）
```

---

## 写真管理の工夫

- **紐付け管理**: 写真は `result_id`, `schedule_id`, `announcement_id`, `current_team_post_id`, `history_id` のFKで各テーブルに紐付け
- **フォルダ表示**: ギャラリー・編集画面では紐付け先ごとにフォルダ分け（折りたたみ式、日付降順）。各フォルダに「詳細ページを見る →」リンク付き
- **一括操作**: 選択モードで複数写真をチェック → 一括削除（ゴミ箱へ）
- **ゴミ箱**: ソフトデリート（`deleted_at`カラム）で7日間保持。復元 or 完全削除。GitHub Actionsで自動パージ
- **容量表示**: ストレージ使用量をプログレスバーで可視化（写真/動画の内訳付き）
- **自動リサイズ**: クライアント側で長辺1200px・JPEG品質85%にリサイズしてからアップロード
- **写真クリック拡大**: 全公開ページの写真をクリックで別タブに原寸表示（ピンチズーム対応）

---

## 編集の安全網

- **ソフトデリート**: 4コンテンツテーブル（results/schedule/announcements/current_team_posts）+ photos に `deleted_at` カラム。削除はゴミ箱に移動、7日後に自動パージ
- **変更履歴**: DBトリガーでUPDATE/DELETE時に旧データを `*_history` テーブルに自動保存。admin が各編集ページの「履歴」ボタンから過去バージョンを閲覧・復元可能
- **ゴミ箱ページ**: `/admin/trash` で全テーブルの削除済みレコードを一覧（残り日数表示）、復元・完全削除

---

## FC2元サイトからの移管

旧公式サイト（FC2）から段階的に情報を移管中。

- **移管済み**: OB会概要・設立趣旨、活動内容、應援團（校歌・応援歌）、関連リンク、注意書き、OB会規約（全15条+付則+振込先）
- **データ化済み**: 現役野球部の歴代戦績（1955〜2024年、680試合）→ `data/senseki.json`（検証率41%、260試合検証済み）
- **データ化済み**: マスターズ甲子園過去戦績（2011〜2024年、20試合）→ `results` テーブル
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

---

## セキュリティ

- **RLS**: 全テーブルで Row Level Security 有効。`members` テーブルは admin のみ読み取り可
- **service_role キー**: サーバーサイド専用（`lib/supabase/admin.ts`）。`import "server-only"` でクライアント漏洩を防止
- **CODEOWNERS**: `.github/workflows/`, `config/`, `supabase/` の変更は管理者レビュー必須
- **Branch protection**: masterへの直push禁止、PR必須+CODEOWNERS必須
- **Secret scanning + push protection**: 有効化済み
- **Dependabot**: 脆弱性パッケージ自動検知
- **全ワークフローに `permissions` 最小化**: 必要最小限の権限のみ付与
- **UID方式**: `config/members.yml` にメールアドレスを含めない（個人情報保護）

---

## 開発のポイント

- **Tailwind CSS 4 の `@theme`**: CSS変数でライト/ダークモードの色を一元管理（`--color-accent`, `--color-team`, `--color-team-dark` 等）
- **ダークモード**: `color-scheme: dark` + CSS変数でフォームコントロールも完全対応
- **共通コンポーネント**: `PhotoSection`（写真アップロード）、`GameTypeBadge`（試合種類バッジ）、`HistoryModal`（変更履歴）等を再利用
- **Server Components**: 公開ページはサーバーコンポーネントで高速表示、編集ページはクライアントコンポーネント
- **Supabase CLI**: マイグレーションファイルで DB スキーマをバージョン管理（015まで適用済み）
- **GitHub Org**: `minami-baseball-ob` Org配下で運用（public）
