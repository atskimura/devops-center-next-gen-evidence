# 2026-08-30 PR 必須の Ruleset と昇格の直接 push が両立するか（5-6）

- 目的: `Require a pull request before merging` を有効にした状態で、
  DevOps Center の昇格が通るか
- 使った org: `dcng-prod`（Hub）/ `dcng-dev1` / `dcng-stg`
- きっかけ: [5-2](2026-08-28-ci-branch-protection.md) の6章で
  **昇格がステージブランチへの直接 push である**と分かったため。
  一般的な Git 開発では `main` / `staging` への直接 push を禁止するので、両立するかが問題になる

---

## 1. 何が問題だったか

5-2 で確認したのは required status check だけで、**PR 必須は有効にしていなかった。**

昇格の実体は `refs/heads/staging:refs/heads/staging` への push である。

```
<COMMIT_SHA>  author=<EMAIL_REDACTED>  Merge branch 'WI-000028' into staging
```

メッセージがローカルの `git merge` の既定形で、GitHub でマージしたときの
`Merge pull request #39 from ...` ではない。**DevOps Center は PR のマージ API を叩いていない。**

→ `Require a pull request before merging` は直接 push を禁止するルールなので、
**衝突する可能性があった。**

## 2. Ruleset の設定

```json
{
  "name": "staging-pr-required-5-6",
  "target": "branch",
  "enforcement": "active",
  "bypass_actors": [],
  "conditions": { "ref_name": { "include": ["refs/heads/staging"], "exclude": [] } },
  "rules": [
    { "type": "pull_request",
      "parameters": {
        "required_approving_review_count": 0,
        "dismiss_stale_reviews_on_push": false,
        "require_code_owner_review": false,
        "require_last_push_approval": false,
        "required_review_thread_resolution": false } },
    { "type": "required_status_checks",
      "parameters": {
        "strict_required_status_checks_policy": false,
        "required_status_checks": [ { "context": "check" } ] } }
  ]
}
```

**承認は0件、bypass は空。** required status check は**成功させた状態**で実行した
（止まった原因を PR 必須に絞るため）。

作成時、指定していないパラメータが既定値で入った。

```
require_extra_approval_for_unattributed_changes: true
allowed_merge_methods: ["merge", "squash", "rebase"]
```

**前者は GitHub Copilot が人に帰属しない Pull Request を作った場合の追加承認**であって、
「GitHub アカウントに紐づかないコミット」一般の話ではない。
**承認必須数が0件なら効果が無い**ので、今回の設定では最初から関係しない。

## 3. 実行前の状態

| 何 | 値 |
|---|---|
| `staging` の HEAD | `<COMMIT_SHA>` |
| PR #39 | `OPEN` / `MERGEABLE` / `mergeStateStatus: CLEAN` |
| check | `SUCCESS` |
| 有効なルール | `pull_request` と `required_status_checks`（ruleset_id `<RULESET_ID>`） |
| `WI-000028` | `READY_TO_PROMOTE`、変更リスト1件（`HybridTest__c.PrRuleNote__c` / `NEW`） |

## 4. 結果: 昇格は通った

18:40:47 に Connect API で昇格を実行し、**22秒で `PROMOTED`。**

| 何 | 実行前 | 実行後 |
|---|---|---|
| `staging` の HEAD | `<COMMIT_SHA>` | **`<COMMIT_SHA>`** |
| PR #39 | `OPEN` | **`MERGED`** |
| `mergeCommit` | — | `<COMMIT_SHA>`（**HEAD と一致**） |
| `mergedBy` | — | `github-user` |
| stg の org | 無し | **`PrRuleNote__c` が届いた** |

マージコミットの形は従来どおり。

```
<COMMIT_SHA> | author=<EMAIL_REDACTED> | committer=同じ | Merge branch 'WI-000028' into staging
```

## 5. 対照実験: PR 無しの直接 push は止まる

**「ルールが効いていなかっただけ」を排除するため**、同じ `staging` に PR 無しで push した。

```
git checkout staging
（README.md に1行足す）
git push origin staging
```

拒否された（原文）。

```
remote: error: GH013: Repository rule violations found for refs/heads/staging.
remote:
remote: - Changes must be made through a pull request.
remote:
remote: - Required status check "check" is expected.
remote:
 ! [remote rejected] staging -> staging (push declined due to repository rule violations)
```

`staging` の HEAD は `<COMMIT_SHA>` のまま動かなかった。

**同じブランチへの push が、PR の有無で結果が分かれた。**

## 6. 分かったこと

**`Require a pull request before merging` と DevOps Center の昇格は両立する。**

GitHub は「既存の open PR の head を含むマージコミットの push」を、
**その PR のマージとして扱う。** PR #39 の `mergeCommit` が push されたコミットと一致したことが裏付け。

| 操作 | PR 必須のルール下で |
|---|---|
| DevOps Center の昇格（PR がある） | **通る** |
| PR 無しの直接 push | **止まる**（`GH013`） |

→ **`staging` への直接 push を禁止する一般的な運用と、DevOps Center は共存できる。**
5-2 の required status check と合わせて、**CI と PR 必須の両方をゲートにできる。**

### まだ確認していないこと

- **承認を必須にした場合**（`required_approving_review_count: 1` 以上）。
  利用できる GitHub アカウントが1つなので、2人目の承認を要する状況を作れない
- **bypass 権限を持たない利用者からの操作**（同じ理由で実測不可）

## 7. 後始末

| 何 | 状態 |
|---|---|
| Ruleset `<RULESET_ID>` | **削除した**（`gh api ... --method DELETE`） |
| `staging` のルール | 0件に戻った |
| ローカルの対照実験用コミット | `git reset --hard origin/staging` で破棄 |
| `WI-000028` / `PrRuleNote__c` | **ステージングに残っている**（後片付けの対象） |
