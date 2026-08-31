# 次世代 DevOps Center 検証ログ

Salesforce の次世代 DevOps Center を、GitHub とSandbox組織を接続した検証環境で確認した記録です。

記事で結論の根拠として参照したログだけを公開しています。
各ログは検証時点の製品挙動であり、すべての組織・設定・将来のリリースで同じ結果になることを保証するものではありません。

## 読み方

ファイル名の日付は検証日です。
ログには、目的、事前の予測、実施した操作、結果、未確認事項を残しています。
同じ検証を再現する場合は、組織の設定、権限、パイプライン構成、GitHub のルールを確認してください。

## 公開した検証ログ

| 分類 | 検証 | ログ |
| --- | --- | --- |
| 変更の検知 | 項目レベルセキュリティを含む変更追跡 | [2026-08-26-fls-tracking.md](verification/2026-08-26-fls-tracking.md) |
| GitHub連携 | 外部でマージしたPull Requestの検出 | [2026-08-27-external-merge-3-1.md](verification/2026-08-27-external-merge-3-1.md) |
| GitHub連携 | GitHub認証が利用者ごとか | [2026-08-30-github-auth-per-user.md](verification/2026-08-30-github-auth-per-user.md) |
| GitHub連携 | Pull Requestのタイトル・本文を変更した場合 | [2026-08-31-pr-title-rewrite.md](verification/2026-08-31-pr-title-rewrite.md) |
| 開発フロー | MCPから作業項目を作成し昇格する | [2026-08-27-mcp-workitem.md](verification/2026-08-27-mcp-workitem.md) |
| 開発フロー | 作業項目を作らずにステージブランチへマージする | [2026-08-29-rogue-branch.md](verification/2026-08-29-rogue-branch.md) |
| GitHubルール | Required status checkで昇格を止める | [2026-08-28-ci-branch-protection.md](verification/2026-08-28-ci-branch-protection.md) |
| GitHubルール | Pull Request必須のRulesetと昇格 | [2026-08-30-pr-required-ruleset.md](verification/2026-08-30-pr-required-ruleset.md) |
| 本番との差分 | 本番の変更をGitへ取り込む | [2026-08-27-recovery-r1.md](verification/2026-08-27-recovery-r1.md) |
| 本番との差分 | Run Back Syncが編集中の変更へ与える影響 | [2026-08-27-run-back-sync-3-7.md](verification/2026-08-27-run-back-sync-3-7.md) |
| 本番との差分 | レイアウト配置の復旧 | [2026-08-28-layout-recovery-r2.md](verification/2026-08-28-layout-recovery-r2.md) |
| 本番との差分 | 本番の写しをマージしてから昇格する | [2026-08-29-layout-merge-r2b.md](verification/2026-08-29-layout-merge-r2b.md) |
| 障害時の対応 | 意図的なデプロイ失敗の表示と導線 | [2026-08-28-deploy-failure-4-1.md](verification/2026-08-28-deploy-failure-4-1.md) |
| 障害時の対応 | MCPによるデプロイ失敗の調査 | [2026-08-28-resolve-failure-9-3.md](verification/2026-08-28-resolve-failure-9-3.md) |
| 障害時の対応 | 昇格した項目を削除する | [2026-08-28-delete-rollback-4-3.md](verification/2026-08-28-delete-rollback-4-3.md) |
| 障害時の対応 | Sandbox組織のリフレッシュと環境の置き換え | [2026-08-29-sandbox-refresh-r3.md](verification/2026-08-29-sandbox-refresh-r3.md) |
| 設定変更 | GitHubリポジトリを組織に移管する | [2026-08-28-repo-transfer.md](verification/2026-08-28-repo-transfer.md) |
| 自動化 | CLIからの昇格 | [2026-08-28-cli-promotion-5-5.md](verification/2026-08-28-cli-promotion-5-5.md) |

## 公開用の編集

検証の結論に不要な組織ID、利用者のメールアドレス、内部の計画資料へのリンクは除きました。

画面キャプチャは検証の文脈を残すため、元の記録と同じ位置に掲載しています。
利用者を識別できる情報は画像でもマスクしました。

記事で参照しない試行記録と個人間のやりとりは公開していません。
