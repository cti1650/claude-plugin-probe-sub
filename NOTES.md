# 検証ログ

プラグイン配布そのものの挙動（marketplace取り込み、symlinkの保持、version更新の伝播、
Codex CLIとの互換性など）は、対向リポジトリ側に記録してある。

→ [cti1650/claude-plugin-probe の NOTES.md](https://github.com/cti1650/claude-plugin-probe/blob/main/NOTES.md)

このリポジトリは**差し替えが効くか**の確認に使うもので、単体の挙動は probe と同じである。
2つを並べたときに分かったことがあれば、ここに追記する。

## 差し替えの検証

| # | 観点 | 結果 |
|---|---|---|
| 1 | 2つのmarketplaceを同時に登録できる | 未検証 |
| 2 | 指定したプラグインだけがロードされる | 未検証 |
| 3 | 呼び出しごとに差し替えが効く | 未検証 |
| 4 | 両方を同時にロードできる | 未検証 |
