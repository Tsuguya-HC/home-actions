# home-actions

ホームラボの各リポジトリから `workflow_call` で呼ぶ共有ワークフロー。

## なぜ分けたか

同じ lint を 6 リポジトリで個別に持っていた結果、抽出前の時点で
`actionlint` が 4 通り、`zizmor` が 4 通りに分かれていた。
`kubeconform` はバージョンそのものが v0.7.0 と v0.8.0 に割れていた。

ツールのピンが 6 箇所にあると、Renovate はリポジトリごとに PR を出す。
1 箇所に寄せれば、ツールの更新はここだけで起き、各リポジトリは
呼び出し先の SHA を 1 行上げるだけになる。

## 使い方

```yaml
jobs:
  lint:
    uses: Tsuguya-HC/home-actions/.github/workflows/lint-workflows.yml@<sha>
    permissions:
      contents: read
```

ワークフローは 3 本あり、どれも `contents: read` だけで呼ぶ。

| ワークフロー | 検査するもの | 呼び出し側に要るもの |
|---|---|---|
| `lint-workflows.yml` | `.github/workflows` を actionlint と zizmor（pedantic）で | 無し（actionlint のピンはこのリポのものを使う） |
| `lint-renovate.yml` | Renovate 設定を `renovate-config-validator --strict` で | Renovate 設定ファイル。無ければ「何も検証せず緑」を避けるため赤になる |
| `lint-aqua.yml` | `aqua-checksums.json` が `aqua.yaml` と一致しているか | リポ直下の `aqua.yaml` と `aqua-checksums.json`。無ければ赤になる |

public / private で分岐しない。zizmor は SARIF ではなく注釈で出すので
`security-events: write` は要らない。指摘があれば job が赤くなる点は
SARIF と同じで、違うのは読み方だけ（注釈は `::error:: file:line message`
としてログに直接出る）。

## 呼び出し側の前提

**必須ステータスチェックは `CI Gate` のような集約ジョブ 1 個にすること。**

`uses:` で呼ばれたジョブのチェック名は `<呼び出し側 job id> / <呼ばれた job 名>`
になる。個別ジョブ名を必須チェックに登録していると、ここで名前を変えた瞬間に
その context が報告されなくなり、全 PR が "Waiting for status to be reported"
で永久にブロックされる。

## ツールのピンの置き場所

`zizmor` は `zizmor-action` の digest と `zizmor-version` 入力の両方をここが持っているので、更新はここだけで起きる。

**`actionlint` も寄せた（v1.1.0〜）。** 以前は `aqua i` が読むのが
「チェックアウトされているリポジトリの `aqua.yaml`」＝**呼び出し側**だったため、
`rhysd/actionlint` のピンが呼び出し側6箇所に散っていた。

現在は `lint-workflows.yml` が `job.workflow_sha`（この reusable workflow が
解決された commit）でこのリポジトリの `aqua.yaml` / `aqua-checksums.json` だけを
sparse-checkout し、`AQUA_CONFIG` をそちらへ向けている。**呼び出し側がピンした
SHA と、そこで動く actionlint の版が同一 commit で凍結される。**

sparse-checkout にしているのは、ワークフローまで持ってくると actionlint が
home-actions 自身の `.github/workflows` も検査対象にしてしまうため。

### 呼び出し側に aqua.yaml は要らなくなった

`rhysd/actionlint` だけを宣言していたリポジトリは `aqua.yaml` と
`aqua-checksums.json` ごと削除できる。他のツール（kubeconform / talhelper 等）を
持つリポジトリは、その行だけ残せばよい。ただし削除したリポジトリは
`lint-aqua.yml` を呼ばないこと（検査対象が無いので赤になる）。

以前はこの前提を知らずに新しい呼び出し側を足すと、actionlint が PATH に無いまま
`exit 127` だけ出して死んだ（talos-custom-build と、このリポ自身の初回がそれ）。
