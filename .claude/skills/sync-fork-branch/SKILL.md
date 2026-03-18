---
name: sync-fork-branch
description: "フォーク元のリリースタグをバージョン順にフィーチャーブランチへマージして同期する。(1) フォーク元が新しいリリースを公開したとき、(2) 複数のリリースに追いつく必要があるとき、(3) バージョン追跡のためにフォーク元のタグをローカルに取得したいとき、に使用する。upstreamリモートからタグを直接取得し、未マージのタグを1つずつ順番にマージする。"
---

# sync-fork-branch

フォーク元のリリースタグを `feature/replace-roslyn-with-assembly-builder` にバージョン順でマージする。

**rebaseではなくmergeを使う** — 既存のコミットハッシュを保持する必要があるため。

## リモート構成

| リモート名 | リポジトリ | 役割 |
|-----------|-----------|------|
| `origin` | `naichilab/unity-cli-loop` | 自分のフォーク。push先 |
| `upstream` | `hatayama/unity-cli-loop` | フォーク元。fetchのみ（push無効化済み） |

## 手順

### 1. upstreamリモートの確認・登録

```bash
# 既に登録されているか確認。なければ追加してpushを無効化する
git remote get-url upstream 2>/dev/null || {
  git remote add upstream https://github.com/hatayama/unity-cli-loop.git
  git remote set-url --push upstream no-push
}
```

既に登録済みの場合は、push URLが無効化されているか確認する：

```bash
git remote -v  # upstream の push が "no-push" になっていること
```

### 2. フォーク元からタグを取得

```bash
git fetch upstream --tags
```

### 3. フィーチャーブランチに切り替え

```bash
git checkout feature/replace-roslyn-with-assembly-builder
```

### 4. 未マージタグの特定

フィーチャーブランチに含まれていないタグをバージョン順で列挙する：

```bash
git tag -l 'v*' --sort=version:refname | while read tag; do
  git merge-base --is-ancestor "$tag" HEAD 2>/dev/null || echo "$tag"
done
```

未マージタグがない場合は、ここで終了する（ブランチは最新の状態）。

### 5. ユーザーに確認

マージ対象のタグ一覧をユーザーに提示し、進めてよいか確認する。

### 6. タグを1つずつマージ

未マージタグをバージョン順に、1つずつマージする：

```bash
git merge <tag> -m "Merge upstream tag <tag> into feature/replace-roslyn-with-assembly-builder"
```

競合が発生した場合は解決してからコミットする：

```bash
git commit -m "Merge upstream tag <tag> into feature/replace-roslyn-with-assembly-builder"
```

### 7. リモートにpush

```bash
git push origin feature/replace-roslyn-with-assembly-builder
git push origin --tags
```

mergeコミットなので `--force` は不要。

## 競合解決ルール

### Roslyn関連ファイル（modify/delete競合）

このブランチでは `Packages/src/Editor/Roslyn/` ディレクトリを完全に削除している。フォーク元がこれらのファイルを変更すると modify/delete 競合が発生する。**常にこちら側（削除）を優先する**：

```bash
# 競合ファイルを一覧
git diff --name-only --diff-filter=U

# Roslyn関連ファイルはすべて削除
git diff --name-only --diff-filter=U | grep -i roslyn | xargs git rm
```

### その他の競合

内容を確認して手動で解決する。どちらか一方を盲目的に採用しないこと。

## 背景

このリポジトリはフォーク元の改造版で、`feature/replace-roslyn-with-assembly-builder` ブランチでRoslynベースの動的コンパイルをUnityの `AssemblyBuilder` APIに置き換え、`Microsoft.CodeAnalysis` 依存を完全に除去している。

マージ競合の主な原因は削除済みの `Packages/src/Editor/Roslyn/` ディレクトリ。フォーク元がこれらの削除済みファイルを変更すると modify/delete 競合が発生する。常にこちら側（削除）を優先して解決すること。
