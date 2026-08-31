# 2026-08-29 本番の写しを git merge して衝突を解決する（R-2b）

- 目的: **本番の写しをGitブランチにすると、レイアウトの衝突をGitの道具で解決できるか**を確かめる。
- 使った org: `dcng-prod`（Hub）/ `dcng-dev1`
- **予測を先に書いてから実行した。**
- 前提: [R-2](2026-08-28-layout-recovery-r2.md) で、復旧の昇格が本番の配置を消すと分かった

---

## 1. なぜこの経路を試したか

R-2 で「昇格するファイルに本番の配置も入っていれば失われないはず」と書いたが、
**どうやって入れるか**が残っていた。

| やり方 | 難点 |
|---|---|
| 本番から retrieve した XML とリポジトリの XML を手で突き合わせる | DevOps Center の変更リストは XML の diff を見せず、git の道具も使えない |
| **本番の写しを別ブランチに置き、作業項目ブランチへ `git merge` する** | これを試した |

## 2. 手順と所要時間

| 時刻 | やったこと | 所要 |
|---|---|---|
| 00:21:11 | prod のレイアウトに `FullDeployVehicle__c` を直接配置（材料づくり） | |
| 00:21:41 | **`prod-snapshot` ブランチを作って push**（本番の写し） | **3秒** |
| 00:22:00 | 作業項目 `WI-000025` を作る | 12秒 |
| 00:22:43 | 開発側で `Account.MergeTestNote__c` を作り、**本番と同じ位置**に配置 | |
| 00:24:19 | dev1 へ deploy → retrieve → commit → push | 1分36秒 |
| 00:24:32 | **`git merge origin/prod-snapshot`** | |
| 00:24:57 | コンフリクトを解決して push | **25秒** |
| 00:25:12 | PR #33 を作る | |
| 00:25:59 | 画面を開いて変更リストを作り、staging へ昇格 | |
| 00:27:16 | staging 完了 | |
| 00:29:26 | 本番へ昇格 完了 | |

**`prod-snapshot` を作る工程は3秒。** retrieve 済みのファイルをコピーしてコミットするだけである。

## 3. コンフリクトは起きた（予測は外れた）

```
git merge origin/prod-snapshot
→ Auto-merging force-app/main/default/layouts/Account-取引先 レイアウト.layout-meta.xml
  CONFLICT (content): Merge conflict in force-app/main/default/layouts/Account-取引先 レイアウト.layout-meta.xml
  Automatic merge failed; fix conflicts and then commit the result.
```

**予測では「追加位置が離れていれば自動マージされる」としたが、
今回は本番と開発側が同じ位置（`ProdOnlyNote__c` の直後）に追加したのでコンフリクトになった。**

### コンフリクトの形が扱いやすい

```xml
            <layoutItems>
                <behavior>Edit</behavior>
<<<<<<< HEAD
                <field>MergeTestNote__c</field>
=======
                <field>FullDeployVehicle__c</field>
>>>>>>> origin/prod-snapshot
            </layoutItems>
```

**`<layoutItems>` と `<behavior>` は共通部分として扱われ、衝突したのは `<field>` の1行だけ。**
どちらの項目が衝突しているかが一目で分かる。

**ただし `--ours` / `--theirs` の2択では片方が消える。**
両方を残すには `<layoutItems>` ブロックを補って書く必要がある。

```xml
            <layoutItems>
                <behavior>Edit</behavior>
                <field>MergeTestNote__c</field>
            </layoutItems>
            <layoutItems>
                <behavior>Edit</behavior>
                <field>FullDeployVehicle__c</field>
            </layoutItems>
```

**解決は25秒。** XML の構造を知っていれば機械的な作業である。

## 4. 判定: 両方残った

| 配置 | 出どころ | 昇格後の prod |
|---|---|---|
| **`FullDeployVehicle__c`** | **本番の直接変更** | **残った** |
| `MergeTestNote__c` | 開発側の追加 | 残った |
| `DevSideNote__c` / `ProdOnlyNote__c` / `VerificationNote__c` | 既存 | 残った |

**手順として成立する。** R-2 で失われた配置が、この経路では保たれた。

変更リストにはレイアウトが `CHANGE` として2行出た（同じレイアウトが2件）。
理由は追えていない。

## 5. 予測との照合

| 何 | 予測 | 結果 |
|---|---|---|
| `git merge` の結果 | 自動マージされる | **外れ。コンフリクトが起きた**（同じ位置に追加したため） |
| 追加位置が近いとき | コンフリクトが起きる | **当たり** |
| 昇格後の prod | 両方の配置が残る | **当たり** |

## 6. 分かったこと

- **本番の写しを git のブランチにすれば、レイアウトの衝突を git の道具で扱える。**
  `prod-snapshot` を作る工程は3秒
- **コンフリクトは `<field>` の1行に絞られる。** どの項目が衝突しているか一目で分かる
- **`--ours` / `--theirs` では解決できない。** 両方を残すには
  `<layoutItems>` ブロックを補う必要がある。XML の構造を知っている人向けの作業
- 解決は25秒。**GitHub の Web エディタでも同じことができる**（未確認）
- 昇格すると**本番の配置も開発側の配置も両方残った**

### R-2 との比較

| | R-2（本番の写しを使わない） | R-2b（本番の写しをマージ） |
|---|---|---|
| 本番で後から加えた配置 | **失われた** | **残った** |
| 増える工程 | なし | `prod-snapshot` を作る（3秒）+ マージと解決（25秒） |
| 必要な知識 | なし | レイアウトの XML 構造 |

**増えるのは30秒程度で、失う配置が無くなる。**

## 7. 未確認として残したもの

- **GitHub の Web エディタでこのコンフリクトを解決できるか。** 手元の git で解決した
- 追加位置が離れている場合に自動マージされるか。今回は同じ位置に作った
- `prod-snapshot` ブランチを継続的に更新する運用。**R-6 の定期検知と組み合わせる形**になるが未設計
- 変更リストにレイアウトが2行出た理由
- 他のメタデータ（`Profile` / `FlexiPage` など）でも同じ形のコンフリクトになるか
