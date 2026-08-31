# 2026-08-28 CLI から昇格できるか（5-5）

- 目的: Pull RequestのマージをトリガーにGitHub Actionsから`sf project deploy pipeline start`を実行し、
  DevOps Center の画面を一度も開かずに1周させたい
- 使った org: `dcng-prod`（Hub）/ `dcng-dev1`
- **予測を先に書いてから実行した。**

---

## 1. 画面を開かずに PR まで進めた

`WI-000015` を作り、変更リストが作られない状態を保ったまま PR まで進めた。

| 手順 | 使ったもの | 結果 |
|---|---|---|
| 作業項目の作成 | `create_devops_center_work_item` | `WI-000015` |
| 開発環境の割り当て | `sf data update record` | dev1 |
| 「進行中」にする | `update_devops_center_work_item_status` | ブランチが作られた |
| 項目の作成 | `CliNote__c` を dev1 にデプロイ | Created |
| コミット | `commit_devops_center_work_item` | `<COMMIT_SHA>` |
| PR | `create_devops_center_pull_request` | PR #19 |

この時点の状態。

- `WorkItemComponentList`: **0件**（画面を開いていないので `INSPECT` が走っていない）
- `WorkItem.Status`: `IN_REVIEW`
- `staging` の先端: `<COMMIT_SHA>`（`WI-000015` は未マージ）

## 2. CLI はマージをしない

ヘルプに前提が書いてある。

> Before you run this command, changes in the pipeline stage's branch must be merged
> in the source control repository.

**マージは自分でやる。** 予測どおりだった。
そこで PR #19 を GitHub 側でマージした。

```
<COMMIT_SHA> Merge pull request #19 from example-company/WI-000015
```

このマージコミットは `Merge pull request #19 from ...` で、author は `github-user`。
DevOps Center が作るマージコミット（`Merge branch 'X' into staging` / author はステージの org の
Salesforce ユーザー）と形が違う。
[5-2 のログ](2026-08-28-ci-branch-protection.md) の6章で書いた区別が、逆向きにも確認できた。

## 3. コマンドが org を認識しない

```
sf project deploy pipeline start -b staging -p verification -c dcng-prod
→ Error (DevopsAppNotInstalledError): The DevOps Center app wasn't found in the specified org.
  Verify the org username and try again.
```

`-c` に username（`<EMAIL_REDACTED>`）を指定しても同じだった。

最初に試したときのプラグインは **1.2.27** で、npm の最新は **2.2.0**（2026-08-18 更新）だった。
**版を確かめずに叩いていた。**

## 4. プラグインを 2.2.0 にするまで

`sf plugins update` は 1.2.27 のままで、`sf plugins install` は npm のエラーで失敗した。

```
npm error code ERR_INVALID_ARG_TYPE
npm error The "from" argument must be of type string. Received undefined
```

npm のログを見ると `reify:unretire`（既存の `node_modules` を退避して戻す処理）で落ちていた。
`node_modules` には展開されるが `package.json` の `dependencies` に登録されず、
`sf` は毎回「未インストール」と判断して入れ直そうとする。

環境を2つ動かして解決した（**人の判断を仰いでから実行した**）。

| 何 | 変更 |
|---|---|
| node（グローバル） | 22.12.0 → **24.11.0**（`~/.tool-versions`） |
| sf CLI | 2.135.7 → **2.149.9** |

それでも `sf plugins install` は同じエラーで通らなかったので、
ローカルの作業ディレクトリで `npm install` してから `sf plugins link` で登録した。

```
sf plugins link <PLUGIN_WORKDIR>/dcplugin/node_modules/@salesforce/plugin-devops-center
→ devops-center 2.2.0 (link)
```

## 5. 2.2.0 でも同じエラーだった

```
sf project deploy pipeline start -b staging -p verification -c dcng-prod
→ Error (DevopsAppNotInstalledError): The DevOps Center app wasn't found in the specified org.
```

プラグインのコードを読むと、判定は SOQL の成否で行っている。

```javascript
// lib/common/utils.js
stages = await selectPipelineStagesByProject(targetOrg.getConnection(), projectName);
// catch: if (error.name === 'Query-failedError') throw messages.createError('error.DevopsAppNotInstalled');
```

そのクエリがこれである。

```sql
SELECT Id, sf_devops__Pipeline__r.sf_devops__Project__c, sf_devops__Branch__r.sf_devops__Name__c,
       (SELECT Id FROM sf_devops__Pipeline_Stages__r)
FROM sf_devops__Pipeline_Stage__c
WHERE sf_devops__Pipeline__r.sf_devops__Project__r.Name = '...'
```

`sf_devops__Pipeline_Stage__c` は**旧版のマネージドパッケージのオブジェクト**である。
次世代は `DevopsPipelineStage`（名前空間なしの標準オブジェクト）なのでクエリが失敗し、
プラグインはそれを「DevOps Center が入っていない」と解釈する。

昇格のエンドポイントも旧版の Apex REST だった。

```javascript
// lib/common/constants.js
export const REST_PROMOTE_BASE_URL = '/services/apexrest/sf_devops/pipeline/promote/v1/';
export const ASYNC_OPERATION_CDC = '/data/sf_devops__Async_Operation_Result__ChangeEvent';
```

検証 org に旧版パッケージは入っていない。

```
sf data query -o dcng-prod -q "SELECT Id FROM sf_devops__Pipeline__c LIMIT 1"
→ sObject type 'sf_devops__Pipeline__c' is not supported.

sf package installed list -o dcng-prod
→ 0件
```

## 6. ドキュメントは CLI リファレンスにある

コマンドは CLI リファレンスに載っている。

> [project deploy pipeline start (Beta)](https://developer.salesforce.com/docs/platform/salesforce-cli-reference/guide/cli_reference_project_deploy_pipeline_start.html)

内容は `--help` と同じで、次世代への言及は無い。**ページに版や更新日の表示は無い**（`meta` にも無い）。

文言には旧版の前提が出ている。

> The first time you run any "project deploy pipeline" command, be sure to authorize the org
> in which DevOps Center **is installed**.

> You must indicate the bundle version if deploying to the environment that corresponds to
> the first stage after the **bundling stage**.

「インストールされている org」という言い回しと、**bundling stage**（旧版の概念）である。

ドキュメントが CLI リファレンスにあることと、実体がプラグインであることは両立する。
`sf` は `project deploy pipeline` を叩かれた時点で
`@salesforce/plugin-devops-center` を JIT で取りに行く。

```
sf plugins --core
→ devops-center 2.2.0

（未インストール状態でコマンドを叩いたとき）
→ JitPluginInstallError: Could not install @salesforce/plugin-devops-center
```

## 7. 対応を求める issue は4ヶ月放置されている

> [salesforcecli/plugin-devops-center#505](https://github.com/salesforcecli/plugin-devops-center/issues/505)
> **Support for next gen devops center**
>
> will this support the next gen devops center, which is now in beta?

| 項目 | 値 |
|---|---|
| 作成 | 2026-04-24（次世代の GA の2日後） |
| 状態 | `open` |
| コメント | **0件** |
| `updated_at` | `created_at` と同じ（一度も更新されていない） |

## 8. 予測との照合

| 何 | 予測 | 結果 |
|---|---|---|
| CLI はマージをしない | しない | **当たり**（ヘルプに明記） |
| CLI は変更リストを要らない | 要らない | **確かめられなかった**（org の判定で先に落ちた） |
| `WorkItem.Status` が `PROMOTED` にならない | ならない可能性 | 同上 |
| 昇格が `DevopsRequestInfo` に残る | 残る | 同上 |

## 9. 分かったこと

- **`sf project deploy pipeline` は旧版のマネージドパッケージ専用。**
  次世代の org では動かない。プラグインの版（1.2.27 / 2.2.0）を問わない
- 判定はオブジェクト名で行われている。`sf_devops__Pipeline_Stage__c` を引けないと
  「DevOps Center が入っていない」と解釈される
- 対応を求める issue は GA の2日後に立ち、**4ヶ月コメントが付いていない**
- → **GitHub Actions から CLI で昇格する案は、この経路では成立しない**

MCP にもステージ昇格のツールが無い（→ MCPに関する検証）ので、
**ステージ昇格を画面以外から実行する手段が見つかっていない。**

## 10. 未確認として残したもの

- **MCP の `promote_devops_center_work_item` が叩いている API。**
  作業項目の昇格は MCP から動くので、その API を特定すれば
  GitHub Actions から直接呼べる可能性がある。未着手
- 次世代に bundling stage に相当する概念があるか。
  ドキュメントの前提が旧版だと判断した根拠の1つだが、次世代側を調べていない
- `sf plugins install` が `ERR_INVALID_ARG_TYPE` で失敗する原因。
  `sf plugins link` で回避したので追っていない

## 11. 作業の記録に残る誤り

- **プラグインの版を確かめずに叩いた。** 入っていたのは 1.2.27 で、最新は 2.2.0 だった。
  結果は同じだったが、確かめる前に「CLI は次世代で動かない」と判断しかけた
- 08-27 にGitHubルールと自動化に関する検証へ「CLI に昇格のコマンドが揃っている」と書いた。
  `--help` を読んだだけで実行していない。
  **コマンドが存在することと、対象の org で動くことは別である**
- `DevopsAppNotInstalledError` を見た時点で node のバージョンを疑い、
  環境を2つ更新した。原因はプラグインが見るオブジェクト名で、node は関係なかった
  （更新自体は 2.2.0 を入れるために必要だった）

---

## 12. Connect API を特定した（CLI が駄目でも道はあった）

CLI が使えないと分かった時点で切り上げようとしたが、
**MCP は動いているので API は存在する**という指摘を受けて追った。
プラグインのコードを読んだのと同じ方法で、MCP サーバーの実装を読む。

MCP サーバーは `npx @salesforce/mcp@latest` で動いており、実体はここにある。

```
~/.npm/_npx/<hash>/node_modules/@salesforce/mcp                     0.30.15
  node_modules/@salesforce/mcp-provider-devops/dist/                ← DevOps Center のツール群
```

`connect/devops` 配下のエンドポイントを抜き出した（`API_VERSION = 'v65.0'`）。

| 操作 | メソッドとパス |
|---|---|
| 作業項目の作成 | `POST /services/data/v65.0/connect/devops/projects/{projectId}/workitem` |
| 作業項目の更新 | `/services/data/v65.0/connect/devops/projects/{projectId}/workitem/{workItemId}` |
| **同期** | `POST /services/data/v65.0/connect/devops/projects/{projectId}/workitem/{workItemId}/sync` |
| PR の作成 | `/services/data/v65.0/connect/devops/workItems/{workItemId}/review` |
| **昇格** | `POST /services/data/v65.0/connect/devops/pipelines/{pipelineId}/promote` |
| VCS | `/services/data/v65.0/connect/devops/vcs/{vcsType}` |

### 昇格のリクエスト

```javascript
const body = {
    workitemIds: workitems.map(w => w.id),
    targetStageId,
    allWorkItemsInStage,
    isCheckDeploy: false,
    deployOptions   // { testLevel: "NoTestRun", isFullDeploy: false } が既定
};
const path = `/services/data/${API_VERSION}/connect/devops/pipelines/${pipelineId}/promote`;
```

`allWorkItemsInStage` は、対象の作業項目が全部同じステージにあるかで決まる。

```javascript
const uniqueStageIds = Array.from(new Set(workitems.map(w => w.PipelineStageId).filter(Boolean)));
const allWorkItemsInStage = uniqueStageIds.length === 1;
```

**`targetStageId` に本番のステージ Id を渡せば「フェーズを昇格」に相当するはずである**（未検証）。
MCP にステージ昇格のツールが無いだけで、**エンドポイントは同じものが使える見込みが立った。**

`deployOptions` には `testLevel` と `isFullDeploy` があり、`isCheckDeploy` で validate only にできる。

### 同期が `INSPECT` の正体

`syncWorkItem` のコメントにこう書いてある。

> Sync reconciles the work item with VCS (branch/review/CR/**external commit inspection**, etc.).

5-2 で突き止めた `INSPECT`（変更リストを作る処理）がこれである。
MCP の commit もこの sync を呼んでいて、
返り値の `hasUpdates` が「DevOps sync hasUpdates: false」の正体だった。

実際に叩くと通った。

```
sf api request rest "/services/data/v65.0/connect/devops/projects/<projectId>/workitem/<workItemId>/sync" \
  --method POST --body '{}' -o dcng-prod
→ { "hasUpdates": false, "success": "true" }
```

`WI-000015` は画面を開いた後だったので変更リストが既にあり、`hasUpdates` は `false` だった。
**画面を開かずに sync だけで変更リストが作られるかは、別の作業項目で確かめる。**

### 判定の訂正

- 「ステージ昇格を画面以外から実行する手段が見つかっていない」は**言い過ぎだった。**
  CLI プラグインが使えないだけで、**Connect API は存在する**
- **CLI が駄目だと分かった時点で調べるのをやめかけた。** 同じ日に2回使った
  「実装を読む」という手が、MCP にも使えることに気づいていなかった

## 13. Connect API だけで1周できるか

`WI-000016` を作り、**MCP を介さず `sf api request rest` だけ**で進めた。

| 操作 | 叩いたもの | 結果 |
|---|---|---|
| 作業項目の作成 | `POST /connect/devops/projects/{projectId}/workitem` | `WI-000016` |
| 開発環境の割り当て | `sf data update record`（API は見つかっていない） | dev1 |
| 「進行中」にする | `PATCH /connect/devops/projects/{projectId}/workitem/{workItemId}` <br> body `{"status":"IN_PROGRESS"}` | ブランチが作られた |
| PR の作成 | `POST /connect/devops/workItems/{workItemId}/review` <br> body `{}` | PR #20 |

**ここまでは REST だけで通る。**

### 変更リストは REST から作れなかった

`sync` を叩いても変更リストができない。

```
POST /connect/devops/projects/{projectId}/workitem/{workItemId}/sync
body: {}
→ { "hasUpdates": false, "success": "true" }
```

画面の変更リストタブを開いたときのリクエストを見ると、
**画面もまったく同じ `syncWorkItem` を同じパラメータ（`workitemSyncInput: {}`）で呼んでいて、
返り値も `hasUpdates: false` だった。** パラメータの差ではない。

時系列を突き合わせると、`INSPECT` は画面を開いた瞬間に走っている。

| 時刻 | 何 |
|---|---|
| 03:01 頃 | REST で `sync` を叩いた（`INSPECT` は走らない） |
| **03:04:12** | **`INSPECT`**（作業項目の画面を開いた） |
| 03:04:18 | `WorkItemComponentList` と `WorkItemComponent` が作られた |
| 03:04:53 | 画面の `syncWorkItem`（変更リストタブのクリック） |
| 03:05:47 | PR #20 の作成 |

PR より前に `INSPECT` が走っているので、PR も引き金ではない。

公式の記述と符合する。

> VCS Synchronization は「ユーザー操作時」に走る（`viewing a work item` を含む）

**作業項目を「見る」ことがトリガーで、特定の API 呼び出しではない。**

### MCP のコードにも手段が無い

`INSPECT` や `WorkItemComponentList` を扱う実装は MCP に無い。
`checkCommitStatus` は `DevopsRequestInfo` を読むだけ、
`vcs/{vcsType}` は GET でリポジトリの持ち主を返すだけだった。

### 変更リストとは何か

`WorkItemComponent` の実データを見ると、**メタデータ1つが1レコード**である。

| 項目 | 例 |
|---|---|
| `MemberName` | `HybridTest__c.ApiNote__c` |
| `MemberType` | `CustomField` / `CustomObject` / `Layout` / `Profile` / `PermissionSet` / `ApexClass` |
| `ChangeType` | `NEW` / `CHANGE` / `MANUAL` |
| `ChangedBy` | `github-user <EMAIL_REDACTED>`（git のコミッタ） |
| `ChangedOn` | git commit の時刻 |
| `NumberOfRecords` | これまで全件 `None`（データを運んだことがない） |

**git のコミットを読んで、含まれるメタデータをレコード化したもの**である。
昇格はこのレコードを対象に組み立てられるので、
空だと `No components to deploy` になる。

これまでの27件を見て気づいたことが3つある。

- `WI-000001` に**同じ4件が2回ずつ**入っている（`<WORK_ITEM_COMPONENT_ID>`〜`004` と `005`〜`008`）。
  08-07 の最初の作業項目で、コミットが2回か `INSPECT` が二重に走ったと見られる。**未確認**
- `WI-000004` の `ChangeType` が **`MANUAL`**。
  「メタデータを追加」から手で足した操作が、この値として残る
- `VersionNumber` は `WI-000001`〜`005` にだけ入っていて（1〜88）、
  `WI-000006` 以降は全件 `None`。08-07 と 08-26 の間で挙動が変わっている。**理由は未確認**

### 5-5 の答え

**完全な無人化はできない。** 変更リストの生成だけが画面依存で、
CLI にも MCP にも REST にも起こす手段が見つからなかった。

ただし**それ以外はすべて Connect API で叩ける。**
「作業項目を画面で1回開く」だけ人が入れば、残りは自動化できる。

未検証の回避策として、昇格の `deployOptions` に `isFullDeploy: true` を渡す手がある。
ブランチ全体を配るので変更リストを参照しない可能性があるが、
**毎回フルデプロイになるので実務の運用としては勧められない。**

### MCP は旧版と次世代の両方を見ている

MCP のコードが投げる SOQL の対象を数えると、両方が混ざっていた。

```
FROM WorkItem                      次世代
FROM DevopsPipelineStage           次世代
FROM DevopsRequestInfo             次世代
FROM sf_devops__Work_Item__c       旧版
FROM sf_devops__Pipeline_Stage__c  旧版
FROM sf_devops__Work_Item_Promote__c 旧版
```

CLI プラグインが旧版だけを見ているのと対照的である。

## 14. `isFullDeploy: true` なら画面なしで1周できた

13章で「完全な無人化はできない」と書いたが、**`isFullDeploy` を試していなかった。**
運用として勧められないという理由で優先度を下げていたが、
**「画面が必須か」を決める材料としては別である**という指摘を受けて確かめた。

`WI-000017` を作り、**画面を一度も開かずに** `sf api request rest` だけで進めた。
同じ作業項目に対して `isFullDeploy` だけを変えて2回叩いた。

| `isFullDeploy` | `DevopsRequestInfo` | 結果 |
|---|---|---|
| `false` | `ERROR` | `DEPLOYMENT_FAILURE: No components to deploy — the resolved component set was empty.` |
| **`true`** | **`SUCCESS`** | **`WI-000017` が `PROMOTED` になった** |

変更リスト（`WorkItemComponentList`）は**最後まで空のまま**である。

| 何 | 結果 |
|---|---|
| `staging` ブランチ | `Merge branch 'WI-000017' into staging`（DevOps Center がマージした形） |
| PR #21 | `MERGED` |
| stg org | `FullNote__c` が届いた |

**変更リストが要るのは差分デプロイのときだけだった。**
フルデプロイはブランチの内容を直接配るので、変更リストを参照しない。

### 何個配られたか

`IsFullDeploy` は**このときも `false`** で、判別に使えない（08-26 の記録と一致）。
昇格先の org の `DeployRequest` を Tooling API で引くと差が出る。

```
sf data query -o dcng-stg --use-tooling-api \
  -q "SELECT StartDate, Status, NumberComponentsTotal FROM DeployRequest ORDER BY StartDate DESC"
```

| 時刻 | `NumberComponentsTotal` | 何 |
|---|---|---|
| **04:02:28** | **18** | **`isFullDeploy: true`** |
| 03:59:51 | 1 | `WI-000015` の昇格（通常） |
| 01:57:09 | 1 | `WI-000014`（5-2） |
| 01:39:58 | 1 | 5-2 |
| 01:36:06 | 1 | `WI-000013` |

変更したのは `FullNote__c` 1つだけなので、**残り17個は staging ブランチの全メタデータ**である。
リポジトリのメタデータが増えれば、そのぶん毎回配ることになる。

### 画面なしで1周する手順

```
POST   /connect/devops/projects/{projectId}/workitem            作成（返り値の Id は15桁）
（DevelopmentEnvironmentId は sf data update record で設定。API は見つかっていない）
PATCH  /connect/devops/projects/{projectId}/workitem/{id}       {"status":"IN_PROGRESS"}
（git commit と push は自分でやる）
POST   /connect/devops/workItems/{id}/review                    PR の作成
PATCH  /connect/devops/projects/{projectId}/workitem/{id}       {"status":"READY_TO_PROMOTE"}
POST   /connect/devops/pipelines/{pipelineId}/promote           昇格
```

### Id は18桁でないと通らない

作業項目の作成 API が返すのは**15桁**（`<DEVOPS_RECORD_ID>`）だが、
そのまま昇格の `workitemIds` に渡すとエラーになる。

```
PROMOTION_SOURCE_CODE_REPOSITORY_BRANCH_NOT_FOUND:
No source code repository branch found for work item: <DEVOPS_RECORD_ID>
```

18桁（`<DEVOPS_RECORD_ID>`）にすると通る。**作成の返り値をそのまま使えない。**

### 指定と違う作業項目が昇格した

最初の `isFullDeploy: false` の呼び出しで、
`workitemIds` に `WI-000017` の Id を渡したのに、
**`promotedWorkitemIds` に `WI-000015` の Id が返り、`WI-000015` が `PROMOTED` になった。**
`SUCCESS` で返っており、エラーは出ていない。

`WI-000015` は外部マージで昇格待ちのまま残っていた作業項目である（1〜2章）。
2回目の呼び出しでは `WI-000017` が正しく対象になった。

**指定した作業項目と違うものが昇格することがある。** 条件は特定できていない。
結果として `WI-000015` の外部マージは解消され、`CliNote__c` が stg org に届いた。

## 15. 5-5 の答え（13章を訂正）

**画面を開かずに1周できる。ただし毎回フルデプロイになる。**

| | |
|---|---|
| CLI（`sf project deploy pipeline`） | **使えない**（旧版専用） |
| MCP | 作業項目の昇格まで。ステージ昇格のツールは無い |
| **Connect API** | **作成から昇格まで通る** |
| 差分デプロイ | 変更リストが要る。**変更リストは作業項目の画面を開かないと作られない** |
| フルデプロイ | **変更リスト不要。画面も不要** |

13章の「完全な無人化はできない」は**誤りだった**ので、この章で訂正する。
正しくは「**差分デプロイでの無人化はできない。フルデプロイなら無人化できる**」。

未検証のまま残っているもの。

- **ステージ昇格（staging → prod）が API でできるか。**
  本番に送る作業項目は staging へ上がった時点で変更リストを持っているので、
  差分デプロイでも通る可能性がある。`isCheckDeploy: true` なら本番を変えずに試せる
- 「指定と違う作業項目が昇格する」条件

## 16. ステージ昇格（本番リリース）も API で通った

15章で「ステージ昇格が API でできるかは未検証」と残した分を確かめた。
staging には prod 未昇格の作業項目が11件あり、**すべて変更リストを持っている。**

### validate only は結果の保存で失敗する

まず `isCheckDeploy: true` で叩いた。

```json
{
  "workitemIds": [ ...11件... ],
  "targetStageId": "<DEVOPS_RECORD_ID>",
  "allWorkItemsInStage": true,
  "isCheckDeploy": true,
  "deployOptions": { "testLevel": "NoTestRun", "isFullDeploy": false }
}
```

11件すべてが `promotedWorkitemIds` に返り、`SUBMITTED` になった。
`DevopsRequestInfo` は `SUCCESS` が1件、そのあと `ERROR` が1件。

```
{"errorType":"DEPLOYMENT_FAILURE",
 "errorMessage":"Failed to persist validation result — CheckDeployIdentifier unavailable"}
```

**本番 org では validate が実際に走って成功していた。**

```
sf data query -o dcng-prod --use-tooling-api \
  -q "SELECT StartDate, Status, CheckOnly, NumberComponentsTotal, NumberComponentErrors FROM DeployRequest"

<DEPLOY_REQUEST_ID>  2026-08-28T04:48:38  Succeeded  CheckOnly=true  12  errors=0
```

失敗したのは DevOps Center 側に結果を保存する処理だけである。
`DevopsEnvDeployment` は `<RECORD_ID>` が `NEW` のまま残った。
**`isCheckDeploy: true` に固有の不具合に見える。**

### 本番リリースは通った

`isCheckDeploy: false` で叩き直すと完走した。

| 何 | 結果 |
|---|---|
| `DevopsRequestInfo` | `PROMOTE` / `SUCCESS` |
| 作業項目 | 11件すべて **`CLOSED`**（`PROMOTED` は0件になった） |
| prod org | **`HybridTest__c` が届いた** |
| prod の `DeployRequest` | `CheckOnly: false` / **12コンポーネント** / `Succeeded` |
| `main` ブランチ | `Merge branch 'staging'` |

**差分デプロイのまま通っている**（12個。フルの18個ではない）。
本番に送る作業項目は staging へ上がった時点で変更リストを持っているので、
ステージ昇格では新たに `INSPECT` が要らない。

画面は一切触っていない。

## 17. 5-5 の答え（15章に追記）

| 経路 | 画面が要るか |
|---|---|
| 開発環境 → 第1ステージ（差分） | **要る**（変更リストの生成に `INSPECT` が必要） |
| 開発環境 → 第1ステージ（フル） | 要らない。ただし毎回ブランチ全体を配る |
| **第1ステージ → 本番（差分）** | **要らない** |

**本番リリースの自動化はそのままできる。** 引っかかるのは開発の入口だけである。

### 未確認として残るもの

- `isCheckDeploy: true` が `CheckDeployIdentifier unavailable` で失敗する条件。
  本番側の validate は成功しているので、DevOps Center 側の保存処理の問題に見える
- 「指定と違う作業項目が昇格する」条件（14章）
- 開発環境の割り当て（`DevelopmentEnvironmentId`）を設定する Connect API。
  見つからないので `sf data update record` を使っている

## 18. External Merge でも差分デプロイは変更リストを要求する

「staging ブランチに既に変更が入っているなら、変更リストを見ずに配れるのではないか」
という仮説を試した。3-1 の「プロモーションを完了」が
org 未デプロイを解消する操作なので、そこに乗れると考えた。

`WI-000018` を作り、**画面を一度も開かずに**進めた。

| 手順 | 結果 |
|---|---|
| 作成 / `IN_PROGRESS` / 項目のデプロイ / commit / push / PR #22 | すべて REST と git で通った |
| **PR #22 を自分でマージ**（External Merge） | `staging` に `ExtMergeNote__c` が入った |
| 変更リスト | **空** |
| `READY_TO_PROMOTE` にして差分（`isFullDeploy: false`）で昇格 | **失敗** |

```json
{"errorType":"DEPLOYMENT_FAILURE",
 "errorMessage":"No components to deploy — the resolved component set was empty."}
```

**仮説は成り立たなかった。** 差分デプロイは External Merge かどうかに関係なく、
変更リストから対象を組む。

同じ作業項目を `isFullDeploy: true` で叩くと通った（`PROMOTED`）。
**差分だけが変更リストを要求する**という 14章の結論の裏付けになる。

### `WI-000015` が通った理由の訂正

14章で `WI-000015`（External Merge 済み）が API の promote で通ったと書いた。
これを「API なら『プロモーションを完了』を押さずに済む」と読んだが、
**`WI-000015` は画面を開いた後だったので変更リストがあった。**
External Merge のおかげではない。

### 変更リストが要るかの一覧

| 経路 | 変更リスト |
|---|---|
| 開発環境 → 第1ステージ（差分） | **要る** |
| 開発環境 → 第1ステージ（差分・External Merge） | **要る** |
| 開発環境 → 第1ステージ（フル） | 要らない |
| 第1ステージ → 本番（差分） | 要らない（各作業項目が既に持っている） |

## 19. External Merge は他の開発環境の昇格も1回空振りさせる

08-27 の 3-1 で「そのステージから先の昇格が止まる」と書いたが、
**他の開発環境から同じステージへ昇格する場合も止まるか**は未確認だった。
並行開発を再現して確かめた。

### 手順

| # | やったこと |
|---|---|
| 1 | `WI-000019`（dev1）と `WI-000020`（dev2）を作り、両方 PR まで進める（#24 / #25） |
| 2 | **PR #24 だけ自分でマージ**（External Merge。解消しない） |
| 3 | `WI-000020` の画面を開いて変更リストを作る（1件） |
| 4 | `WI-000020` を差分で昇格 |

2 の時点の状態を確認してある。

- `staging` ブランチ: `BlockANote__c` が**ある**
- stg org: `BlockANote__c` が**無い**（未デプロイ）
- 両方の作業項目: `IN_REVIEW`

### 結果

| 回 | `workitemIds` に渡した Id | 実際に昇格したもの |
|---|---|---|
| 1回目 | `WI-000020` | **`WI-000019`**（External Merge 待ちの方） |
| 2回目 | `WI-000020` | `WI-000020` |

1回目のレスポンスは `SUBMITTED`、`DevopsRequestInfo` は `PROMOTE` / **`SUCCESS`**。
**エラーは一切出ない。**
`WI-000019` が `PROMOTED` になり、stg org に `BlockANote__c` が届いた（External Merge が解消された）。
`WI-000020` は `READY_TO_PROMOTE` のまま残った。

2回目で `WI-000020` が通った。

### 分かったこと

**External Merge の未解消があると、他の開発環境からの昇格も1回空振りする。**
「止まる」というより、**指定した作業項目が黙って無視され、
未解消の External Merge が代わりに処理される。**

実務ではこうなる。

1. A が自分で PR をマージして放置する
2. B が別の開発環境から昇格する
3. **成功したように見えるが、B の変更は配られていない**（A の分が配られる）
4. B がもう一度昇格すると通る

**成功と表示されて何も起きないのが危ない。**

14章で「指定と違う作業項目が昇格する条件は特定できていない」と書いたが、
**未解消の External Merge があること**が条件だった。
`WI-000015` のときも同じ状況だった（自分で PR #19 をマージした直後）。

3-1 の「そのステージから先の昇格が止まる」は、
**同じステージへの昇格も1回空振りする**まで含めて読む必要がある。

## 20. `isFullDeploy: true` は後段で取りこぼす

片付けとして `WI-000018` / `WI-000019` / `WI-000020` の3件を本番へステージ昇格したあと、
prod org を retrieve して確認したら**`ExtMergeNote__c` が届いていなかった。**

| | 変更リスト | stg org | `main` ブランチ | **prod org** |
|---|---|---|---|---|
| `ExtMergeNote__c`（`WI-000018`） | **0件** | ある | ある | **無い** |
| `BlockANote__c`（`WI-000019`） | 1件 | ある | ある | ある |
| `BlockBNote__c`（`WI-000020`） | 1件 | ある | ある | ある |

`WI-000018` は 18章で `isFullDeploy: true` で第1ステージに昇格した作業項目である。
**フル昇格では変更リストが作られないので、その先のステージ昇格で運ばれない。**

**git（`main` ブランチ）には入っているのに、prod org には無い。**
git と org がずれた状態が残る。

`isFullDeploy: true` の問題は「毎回ブランチ全体を配る」だけではなかった。
**後段の昇格で取りこぼす。**

### 分からないこと

`WI-000017` も変更リストは0件（`isFullDeploy: true` で昇格した）だが、
`FullNote__c` は prod に届いている。
16章で11件をまとめて昇格したときの12コンポーネントに含まれていた。
**同じ条件で片方は届き、片方は届かない理由が分からない。**

違いとして考えられるのは `allWorkItemsInStage` の扱いだが、
どちらの呼び出しも `true` を渡している。未確認。

## 21. R-6 の入口を試した（設計を3回直した）

20章で作ったずれ（`ExtMergeNote__c` が `main` にあるのに prod org に無い）を材料に、
R-6（本番と git のずれを検知する仕組み）の入口を試した。

### 1回目: `-d force-app` は org 側だけのものを取れない

```
sf project retrieve start -o dcng-prod -d force-app
```

`-d` は「ローカルにあるメタデータを org から取り直す」ので、
**org 側にしかないものは見えない。**

ずれは検知できたが、`git diff` ではなく retrieve の warning に出た。

```
│ unpackaged/package.xml │ Entity of type 'CustomField' named
│                        │ 'HybridTest__c.ExtMergeNote__c' cannot be found │
```

`git diff` には819行の差分が出たが、**全部ノイズだった。**

| 差分 | 中身 |
|---|---|
| 項目ファイル（8個 × 1行） | `<trackHistory>false</trackHistory>` が消える（org は既定値を返さない） |
| `HybridTestReader.cls`（2行） | 末尾の改行が消える |
| `HybridTest__c.object`（151行） | `actionOverrides` が付く（org が自動生成する） |
| `Account.object`（279行） | 同様 |
| `Admin` / `Read Only` プロファイル（389行） | 権限の羅列（プロファイルは org 全体を返す） |

### 2回目: manifest で型を指定する

「git で管理する範囲」を manifest として宣言し、それで retrieve する形にした。
`CustomObject`（`Account` と `HybridTest__c`）/ `ApexClass` / `PermissionSet` /
`Layout` / `Profile` の5種類。

```
sf project retrieve start -o dcng-prod -x drift-manifest.xml
```

**両方向のずれが `git status` に出た。**

| 記号 | 件数 | 意味 |
|---|---|---|
| `D` | **1件** | **`ExtMergeNote__c`**（git にあって org に無い） |
| `??` | 29件 | org にあって git に無い |
| `M` | 13件 | 内容が違う（大半はノイズ） |

### 3回目に分かったこと: 事前準備が前提だった

`??` の29件は `Account` の標準項目26個と listViews / webLinks だった。
**`CustomObject` を丸ごと指定すると標準の付属物まで来る。**

ただし根本の原因は別にある。
**この検証環境は空リポジトリから始めて作業項目で少しずつ足したので、
`main` には「作業項目で運んだもの」しか入っていない。**
`Account` のカスタム項目は4個だけで、標準項目は1つも無い。

正しい運用では、**最初に本番から git 管理対象のメタデータを全部 retrieve して
`main` に入れる**ところがスタートになる。そうすれば `git` と org が一致した状態から始まり、
`??` は「本当に後から org に追加されたもの」だけになる。

→ **R-6 は事前準備が正しくできていることを前提にした仕組みである。**
準備をサボると検知がノイズで埋まる。

### 作業の誤り

`find force-app -type f` で「git に入っているメタデータ」を数えたが、
**そこには1回目の retrieve で作られた未追跡ファイルが混ざっていた**
（`git checkout -- .` は変更を戻すが、新規ファイルは消さない）。
`Account` の項目を26個と数えたのは誤りで、`git ls-tree -r HEAD` で見ると4個だった。

### 残っていること

- manifest の粒度。型だけでは標準の付属物が混ざる。
  `.forceignore` で除くか、`CustomField` を名前で列挙するか
- ノイズ（`trackHistory` / `actionOverrides` / プロファイル）を無視する仕組み
- GitHub Actions で定期実行する形

## 22. 事前準備をやり直して検知がクリーンになるか

21章で「R-6 は事前準備が前提」と分かったので、**正しい準備を再現して前後を比べた。**

### やったこと

1. `manifest/package.xml` を書いて git に置いた（管理する範囲の宣言）
2. prod から manifest 指定で retrieve し、結果をそのまま `main` に直接コミットして push
   （30ファイル → 67ファイル。`Account` の標準項目26個 / listViews / webLinks が入った）
3. `main` を clone し直して同じ manifest で retrieve し、差分を見た

**`main` への直接 push は通る。** DevOps Center は検知しない（08-27 の記録と一致）。

### 準備の前後

| | 準備なし | **準備あり** |
|---|---|---|
| `??`（org にあって git に無い） | **29件** | **0件** |
| `M`（内容が違う） | **13件（819行）** | **0件** |
| `D`（git にあって org に無い） | 1件 | 1件 |

**ノイズが消えて、本物のずれ1件だけが残った。**

```
 D force-app/main/default/.gitkeep
 D force-app/main/default/objects/HybridTest__c/fields/ExtMergeNote__c.field-meta.xml
```

`.gitkeep` は `force-app/main/default` を空でも残す目印で、retrieve でディレクトリを作り直すと消える。
**実務では `.forceignore` に入れるか、そもそも置かない。**

### `main` だけ進むと `staging` が遅れる

準備をすると `main` が51ファイル進み、`staging` が遅れた。
**昇格の経路は `staging` → `main` の一方向なので、`main` の内容を `staging` に運べない。**

`staging` に `main` をマージすれば揃う。

```
git checkout staging && git merge origin/main && git push origin staging
```

**`staging` への直接 push も DevOps Center は検知しない**（External Merge は PR のマージのこと）。
org には影響せず、git のブランチを揃えるだけである。

> 「経路が無い」と書いたのは**昇格の経路**の話で、git のマージは別だった。混同していた。

### フル昇格で取りこぼしを回収した

`isFullDeploy: true` の本来の用途（ブランチと org を揃える）を実演した。
フル昇格は昇格待ちの作業項目が1つ必要なので、`WI-000021` を乗り物として作った。

| 何 | 結果 |
|---|---|
| 第1ステージへフル昇格 | `PROMOTED`（変更リストは空のまま通る） |
| 本番へフル昇格 | `CLOSED` |
| prod に配られた数 | **60コンポーネント**（差分昇格は2） |
| `ExtMergeNote__c` | **prod org に届いた**（`D` が消えた） |

### 回収後に残った差分

| 差分 | 中身 |
|---|---|
| `ExtMergeNote__c` / `VehicleNote__c` | `<trackHistory>false</trackHistory>` が消える |
| `Admin` / `Read Only` プロファイル | `ExtMergeNote__c` の `fieldPermissions` が**増えた** |
| `sfdcInternalInt__sfdc_slack` 権限セット | 同様 |

**項目を1つ配ると、org 側でプロファイルと権限セットに `fieldPermissions` が自動で足される。**
それが git に無いので差分になる。

### R-6 に残っている課題

- **`trackHistory` とプロファイルの `fieldPermissions` は、デプロイのたびに必ず出る。**
  「差分が出たら通知」では毎回鳴るので使えない。無視する仕組みが要る
- manifest の粒度。`CustomObject` を丸ごと指定すると標準の付属物まで来る
- GitHub Actions で定期実行する形

## 23. ノイズの原因は開発時に retrieve を挟んでいないこと

22章で「`trackHistory` とプロファイルの `fieldPermissions` はデプロイのたびに必ず出る」と書いたが、
**原因は製品ではなく開発の手順だった。**

今日やっていたのはこれである。

1. ローカルで XML を手書きする（`<trackHistory>false</trackHistory>` を書いた）
2. dev1 に deploy する
3. **retrieve せずにコミットする**

git に入るのが「手書きの形」で、org が返す形と違うので後の retrieve で差分になる。

### deploy の直後に retrieve すれば揃う

`VehicleNote__c` で確かめた。

```
（git にある手書きの形）
    <trackHistory>false</trackHistory>
    <trackTrending>false</trackTrending>

sf project retrieve start -o dcng-dev1 -m "CustomField:HybridTest__c.VehicleNote__c"
→ Changed

（retrieve 後）
    <trackTrending>false</trackTrending>
```

**`trackHistory` が消えて org が返す形になった。**
この状態でコミットすれば、以後どの org から retrieve しても差分は出ない。

プロファイルも取れる。

```
sf project retrieve start -o dcng-dev1 -m "Profile:Admin" --ignore-conflicts
→ Changed / VehicleNote__c の fieldPermissions が1件入った（総数16件）
```

`Profile` 単独で取れた（`CustomObject` の併記は不要）。
なお `sf` のソース追跡がローカルの変更を検知して `Conflict` で止まるので、
`--ignore-conflicts` が要る。

### 対策の形

**開発者の手元で deploy と retrieve を必ず対にする。**

```bash
sf project deploy start -o $ORG -d $PATH && \
sf project retrieve start -o $ORG -d $PATH --ignore-conflicts
```

API バージョンで XML の形が変わる問題も同じ構図なので、固定しておく。

| 場所 | 効果 |
|---|---|
| `manifest/package.xml` の `<version>` | retrieve する API バージョンを固定 |
| `sfdx-project.json` の `sourceApiVersion` | ローカルのソース形式を固定 |

上げるときは意図的に上げて、そのとき出る差分をまとめてコミットする。

### CI 化の方向（未着手）

**スクラッチ組織を作って deploy → retrieve し、git の形と一致するかを検証する。**
CI が既存の org に依存しなくなるので、作業項目ごとに開発環境が違う問題を避けられる。

今回は実装しない。**こういう問題があるので、こうすると良さそうだという段階。**
