# 2026-08-28 GitHub リポジトリを組織に移管する

- 目的: パイプラインに接続したリポジトリの持ち主が変わったとき、DevOps Center が追従するか。
  個人アカウント `github-user` から組織 `example-company` への transfer で試す
- 使った org: `dcng-prod`（Hub）
- 動機は2つある。
  個人リポジトリではブランチ保護の検証に制約があったこと、
  企業でリポジトリの持ち主が変わる場面が実際にあること。

---

## 1. transfer の前に作業中の作業項目を片付けた

`WI-000011`（`dev1 で git merge → deploy で取り込む（3-11）`）が `IN_PROGRESS` で残っていた。
作業項目の詳細画面の「その他のアクション」を開くと、選択肢は1つだけだった。

- 状況を [対象外] に変更

押すと確認モーダルが出た。

> 作業項目 WI-000011 の状況を [対象外] に変更しますか?
>
> WI-000011 は無効になり、確定済みのメタデータおよびデータは利用可能は変更リストに戻されます。このアクションは元に戻すことができません。

（原文のまま。「データは利用可能は」は画面の表記）

→ [スクショ](screenshots/2026-08-28-01-対象外に変更の確認.png)

「[対象外] に変更」を押すと `Status` が `NEVERED` になった。
モーダルは a11y ツリーにも出ていたので、`take_snapshot` で `button "[対象外] に変更"` を掴んで押せた。

transfer 前の作業項目の状況は次のとおり。

| 状況 | 件数 | 作業項目 |
|---|---|---|
| `CLOSED` | 4 | `WI-000001` / `WI-000003` / `WI-000004` / `WI-000005` |
| `NEVERED` | 2 | `WI-000002` / `WI-000011` |
| `PROMOTED` | 5 | `WI-000006`〜`WI-000010` |

`PROMOTED` の5件は staging まで昇格して prod へは未昇格の状態で残る。
**作業中（`NEW` / `IN_PROGRESS` / `IN_REVIEW` / `READY_TO_PROMOTE`）は0件**にしてから transfer に進んだ。

## 2. transfer 前のリポジトリ関連レコード

リポジトリの持ち主を持つレコードは `SourceCodeRepository` の1件だけだった。

```
sf data query -o dcng-prod -q "SELECT Id, Name, RepositoryOwner, ExternalRepositoryName, DefaultBranch, Provider, ExternalId FROM SourceCodeRepository"
```

| 項目 | 値 |
|---|---|
| `Id` | `<DEVOPS_RECORD_ID>` |
| `Name` | `example-project` |
| `RepositoryOwner` | `github-user` |
| `ExternalRepositoryName` | 空 |
| `DefaultBranch` | 空 |
| `Provider` | `GITHUB` |
| `ExternalId` | 空 |

リポジトリ名は `ExternalRepositoryName` ではなく `Name` に入っていた。
`DefaultBranch` も空で、既定ブランチはこのレコードでは持っていない。

持ち主やリポジトリの場所を持つ項目が他のオブジェクトにもあるかを確かめた。
`Devops*` / `Source*` / `WorkItem*` で始まるオブジェクトは93個あるので、
リポジトリ関連のものを `describe` して `Repo|Owner|Branch|Url|Provider|External` に当たる項目名を拾った。

| オブジェクト | 該当した項目 |
|---|---|
| `SourceCodeRepository` | `RepositoryOwner` / `ExternalRepositoryName` / `DefaultBranch` / `Provider` / `SourceCodeRepoProviderId` |
| `SourceCodeRepoProvider` | `BaseUrl` / `Provider`（**レコードが0件**） |
| `DevopsPipeline` | `SourceCodeRepositoryId`（参照のみ） |
| `SourceCodeRepositoryBranch` | `SourceCodeRepositoryId`（参照のみ） |
| `WorkItem` | `RebaseBranchId` / `SourceCodeRepositoryBranchId`（参照のみ） |
| `DevopsPipelineStage` | `SourceCodeRepositoryBranchId`（参照のみ） |
| `DevopsProject` / `DevopsRequestInfo` | 無し（`OwnerId` はレコード所有者） |

`SourceCodeRepoProvider` は0件で、`SourceCodeRepository.SourceCodeRepoProviderId` も空だった。
持ち主の文字列を持つのは `SourceCodeRepository.RepositoryOwner` の1件だけで、
残りはすべて Id 参照だった。
書き換えで復旧を試す場合の対象もこの1レコードで足りる
（`RepositoryOwner` は `updateable = True`。ただし公式手順の外。→ セットアップに関する検証）。

## 3. transfer 後の GitHub 側

移管は人が GitHub の画面で実行した。

```
$ gh api /repos/example-company/example-project --jq '{full_name, owner: .owner.login, plan: .owner.type}'
{"full_name":"example-company/example-project","owner":"example-company","plan":"Organization"}

$ gh api /repos/example-company/example-project --jq '.full_name'
example-company/example-project
```

旧パスへの API アクセスは新しい場所を返す。
GitHub がリダイレクトを張るので、旧 URL を持っているクライアントでも追従できる形にはなっている。

DevOps Center 側は画面を一切触らずに作業項目を1つ通すことにした。
先に再接続を試すと、追従したのか再接続で直ったのかが分からなくなる。

## 4. 作業項目を「進行中」にする操作が止まった

MCP から作業項目を作った。

```
create_devops_center_work_item(projectId=<DEVOPS_RECORD_ID>,
  subject="リポジトリを組織へ移管した後に昇格が通るか")
→ { "success": true, "subject": "..." }
```

`WI-000012` ができた（`Status = NEW`）。
開発環境は `null` で作られるので、前回と同じく API で dev1 を設定した。

```
sf data update record -s WorkItem -i <DEVOPS_RECORD_ID> \
  -v "DevelopmentEnvironmentId=<DEVOPS_RECORD_ID>" -o dcng-prod
→ Success
```

ここで**ブランチが作られなかった**。
`SourceCodeRepositoryBranchId` は空のままで、GitHub 側にも `WI-000012` ブランチは現れない。
45秒待っても変わらず、`DevopsRequestInfo` には新しいレコードが1件も積まれなかった
（直近は 08-27 14:32 の `PROMOTE`）。

作業項目の画面を開いても変わらなかった。
画面の「ブランチ」欄は空で、エラーの表示も無い。

→ [作業項目の画面](screenshots/2026-08-28-02-transfer後の作業項目画面.png)

状況が「新規」だったので「[進行中] とマーク」を押した。
**押しても何も起きない。** 状況は「新規」のまま、15秒後の表示にもエラーは出なかった。

→ [押した後の画面](screenshots/2026-08-28-03-進行中とマークを押した後.png)

## 5. 画面に出ないエラーの本文

`aura` のリクエストを見ると、`DevopsConnect.updateWorkItem` は HTTP 200 で
`state: "ERROR"` を返していた。

```json
"message": "SWITCHING_WORKITEM_FAILED:Failed to switch work item context: Unexpected response from named credential callout: {\"message\":\"Moved Permanently\",\"url\":\"<PRIVATE_REPOSITORY_API_URL>\",\"documentation_url\":\"https://docs.github.com/rest/guides/best-practices-for-using-the-rest-api#follow-redirects\"}"
```

`statusCode` は 400、`errorCode` は `INTERNAL_ERROR`。
リクエストの本文は `{"status":"IN_PROGRESS"}` への更新だった。

読み取れることが3つある。

1. DevOps Center が呼ぶのは `api.github.com/repositories/<数値 ID>/...` で、
   リポジトリ名ではなく**リポジトリ ID ベース**の URL である
2. GitHub は移管後に `301 Moved Permanently` を返すが、
   **named credential の callout がリダイレクトを追わない**ため失敗する
3. GitHub 側のエラー本文には
   「リダイレクトに従うこと」を勧めるドキュメントの URL まで入っている

止まった場所は、ブランチを切る前段の「作業項目のコンテキストを切り替える」処理だった。
`switch work item context` が最初に叩くのは `branches/staging` で、
作業項目ブランチは staging の先端から切られるという既知の挙動と符合する。

**画面には何も出ない。** 押しても状況が変わらないだけで、原因は開発者ツールでしか読めなかった。

## 6. `RepositoryOwner` を書き換えると直った

エラーの URL がリポジトリ ID ベース（`repositories/<REPOSITORY_ID>`）だったので、
持ち主の文字列を直しても効かないと予測した。
リポジトリ ID は移管では変わらないからである。

```
sf data update record -s SourceCodeRepository -i <DEVOPS_RECORD_ID> \
  -v "RepositoryOwner=example-company" -o dcng-prod
→ Success
```

**予測は外れた。** 作業項目の画面を開き直して「[進行中] とマーク」を押すと通った。

| | 書き換え前 | 書き換え後 |
|---|---|---|
| `WorkItem.Status` | `NEW` | `IN_PROGRESS` |
| `SourceCodeRepositoryBranchId` | 空 | `<DEVOPS_RECORD_ID>` |
| GitHub 側の `WI-000012` ブランチ | 無い | ある |

DevOps Center が ID ベースの URL を組む前に持ち主を見ているのかは追えていない。
効いたという事実だけが確認できた。

復旧に要したのは**画面操作なしの1コマンド**で、
パイプラインもプロジェクトも作り直していない。

## 7. 移管先のリポジトリに push できるか

`HybridTest__c` に `TransferNote__c`（表示ラベル「移管後メモ」）を1つ足して dev1 にデプロイした。

ここで確認方法を間違えた。
デプロイ後に項目が入ったかを3通りで確かめると、いずれも「無い」と返した。

```
sf sobject describe -o dcng-dev1 -s HybridTest__c
→ カスタム項目は0個

sf data query -o dcng-dev1 -q "SELECT TransferNote__c FROM HybridTest__c LIMIT 1"
→ No such column 'TransferNote__c' on entity 'HybridTest__c'

sf data query -o dcng-dev1 -q "SELECT QualifiedApiName FROM FieldDefinition WHERE EntityDefinitionId='HybridTest__c'"
→ 標準項目だけ（AdminNote__c なども出ない）
```

デプロイは `Status: Succeeded` で全項目が `Unchanged` だったので、
表示と org の実態が食い違っていると考えた。

retrieve で org から取り直すと、**4項目すべて存在した**。

```
sf project retrieve start -o dcng-dev1 -m "CustomObject:HybridTest__c"
→ Created: AdminNote__c / EngineerNote__c / ReviewerNote__c / TransferNote__c / HybridTest__c
```

`Unchanged` の表示は正しく、間違っていたのは確認の手段だった。

「無い」と返った理由は項目レベルセキュリティである。
最初は「デプロイ直後だから反映が遅れている」と考えたが、
`AdminNote__c` は 08-27 にデプロイしたものなので説明が付かない。
`FieldPermissions` を見ると、プロファイル管理分（`Parent.IsOwnedByProfile = true`）が**0件**だった。

| 項目 | 読み取りを持つ権限セット |
|---|---|
| `AdminNote__c` | `HybridTestAccess`（ハイブリッド検証アクセス）と `sfdc_slack` |
| `EngineerNote__c` / `ReviewerNote__c` / `TransferNote__c` | `sfdc_slack` だけ |

`sfdc_slack`（Slack インテグレーションユーザー）は org が自動で管理するものである。
admin user への割り当ては `AccessDevOpsCenterNamedCredentials` と自動生成の1件だけで、
`HybridTestAccess` は入っていない。

つまり**システム管理者の SOQL からも項目が見えない**。
Metadata API（retrieve）は項目レベルセキュリティを見ないので、org の実態を返す。

これは B-0 / R-1 で確認した「配った先では FLS 0件で誰にも見えない」と同じ現象で、
今回は自分の確認作業がその状態に引っかかった
（→ [R-1 のログ](2026-08-27-recovery-r1.md) / [FLS の追跡](2026-08-26-fls-tracking.md)）。

**org の実態を見るなら retrieve を使う。**

コミットは MCP から実行した。

```
commit_devops_center_work_item(workItemName="WI-000012",
  commitMessage="移管後の疎通確認として TransferNote__c を追加")
→ Commit SHA: <COMMIT_SHA>
  DevOps sync hasUpdates: false

git push origin HEAD
→ <COMMIT_SHA>..<COMMIT_SHA>  HEAD -> WI-000012
```

移管先のリポジトリへの push が通った。
PR も `example-company` 側に作られた。

```
create_devops_center_pull_request(workItemName="WI-000012")
→ reviewUrl: <PRIVATE_REPOSITORY_URL>
   status: Success
```

作業項目の画面に出るブランチのリンクも `<PRIVATE_REPOSITORY_URL>` になっていた。
持ち主を書き換えたあとは、DevOps Center が組む URL も新しい場所を指す。

## 8. 昇格の前に必要な2手

`promote_devops_center_work_item` をこの時点で呼ぶと失敗した。

```json
{"errorType":"DEPLOYMENT_FAILURE",
 "errorMessage":"No components to deploy — the resolved component set was empty. Verify that the changed metadata entries match files in the project directory."}
```

同じエラーは 08-27 の 3-7 でも出ていて、そのときはリトライで通った
（→ [3-7 のログ](2026-08-27-run-back-sync-3-7.md)）。

作業項目は `IN_REVIEW` のままだった。
画面には次の2つが出ていた。

- 「ソース制御の取り込み要求を表示して承認します。」（リンク先は PR #16）
- 「[昇格準備完了] とマーク」ボタン

変更リストタブを開くと**1件が入っていた**（`HybridTest__c.Tra...` / `CustomField` / 操作 `NEW`）。

> このとき「足りなかったのは状況の遷移だ」と書いたが誤り。
> 変更リスト（`WorkItemComponentList`）は**このタブを開いた 15:51:16 に作られていた**。
> 昇格が失敗したのは 15:50:19 で、その時点では存在しなかった
> （→ [5-2 のログ](2026-08-28-ci-branch-protection.md) で切り分けた）

1件だけという点は、1章で閉じた `WI-000011` の確認モーダルの文言と関係する。
「確定済みのメタデータおよびデータは利用可能は変更リストに戻されます」と書かれていたので、
`WI-000011` の変更が `WI-000012` の変更リストに現れることを想定していた。
現れたのは今回作った `TransferNote__c` だけである。
`WI-000011` はコミットしていない作業項目だったので、戻す対象が無かったとも読める。

→ [変更リスト](screenshots/2026-08-28-05-変更リスト.png)

「[昇格準備完了] とマーク」を押すと通った。

> 成功
> 作業項目状況が正常に更新されました。

承認の記録も画面に出た（「承認者 admin user 日付: 2026年8月28日 0:52」）。
PR を GitHub で承認する操作はしていない。

→ [昇格準備完了にした後](screenshots/2026-08-28-06-昇格準備完了を押した後.png)

`READY_TO_PROMOTE` にしてから `promote` を呼び直すと通った。

| | 結果 |
|---|---|
| `DevopsRequestInfo` | `PROMOTE` / `SUCCESS`（15:52:19） |
| `WorkItem.Status` | `PROMOTED` |
| PR #16 | `MERGED`（15:52:34 / タイトルは `[DevOps Center] Merge WI-000012 to staging`） |
| `staging` ブランチ | `TransferNote__c.field-meta.xml` がある |
| staging org | `TransferNote__c` がある（retrieve で確認） |

失敗した `promote` から通った `promote` までは2分。
移管後に手を入れたのは `RepositoryOwner` の1コマンドだけである。

## 9. 分かったこと

- **リポジトリを移管すると DevOps Center は止まる。**
  GitHub は `301 Moved Permanently` を返すが、named credential の callout が追わない
- **止まり方が分かりにくい。** ボタンを押しても状況が変わらないだけで、
  画面にエラーが出ない。本文は開発者ツールの `aura` 応答にしかない
- **画面からリポジトリを差し替える手段は無い。**
  差し替えられないということは、公式の答えはパイプラインの作り直しである
  （旧版からの移行についても公式は
  「set up projects, pipelines, and environments again」と書いている。→ セットアップに関する検証）
- `SourceCodeRepository.RepositoryOwner` を書き換えると1コマンドで直り、
  パイプラインもプロジェクトも作り直さずに昇格まで到達した。
  **ただしこれは公式手順の外で、副作用の有無も確かめていない**
- 持ち主の文字列を持つレコードはこの1件だけで、他はすべて Id 参照だった
- 未確認: `ExternalRepositoryName` と `DefaultBranch` は空のまま動いている。
  リポジトリ名まで変える移管では、`Name` の書き換えも要るかは試していない
- 未確認: **接続している GitHub ユーザーが移管先にアクセスできることが前提になっている。**
  今回は移管元と移管先の両方に権限を持つアカウントで接続していた。
  GitHub App や接続の側で追加の承認が要る場合があるかは確かめていない
  （`gh api /user/installations` はユーザートークンでは 403 で、App の設置状況が取れなかった）

## 10. 作業の記録に残る誤り

- `promote` を `IN_REVIEW` の状態で呼んで `No components to deploy` を踏んだ。
  この経路は 08-27 にも踏んでいて、そのときは原因を追えずリトライで通していた
- デプロイした項目の実在を `describe` / `SOQL` / `FieldDefinition` で確かめて、
  3つとも「無い」と返ったのを org の状態だと考えた。
  実際には入っていて、retrieve だけが正しく返した
- その食い違いを「デプロイ直後だから反映が遅れている」と書いた。
  1日前にデプロイした項目も同じく見えなかったので、この説明は成り立たない。
  原因は項目レベルセキュリティだった
- 移管後の復旧について「リポジトリ ID ベースの URL なので持ち主の書き換えは効かない」と予測して外した
