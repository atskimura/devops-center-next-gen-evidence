# 2026-08-28 デプロイの失敗を MCP に解かせる（9-3）

- 目的: `resolve_devops_center_deployment_failure` に[デプロイ失敗の検証](2026-08-28-deploy-failure-4-1.md)で作った失敗を渡す。
- 使った org: `dcng-prod`（Hub）
- **実装を読んで予測を書いてから実行した。**

---

## 1. ツールは AI が修正するものではない

説明文の原文。

> Determine if **full promotion** can fix a deployment failure and guide the user.

**「フル昇格で直るか」を判定するだけである。** 修正案を作るわけではない。

判定は4分岐（`resolveDeploymentFailure.js`）。

| 条件 | 返る理由 |
|---|---|
| `MERGE_CONFLICT` / `CONFLICTS:` を含む | `merge_conflict`（別ツールを使え） |
| 依存を解析できない | `no_dependency_parsed` |
| 解析できたがソースブランチに無い | `dependency_not_in_source_branch` |
| 解析できてソースブランチにある | `dependency_in_source_branch`（フル昇格で直る） |

## 2. 実際のエラーでは何も解析できなかった

4-1 のエラーをそのまま食わせた。

```
{"errorType":"DEPLOYMENT_FAILURE","errorMessage":"HybridTest__c.DepFormula__c: 項目DepBase__cは存在しません。スペルを確認してください。 (objects/HybridTest__c.object)"}
```

返ってきたもの（原文）。

> Full promotion **cannot** fix this failure based on the check.
>
> **Generic instructions to resolve the deployment failure:**
> - Review the deployment error details and identify any missing metadata or dependencies.
> - Add missing components in a separate work item, promote that work item first, then retry promoting "WI-000022".
> - If there are merge conflicts, use **resolve_devops_center_merge_conflict** with workItemName: "WI-000022", then commit, push, and retry.
> - If the error persists, check pipeline logs or contact your DevOps admin.

**汎用の手順だけで、依存の名前が出てこない。**

## 3. 英語に直すと解析できる

同じ内容を英語の文言に直して叩いた。

```
"HybridTest__c.DepFormula__c: no CustomObject named DepBase__c found (objects/HybridTest__c.object)"
```

返答が変わった。

> Full promotion **cannot** fix this failure based on the check.
> **The missing dependency "DepBase__c" was not found in source branch "WI-000022".**

**依存の名前を拾って、ソースブランチに無いことまで判定している。**

| エラーの文言 | 判定の理由 | 依存の名前 |
|---|---|---|
| 日本語（org が実際に返すもの） | `no_dependency_parsed` | **出ない** |
| 英語に直したもの | `dependency_not_in_source_branch` | **`DepBase__c` を拾う** |

## 4. 原因は英語の正規表現

`parseMissingDependency`（`shared/dependencyInBranch.js`）が拾えるパターン。

```javascript
/Variable\s+does\s+not\s+exist:\s*([A-Za-z0-9_]+)/i
/no\s+(ApexClass|ApexTrigger|ApexPage|CustomObject|Flow)\s+named\s+([A-Za-z0-9_]+)\s+found/i
/apexClass\s*-\s*no\s+ApexClass\s+named\s+([A-Za-z0-9_]+)\s+found/i
/Type:\s*(ApexClass|ApexTrigger|ApexPage|Profile|CustomObject|Flow)/i + /Component:\s*([A-Za-z0-9_]+)/i
```

**日本語 org のエラーはどれにも当たらない。**

扱えるメタデータの型も6つだけで、**`CustomField` が入っていない**。

```
ApexClass / ApexTrigger / ApexPage / Profile / CustomObject / Flow
```

3章で英語に直したときに `no CustomObject named DepBase__c found` と書いたので拾えたが、
**実際の項目の欠落（`CustomField`）は型としても対象外**である。

## 5. 予測との照合

| 何 | 予測 | 結果 |
|---|---|---|
| 判定 | `canFix: false` | **当たり** |
| 理由 | `no_dependency_parsed` | **当たり**（英語に直すと理由が変わることで裏付けた） |
| 返るもの | 汎用の手順 | **当たり** |
| 実質 | この失敗は解けない | **当たり** |

## 6. 分かったこと

- **このツールは AI ではない。** 正規表現でエラーを解析し、
  ソースブランチにファイルがあるかを `git` で見て、フル昇格で直るかを判定するだけ
- **日本語 org では機能しない。** エラーの文言が英語の正規表現に当たらないので、
  依存の名前すら取れない
- **項目（`CustomField`）の欠落は型としても対象外。** 扱えるのは6種類だけ
- 解けない場合に返るのは汎用の手順で、**内容は一般的なデプロイ失敗の対処**
  （欠けているものを別の作業項目で先に昇格しろ、など）
- 08-27 の 9-2（コンフリクト）と同じ構図である。
  **MCP の DevOps Center ツールは AI で解決しない。判定と手順書を返す**

## 7. 未確認として残したもの

- 英語 org なら実用になるか。`ApexClass` の欠落なら正規表現に当たる
- `dependency_in_source_branch`（フル昇格で直る）と判定される条件を実際に作れるか。
  ソースブランチにはあるが昇格先の org に無い状態が必要で、
  08-28 の 20章で作った `ExtMergeNote__c` の状況が近い
