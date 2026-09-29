# このリポジトリについて（ミラー）

[SideloadLabs/EeveeSpotifyReincarnated](https://github.com/SideloadLabs/EeveeSpotifyReincarnated)
の**独立したミラー**です。fork ではないため、上流が削除・非公開化されてもこのリポジトリは影響を受けません。

ライセンスは上流と同じ GPL-3.0 です（`LICENSE` 参照）。

## ブランチ構成

| ブランチ | 内容 |
|---|---|
| `Master`（デフォルト） | 上流 `Master` を merge で取り込んだもの + ミラー固有ファイル（本ファイルと `.github/workflows/mirror-sync.yml`） |
| `upstream-master` | 上流 `Master` の**完全な複製**。ミラー固有ファイルを含まない。fast-forward のみ |
| `hysan` / `test` | 上流の同名ブランチの複製。fast-forward のみ |

タグと Releases（添付ファイル含む）も同期されます。

## 同期の安全性

`.github/workflows/mirror-sync.yml` が毎日 04:17 JST に上流を追従します。
**force-push は一切行いません**（すべて追記のみ）。そのため以下が保証されます。

- 過去にこのミラーへ取り込まれた内容は、後の同期で消えることがない
- タグと Releases は **create-only**。既存のものは上書きされない
- 上流に存在しなくなったブランチ・タグをこちら側から削除しない

さらに、同期前に次のガードが働き、**引っかかった場合はミラーを一切変更せずに失敗**します。

| ガード | 条件 | 動く場面 |
|---|---|---|
| 上流到達不可 | `git fetch` 失敗 | 上流が削除・非公開化・リネームされた |
| 履歴書き換え検知 | 前回同期したコミットが上流の現 HEAD の祖先でない | 上流が force-push / rebase / 履歴削除した |
| 大量削除検知 | 削除ファイルが **25%** 超、または同期後の追跡ファイル数が **50** 未満 | 上流が中身を消して README だけにした等 |

大量削除が意図的なもの（正当なリファクタリング等）だと確認できた場合のみ、
Actions から本 workflow を **`allow_shrink = true`** で手動実行してください。
しきい値は workflow の `env:` の `MAX_DELETE_PERCENT` / `MIN_FILES` で調整できます。

### 履歴書き換え検知で止まった場合の復旧手順

上流が force-push すると、以後の定期実行は毎回このガードで失敗し続けます。
上流の差分を確認して問題なければ、手元で以下を行ってください
（`<OLD>` = 失敗ログの「前回同期」、`<NEW>` = 「上流現在」のコミット）。

```sh
git clone https://github.com/naopen/EeveeSpotifyReincarnated.git && cd EeveeSpotifyReincarnated
git remote add upstream https://github.com/SideloadLabs/EeveeSpotifyReincarnated.git
git fetch --no-tags upstream '+refs/heads/*:refs/remotes/upstream/*'

# 1. Master に上流をマージ。競合は上流側を採用し、ミラー固有ファイルだけ残す
git merge --no-ff --no-commit upstream/Master
git rm -rq --cached . && git read-tree upstream/Master
git checkout origin/Master -- .github/workflows/mirror-sync.yml MIRROR.md .mirror
git checkout-index -af && git clean -fd
git diff --cached --stat upstream/Master   # ミラー固有ファイルだけが差分に出ることを確認
git commit -m "mirror: sync upstream SideloadLabs/EeveeSpotifyReincarnated@<NEW の先頭7桁> (manual, upstream history rewritten)"
git push origin HEAD:Master

# 2. 旧 upstream-master の先端をタグで保全
git tag archive/upstream-master-<OLD の先頭7桁> <OLD>
git push origin refs/tags/archive/upstream-master-<OLD の先頭7桁>

# 3. upstream-master を上流の新 HEAD に合わせる（ここだけ強制更新）
git push --force-with-lease=upstream-master:<OLD> origin \
  refs/remotes/upstream/Master:refs/heads/upstream-master
```

その後 Actions から Mirror Sync を手動実行して成功を確認します。
旧コミットは `Master` の履歴と `archive/*` タグに残るため、何も失われません。
（前例: 2026-09 に上流が spoti.pw 関連コミットを削除 → `archive/upstream-master-ee5c721`）

上流が恒久的に消えた場合は、同期 workflow を無効化してください
（Actions → Mirror Sync → ⋯ → Disable workflow）。ミラーの内容はそのまま残ります。

## 必要な設定

- **書き込み権限付き deploy key** をこのリポジトリに登録し、その秘密鍵を
  Secret **`MIRROR_SSH_KEY`** として保存する。
  上流のコミットは `.github/workflows/` を変更しうるが、既定の `GITHUB_TOKEN` では
  workflow ファイルを push できない。この制限は GitHub App トークン固有のもので、
  SSH deploy key には適用されないため deploy key を使っている
  （PAT でも代替可能だが、deploy key はこのリポジトリだけに権限が閉じる）。
- Public リポジトリであること（ビルド workflow が `macos-latest` を使うため。Public なら Actions は無料）

`setup-github.sh` を使った場合、上記はすべて自動で設定済み。

## ビルドの実行

上流のビルド workflow はそのまま使えます（すべて手動実行 `workflow_dispatch`）。

| workflow | 用途 |
|---|---|
| Build IPA (PATCHED) | ブランチとIPA URLを指定してパッチ済み IPA をビルド |
| Build IPA (NO PATCH) | 同上（パッチなし） |
| Build rootless + RootHide .debs + Draft Release | .deb をビルドしてドラフトリリース作成 |
| Create IPA Packages | 既存の IPA と .deb から IPA を生成 |

`Build IPA (...)` の `branch` 入力には、上流と完全に同一の内容をビルドしたい場合
`upstream-master` を、ミラー既定の `Master` でよい場合は `Master` を指定します。
