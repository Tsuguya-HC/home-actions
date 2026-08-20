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
    uses: Tsuguya/home-actions/.github/workflows/lint-workflows.yml@<sha>
    permissions:
      contents: read
```

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

## 中央化できていない部分

`zizmor` は `zizmor-action` の digest と `zizmor-version` 入力の両方をここが持っているので、更新はここだけで起きる。

**`actionlint` は違う。** `lint-workflows.yml` の `aqua i` が読むのは
「チェックアウトされているリポジトリの `aqua.yaml`」＝**呼び出し側**なので、
`rhysd/actionlint` のピンは呼び出し側5箇所（+ 自己呼び出し用にここ）に残っている。
現時点では全て `v1.7.12` で揃っているが、構造としては再び割れうる。

寄せるなら、`lint-workflows.yml` が home-actions 自身を別ディレクトリに
チェックアウトして `AQUA_GLOBAL_CONFIG` をそちらへ向ける必要がある。
