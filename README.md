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
    # public リポジトリなら SARIF を上げる
    # with:
    #   advanced-security: true
    # permissions:
    #   contents: read
    #   security-events: write
```

## 呼び出し側の前提

**必須ステータスチェックは `CI Gate` のような集約ジョブ 1 個にすること。**

`uses:` で呼ばれたジョブのチェック名は `<呼び出し側 job id> / <呼ばれた job 名>`
になる。個別ジョブ名を必須チェックに登録していると、ここで名前を変えた瞬間に
その context が報告されなくなり、全 PR が "Waiting for status to be reported"
で永久にブロックされる。
