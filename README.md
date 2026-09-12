---
トークン使用量推定値: 2123
計測方法: "Anthropic Messages API count_tokens (claude-sonnet-5)"
---

# git-resolve-conflicts

Claude Code用のコンフリクト解決スキルです。
rebase/merge中のコンフリクトを、ファイル種別ごとの方針（設定ファイルは意図を汲んで手動マージ、ロックファイルは再生成コマンド任せ）で解決します。単体でも、[git-rebase](https://github.com/uga-skills/git-rebase)・[git-merge](https://github.com/uga-skills/git-merge)から呼ばれても、GitHub PRのURLを渡しても起動できます。

## Install

### すべてのプロジェクトで利用する場合

```bash
git clone git@github.com:uga-skills/git-resolve-conflicts.git ~/.claude/skills/git-resolve-conflicts
```

### 特定のプロジェクトでサブモジュールとして使う場合

```bash
git submodule add git@github.com:uga-skills/git-resolve-conflicts.git .claude/skills/git-resolve-conflicts
```

## 使い方

### 進行中のrebase/mergeのコンフリクトを解決する

```
/git-resolve-conflicts
```

### GitHub PRを対象に解決する

```
/git-resolve-conflicts https://github.com/<owner>/<repo>/pull/<number>
```

PRのhead/baseブランチを取得し、headブランチをチェックアウトしたうえでbaseブランチにrebaseし、コンフリクトを解決します。解決結果は指定フォーマットでチャットに報告するとともに、同一内容をPRコメントとして投稿します。force pushが必要な場合は必ず事前に確認します。

## 報告フォーマット

```
## <ファイル名> <行数>

\`\`\`diff
<解決後の該当差分>
\`\`\`

<競合原因、解決方法、影響範囲を簡潔に>
```

## 注意

- ロックファイル（`yarn.lock`, `package-lock.json`, `pnpm-lock.yaml` 等）は手編集せず、対応するパッケージマネージャのinstallコマンドで再生成します。
- `package.json`など人間が書く設定ファイルの意味的なコンフリクトは、黙って解決せず必ず要約を報告します。
- `--abort`やforce pushは、ユーザーの明示的な指示なしには実行しません。
