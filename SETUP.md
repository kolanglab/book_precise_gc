# GitHub レビュー構成のセットアップ

このプロジェクトには `ligarb setup-github-review` によって、GitHub 上で本を
公開・レビューするための仕組みが生成されています。`ligarb` を更新したあとに同じコマンドを
再実行すると、足場ファイルが最新テンプレートに**上書き更新**されます（`book.yml` は対象外。
手を入れた箇所は `git diff` で確認してください）。

- 読者は公開された本を読み、気づいた点を **GitHub Issue** で送る。
- Issue をトリガーに GitHub Actions 上で Claude が確認し、修正なら **Pull Request** を、
  疑問なら issue コメントを返す。
- 人間が PR を merge して反映する。

生成されるファイルは「テンプレートのコピー」にすぎません。ligarb 自体は実行時に Claude や
GitHub を呼びません。以下を済ませると、これらのワークフローが動き始めます。

## 生成されるファイル

| ファイル | 役割 | Claude 依存 |
| --- | --- | --- |
| `.github/workflows/deploy-book.yml` | master への push で本をビルドし GitHub Pages に公開 | なし |
| `.github/workflows/build-check.yml` | PR で `ligarb build` が通るかを検証 | なし |
| `.github/ISSUE_TEMPLATE/book-feedback.yml` | 読者向けの構造化フィードバックフォーム | なし |
| `.github/ISSUE_TEMPLATE/config.yml` | Issue チューザの設定（Discussions リンク等） | なし |
| `.github/workflows/claude-feedback.yml` | Issue を Claude が処理し PR/コメントを返す | あり |
| `.github/workflows/claude-pr-mention.yml` | PR コメントに Claude が応答し追加修正をコミット | あり |

---

## A. 本を書く（執筆）

1. `book.yml` を編集し、`title` と `author` を設定する。
2. 章を Markdown ファイルで書き、`chapters:` に列挙する（`images/` に画像を置ける）。

   ```yaml
   title: "My Book"
   author: "Your Name"
   chapters:
     - 01-introduction.md
     - 02-getting-started.md
   ```

3. Markdown の記法・`book.yml` の全オプションは `ligarb help` で確認できる
   （AI に読ませる仕様書も兼ねる）。
4. AI に下書きを書かせたい場合は `ligarb write --init` → `ligarb write` も使える。

> `repository:` の設定は次の「C」で GitHub リポジトリ名を決めてから行います
> （読者向け「Report as issue」UI もそこで有効になります）。

## B. ローカルでビルド・プレビュー

```bash
ligarb build                 # build/index.html を生成
```

`build/index.html` をブラウザで開けば確認できます。執筆中は live reload + レビュー UI 付きの
ローカルサーバも使えます（ローカル専用・公開しないこと）:

```bash
ligarb serve                 # http://localhost:3000
```

## C. GitHub に公開・連携する（gh CLI）

[GitHub CLI](https://cli.github.com/) で、リポジトリ作成から Secret 設定まで一気に行えます。
まず `gh auth login` で認証しておきます。先頭で自分の値に置き換えてください。

```bash
# ── 自分の値に置き換える ──
OWNER=your-github-username
REPO=your-book-repo

# 1) book.yml の repository を設定（公開リンク & 読者フィードバック UI が有効になる）
#    ↓の行を book.yml に追記、または手で編集する
#    repository: "https://github.com/OWNER/REPO"

# 2) コミットしてリポジトリを作成 + push（ブランチは master 想定）
git add -A
git commit -m "Initial book"
gh repo create "$OWNER/$REPO" --public --source=. --remote=origin --push

# 3) Claude 用トークンを Secret に登録（プロンプトに貼り付け。履歴に残さない）
#    事前に `claude setup-token` で sk-ant-oat01-... を生成しておく
gh secret set CLAUDE_CODE_OAUTH_TOKEN

# 4) GitHub Pages を「GitHub Actions」ソースで有効化
gh api -X POST "repos/$OWNER/$REPO/pages" -f build_type=workflow
#    既に有効なら 409 になる。その場合は↓
# gh api -X PUT "repos/$OWNER/$REPO/pages" -f build_type=workflow

# 5) Actions に PR 作成権限を付与（Claude が PR を作るのに必要）
gh api -X PUT "repos/$OWNER/$REPO/actions/permissions/workflow" \
  -f default_workflow_permissions=write \
  -F can_approve_pull_request_reviews=true

# 6) ワークフローが使うラベルを作成（既存なら更新）
for L in feedback approved needs-triage needs-human answered claude-generated strong-model; do
  gh label create "$L" --force
done
```

push すると `deploy-book.yml` が走って Pages に公開され、issue を立てると Claude が動き始めます。

> **トークンの取り扱い**: `sk-ant-oat01-...` はパスワード相当のシークレットです。コードや
> コミット、issue、チャットなどに貼らないでください。`gh secret set` のプロンプトに貼るのが
> 安全です。約1年で失効するので、その際は `claude setup-token` で再生成して同じ手順で更新します。

---

## 補足・各設定の意味（Web UI での代替）

gh を使わない場合や、設定の意味を確認したいときの参照です。

### クレジットの扱い（重要）

- 2026/6/15 以降、GitHub Actions 上での Claude 利用は Agent SDK のクレジットから引かれる。
  クレジットは一度オプトインで請求が必要。
- 「追加使用 / usage credits」は **オフのまま**にしておくこと。枯渇時に停止し、
  青天井の課金を防げる。

### GitHub Pages（手順 C-4 の代替）

- Settings > Pages > Source を **"GitHub Actions"** にする。

### Actions の権限（手順 C-5 の代替）

- Settings > Actions > General で次を有効化する:
  - **Read and write permissions**
  - **Allow GitHub Actions to create and approve pull requests**
- これらが無いと Claude が PR を作成できない。

### ラベルの意味（手順 C-6）

- `feedback` — フィードバック issue
- `approved` — メンテナーが処理を承認した issue
- `needs-triage` — メンバー外の起票（自動処理せず記録のみ）
- `needs-human` — Claude が自信を持てず人手レビューが必要
- `answered` — 疑問に回答済み
- `claude-generated` — Claude が作成した PR（PR 上での自動応答の目印）
- `strong-model` — このラベルが付いた issue / PR では、Claude が強いモデル（opus）で処理する
  （無印は sonnet）。難しい回だけ品質を上げたいときに付ける。

### モデルの選択（strong-model ラベル）

既定では Claude は **sonnet** で動きます。issue や PR に **`strong-model`** ラベルを付けると、
その回だけ **opus** で処理します（教科書の編集には sonnet で十分なことが多く、コストを抑えつつ、
難しい回だけ強いモデルに上げるための仕組み）。

- **メンバー外の issue**: `approved` と一緒に `strong-model` を付ける（承認で起動するため）。
- **自分（メンバー）の issue**: 起票時に `strong-model` を付けておく（起票と同時に走るため）。
- **PR コメントへの応答**: その PR に `strong-model` を付けておく。
- 付け方は Web UI の Labels、または `gh issue edit <番号> --add-label strong-model` /
  `gh pr edit <番号> --add-label strong-model`。
- 将来 opus より強いモデルが出たら、両ワークフローの `'opus'` を新モデル名に差し替えるだけでよい。

### book.yml の `repository:`（手順 C-1）

- `repository: "https://github.com/OWNER/REPO"` を設定すると、公開ページの読者向け
  「Report as issue」UI が有効になる（`ligarb setup-github-review` が
  `github_review.enabled: true` を book.yml に書き込み済み。`repository` 未設定の間は
  build が警告を出して UI 注入をスキップする）。
- これを設定したうえで再度 `ligarb setup-github-review` を実行すると、Issue フォームや
  Discussions リンクのプレースホルダ（`__OWNER__` / `__REPO__`）が実際の値に置換されます
  （足場ファイルは上書き更新されるので、手で直していた箇所は `git diff` で確認を）。

### （任意・推奨）必須チェックの指定

- Settings > Branches で `build-check` を必須チェックに指定すると、
  ビルドが落ちる PR を merge できなくなる。

### claude-code-action の最新仕様の確認

- `anthropics/claude-code-action` はベータで入力仕様が変わりうる。
  初回コミット前に同 action の README で最新の入力名を確認すること。

## ブランチ名について

ワークフローは公開・PR の対象ブランチを `master` と仮定しています。`main` を使う場合は、
`deploy-book.yml` と `build-check.yml` の `branches:` を `main` に変更してください。
