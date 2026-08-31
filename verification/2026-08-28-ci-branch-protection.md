# 2026-08-28 GitHub 側のルールで昇格を止められるか（5-2）

- 目的: 公式が書いている「mergeability rule or merge setting isn't met なら昇格をブロックする」が実機で成り立つか。
- 使った org: `dcng-prod`（Hub）/ `dcng-dev1`
- **予測を先に書いてから実行した。**
- リポジトリは移管直後の `example-company/example-project`
  （→ [移管のログ](2026-08-28-repo-transfer.md)）

---

## 1. CI を落とすファイルで成否を切り替える

`.github/workflows/ci.yml` は1つだけ置き、
リポジトリに `ci-should-fail.txt` があるときだけ `exit 1` にした。

```yaml
name: CI
on:
  pull_request:
    branches:
      - staging
jobs:
  check:
    name: check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: ci-should-fail.txt があれば失敗させる
        run: |
          if [ -f ci-should-fail.txt ]; then
            echo "ci-should-fail.txt がある。CI をわざと落とす（5-2 の検証）"
            exit 1
          fi
          echo "ci-should-fail.txt は無い。成功で返す"
```

失敗する workflow を作業項目で運ぼうとすると、その変更を載せた PR 自身が自分の check に落ちて進めない。
ファイルの有無で切り替えれば、workflow は一度置けば触らずに済む。

`pull_request` の base を `staging` に絞ったのは、
`main` 向けの Promotion PR まで走らせないためである。

## 2. 保護をかける前に1周させた（WI-000013）

`ci.yml` と使い捨ての項目 `CiNote__c` を1つの作業項目で運んだ。
`.github/` は `force-app` の外なので変更リストには乗らないが、
MCP のコミットは作業ツリー全体を対象にするので一緒に入る。

| 手順 | 使ったもの | 結果 |
|---|---|---|
| 作業項目の作成 | `create_devops_center_work_item` | `WI-000013` |
| 開発環境の割り当て | `sf data update record` | dev1 |
| 「進行中」にする | `update_devops_center_work_item_status` | **画面を触らずにブランチが作られた** |
| コミット | `commit_devops_center_work_item` | `<COMMIT_SHA>` |
| PR | `create_devops_center_pull_request` | PR #17 |
| check | GitHub Actions | **pass（3秒）** |

作業項目を「進行中」にする操作は MCP から通った。
[08-27 の 9-1](2026-08-27-mcp-workitem.md) では画面で押していたが、ツールがある。

check が動いたことで **5-1（管理下リポジトリで GitHub Actions が動くか）の答えも出た。**

## 3. `No components to deploy` の原因

「昇格準備完了」にしてから `promote` を呼ぶと失敗した。

```json
{"errorType":"DEPLOYMENT_FAILURE",
 "errorMessage":"No components to deploy — the resolved component set was empty. ..."}
```

`WorkItemComponentList` を引くと、`WI-000013` には**1件も無かった**。
画面の変更リストタブを開くと 01:35:15 にレコードが作られ、そのあとの `promote` は通った。

`DevopsRequestInfo` にはタブを開いた前後で `INSPECT` が2件記録されていた。

```
01:35:09 INSPECT SUCCESS
01:35:11 INSPECT ERROR   Failed to create WorkItemComponentList records - all records were rolled back
01:35:15 （WorkItemComponentList が作られた時刻）
01:35:56 PROMOTE SUCCESS
```

**変更リストを作るのは `INSPECT` という操作で、画面の変更リストタブを開くと走る。**
`INSPECT` が走っていない作業項目を昇格すると `No components to deploy` になる。

> 08-28 の移管のログで、このエラーの原因を「作業項目が `IN_REVIEW` のままだった」と書いたが誤り。
> `WI-000013` は `READY_TO_PROMOTE` にしてから呼んでも同じエラーになった。
> 状況の遷移ではなく、変更リストの有無が条件である。

## 4. required status check で昇格が止まった

`staging` に保護をかけた。最初は classic branch protection で試した。

```json
{
  "required_status_checks": { "strict": false, "contexts": ["check"] },
  "enforce_admins": true,
  "required_pull_request_reviews": null,
  "restrictions": null
}
```

`enforce_admins` を入れたのは、DevOps Center が接続している GitHub ユーザーが
`example-company` の owner だからである。
これを入れないと管理者として保護を素通りし、
「DevOps Center は Git 側のルールを見ない」という誤った結果になる。

**PR 経由は強制していない**（`required_pull_request_reviews` は `null`）。

`ci-should-fail.txt` と項目 `CiFailNote__c` を載せた作業項目 `WI-000014` を作った。
メタデータの変更を一緒に入れたのは、変更リストが空で止まったのではないと分かるようにするためである。

- check: **fail（4秒）**
- PR #18: `mergeable=MERGEABLE` / `mergeStateStatus=BLOCKED`
- 「[昇格準備完了] とマーク」: **通った**（GitHub を触らない操作なので）
- 変更リスト: 1件（`WorkItemComponent` も1件）

この状態で `promote` を呼ぶと失敗した。

```
!  refs/heads/staging:refs/heads/staging  [remote rejected] (protected branch hook declined)
remote: error: GH006: Protected branch update failed for refs/heads/staging.
remote: - Required status check "check" is failing.
error: failed to push some refs to '<PRIVATE_REPOSITORY_URL>'
```

`errorType` は `UNKNOWN`。**`ErrorDetails` に git の出力がそのまま入っている。**

作業項目の画面には何も出ない。状況が変わらないだけである。

→ [失敗直後の作業項目の画面](screenshots/2026-08-28-10-check失敗で昇格が止まった.png)

同じ内容は**活動履歴のレコードで読める。**
ナビゲーションの「活動履歴」から `PROMOTION_COMPLETED` / `FAIL` を開くと、
「エラーの詳細」項目に git の出力がそのまま入っている。

| 項目 | 値 |
|---|---|
| 活動種別 | プロモーションを完了 |
| パイプラインフェーズ | ステージング |
| エラーの詳細 | `{"errorType":"UNKNOWN","errorMessage":"To <PRIVATE_REPOSITORY_URL> GH013: Repository rule violations found ... Required status check \"check\" is failing. ...` |

→ [活動履歴のエラー詳細](screenshots/2026-08-28-11-活動履歴のエラー詳細.png)

08-27 の 3-7 では、パイプラインの画面から
警告 → 「詳細を表示」のモーダル → もう一段で活動履歴、という導線も確認している
（→ [3-7 のログ](2026-08-27-run-back-sync-3-7.md) の5章）。

**読める場所はある。作業項目の画面だけを見ていると気づかない。**

## 5. Rulesets でも止まる

classic branch protection は GitHub の古い方式なので、Rulesets でも試した。

```json
{
  "name": "staging-ci-gate",
  "target": "branch",
  "enforcement": "active",
  "bypass_actors": [],
  "conditions": { "ref_name": { "include": ["refs/heads/staging"], "exclude": [] } },
  "rules": [
    { "type": "required_status_checks",
      "parameters": {
        "strict_required_status_checks_policy": false,
        "required_status_checks": [ { "context": "check" } ] } }
  ]
}
```

`bypass_actors` を空にすると管理者も対象になる。同じ作業項目を昇格すると止まった。

```
!  refs/heads/staging:refs/heads/staging  [remote rejected] (push declined due to repository rule violations)
remote: error: GH013: Repository rule violations found for refs/heads/staging.
remote: - Required status check "check" is failing.
```

| 方式 | エラーコード | 文言 |
|---|---|---|
| classic branch protection | `GH006` | `Protected branch update failed` |
| Rulesets | `GH013` | `Repository rule violations found` |

止まる条件は同じで、`Required status check "check" is failing.` の行も共通だった。

## 6. 昇格はマージではなく push である

エラーが指しているのは `refs/heads/staging:refs/heads/staging` への push である。
`staging` のマージコミットを見ると、この形が裏付けられる。

```
commit    <COMMIT_SHA>
author    <EMAIL_REDACTED>
committer <EMAIL_REDACTED>
parents   （2つ）
subject   Merge branch 'WI-000014' into staging
```

| 見どころ | 意味 |
|---|---|
| メッセージが `Merge branch 'WI-000014' into staging` | ローカルの `git merge` の既定メッセージ。GitHub でマージすると `Merge pull request #18 from ...` になる |
| author が `<EMAIL_REDACTED>` | GitHub アカウントではなく、そのステージの org の Salesforce ユーザー |
| PR の `mergedBy` | `github-user`（push した接続ユーザー） |

**DevOps Center は PR のマージ API を叩かない。**
ローカルで `git merge` して `staging` に push し、
その push で PR の head が base に含まれるので GitHub が PR を `MERGED` と判定する。

このため、**PR のマージだけを対象にした設定ではなく、そのブランチへの push を弾く保護が効く。**
今回は PR 経由を強制していない状態で止まった。

## 7. check を通しても昇格は自力で再開しない

`ci-should-fail.txt` を消して push し直した。

- check: **pass（6秒）**
- PR #18: `mergeStateStatus=CLEAN`

この時点で `DevopsRequestInfo` に新しい `PROMOTE` は記録されなかった。
**昇格をもう一度呼ぶ必要がある。** 呼び直すと通った。

```
01:39:49 PROMOTE ERROR    （classic / GH006）
01:55:20 PROMOTE ERROR    （Rulesets / GH013）
01:57:01 PROMOTE SUCCESS
01:57:07 staging にマージコミット <COMMIT_SHA>
01:57:14 PR #18 が MERGED
```

`WI-000014` は `PROMOTED` になり、`CiFailNote__c` が staging に入った。

なお `.github/` だけの変更は MCP のコミットが受け付けない。

```
commit_devops_center_work_item
→ Error committing work item: No eligible changes to commit (only Unchanged components detected).
```

`force-app` に差分が無いと拒否される。この回は `git commit` と `git push` で入れた。

## 8. 予測との照合

| 何 | 予測 | 結果 |
|---|---|---|
| 「[昇格準備完了] とマーク」 | 通る | **当たり** |
| 昇格 | ここで止まる | **当たり** |
| `ErrorDetails` | `MERGE_CONFLICT` とは別の型 | **当たり**（`UNKNOWN` + git の生出力） |
| 画面 | エラーが出るか不明 | **出ない** |
| check を通した後 | もう一度昇格を押す必要がある | **当たり** |

予測していなかったこと。

- **昇格はマージではなく push である**（6章）
- 変更リストを作るのは `INSPECT` で、画面を開かないと走らない（3章）
- classic と Rulesets でエラーコードが変わる（5章）

## 9. 分かったこと

- **GitHub Actions の status check を required にすると、昇格が止まる。**
  公式の「mergeability rule or merge setting isn't met」は次世代でも成り立つ
- 止まるのは `staging` への push なので、**PR 経由を強制していなくても効く**
- **作業項目の画面にはエラーが出ない。** ただし**活動履歴のレコードで読める**（4章）。
  git の出力がそのまま入っているので、たどり着けば原因は分かる
- check を直しても**昇格は自力で再開しない**。もう一度呼ぶ
- 管理者の迂回を塞がないと実験が成立しない
  （classic なら `enforce_admins`、Rulesets なら `bypass_actors` を空に）

## 10. 未確認として残したもの

- **どのコミットの check を見て `failing` と判定しているか。**
  push されるマージコミットでは workflow が走っていない。
  PR の head の結果を見ているのか、check が無いこと自体を拒否しているのかは区別していない
- **レビュー必須（5-3）。** GitHub は自分の PR を自分で承認できず、
  `example-company` に使えるアカウントが他に無いので、承認して通る側が示せない。
  止まる仕組みは push を弾く点で 5-2 と同じと考えられるが、実機では確かめていない
- `main` 向けの Promotion PR に check を効かせられるか（5-4）

## 11. 作業の記録に残る誤り

- 昇格の失敗直後に画面が「レビュー中 / 0 件の変更されたコンポーネント」と表示していたのを見て、
  「状況が巻き戻り、変更リストが空になる」と書いた。
  SOQL では `READY_TO_PROMOTE` のままで `WorkItemComponent` も1件あった。
  **画面の表示が実態と食い違っていただけである**
- 「DevOps Center は PR をマージせず push する。だから PR は `OPEN` のまま」と書いた。
  前半は正しいが、後半は誤り。push が通れば GitHub が PR を `MERGED` にする。
  `WI-000014` の PR が `OPEN` だったのは push が弾かれていたからである
- 最初に classic branch protection で実験した。GitHub の古い方式なので、Rulesets でやり直した
- 「画面にはエラーが出ない」と書いた。見ていたのが作業項目の画面だけだった。
  活動履歴のレコードには全文が出る。
  **08-27 に自分で「エラーの本文は画面で読める。1クリック余分に必要」と記録していた**
  （→ [3-7 のログ](2026-08-27-run-back-sync-3-7.md) の5章）
