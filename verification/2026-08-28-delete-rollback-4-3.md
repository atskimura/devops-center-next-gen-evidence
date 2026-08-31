# 2026-08-28 昇格した項目を削除して org から消えるか（4-3）

- 目的: ロールバック機能が無いため、**Git側で戻して再デプロイする手順が成立するか**を確かめる。
- 使った org: `dcng-prod`（Hub）/ `dcng-dev1` / `dcng-stg`
- **予測を先に書いてから実行した。**
- 材料: `CliNote__c`（5-5 の CLI 検証で作ったもので用途が終わっていた）

---

## 1. 判定の焦点

Metadata API のデプロイは `destructiveChanges` が無ければ org のコンポーネントを消さない
（→ 本番との差分に関する検証）。
**DevOps Center がこれを生成するかが分からないと、「Git で戻す」が成立するかも分からない。**

`WorkItemComponent.ChangeType` の picklist には `DELETE` があった
（`NEW` / `CHANGE` / `DELETE` / `MANUAL` / `DATA`）。
ただし**値があることは機能があることを意味しない**ので実機で確かめた。

## 2. dev1 から削除する（3.98秒）

```
sf project delete source -o dcng-dev1 -m "CustomField:HybridTest__c.CliNote__c" --no-prompt
→ Elapsed Time: 3.98s
  Deleted Source: HybridTest__c.CliNote__c / CustomField
```

**org とローカルの両方から消える。** retrieve で確認しても0件だった。
git は削除として認識した。

```
 D force-app/main/default/objects/HybridTest__c/fields/CliNote__c.field-meta.xml
```

## 3. 変更リストに `DELETE` として出る

コミットして push し、画面を開いて `INSPECT` を走らせた。

| MemberName | MemberType | ChangeType |
|---|---|---|
| `HybridTest__c.CliNote__c` | `CustomField` | **`DELETE`** |
| `HybridTest__c.ExtMergeNote__c` | `CustomField` | `CHANGE` |
| `HybridTest__c.VehicleNote__c` | `CustomField` | `CHANGE` |

`CHANGE` の2件は、この作業で `retrieve` したときに `<trackHistory>` が消えた分である
（08-28 に決めた「deploy と retrieve を対にする」の副作用が、
**まだ git に入っていなかった分としてここで現れた**）。

## 4. staging と prod の両方から消えた

| 段階 | `DevopsRequestInfo` | org |
|---|---|---|
| 第1ステージへ昇格 | `PROMOTE` / `SUCCESS` | **staging から消えた**（retrieve で0件） |
| 本番へステージ昇格 | `PROMOTE` / `SUCCESS` | **prod から消えた**（13個 → 12個） |

作業項目は `CLOSED` になった。

prod の `HybridTest__c` の項目（12個）。

```
AdminNote__c ApiNote__c BlockANote__c BlockBNote__c CiFailNote__c CiNote__c
EngineerNote__c ExtMergeNote__c FullNote__c ReviewerNote__c TransferNote__c VehicleNote__c
```

`CliNote__c` が消えている。

## 5. 予測との照合

| 何 | 予測 | 結果 |
|---|---|---|
| dev1 で削除すると追跡されるか | される | **当たり** |
| 変更リストの `ChangeType` | `DELETE` | **当たり** |
| staging / prod から消えるか | 消える | **当たり** |
| 手数 | 通常の作業項目と同じ | **当たり** |

## 6. 分かったこと

- **削除は昇格で運ばれる。** `ChangeType: DELETE` として変更リストに出て、
  staging と prod の org から実際に消える
- **「Git 側で戻して再デプロイ」は成立する。** ロールバック機能が無くても、
  通常の作業項目と同じ手順で戻せる
- 手数は通常の変更と同じ。削除のための特別な操作は要らない
- `sf project delete source` は org とローカルの両方から消す。**3.98秒**

## 7. 未確認として残したもの

- **`git rm` だけして org に残した場合**どうなるか。
  DevOps Center は org の変更を追跡する仕組みなので、
  git だけから消しても変更リストに乗らない可能性がある。実行計画では両方見るつもりだったが、
  `sf project delete source` で通ったのでそこで止めた
- **オブジェクトごと削除**した場合（申し送りの「後片付け」の論点）。
  項目1つで成立したので同じだと考えられるが、確かめていない
- 削除と追加が同じ作業項目に混ざったときの順序
