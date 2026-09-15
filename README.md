# claude-plugin-probe

プラグインとして配布したときに **スキルがちゃんと読み込まれるか** だけを確かめる、検証用の最小リポジトリ。
`probe` スキルを呼ぶと `OK` と返るだけで、実処理は持たない。

Claude Code と Codex CLI の両方で、同じ `.claude-plugin/` のマニフェストから配布できるかを試す。

```
.claude-plugin/
  marketplace.json    このリポジトリ自身を1プラグインとして公開する
  plugin.json         プラグインのメタデータ（version はここが正本）
skills/probe/
  SKILL.md            スキルの実体
.claude/skills/probe  -> ../../skills/probe （相対symlink）
```

## 確認できること

| 観点 | 確認方法 |
|---|---|
| marketplaceとして取り込めるか | `marketplace add` が通る |
| `.claude-plugin/` の2ファイルだけで足りるか | Codex側にマニフェストを置かずに動くか |
| symlink経由でスキルが読めるか | リポジトリを直接開いた状態で `probe` が `OK` を返す |
| 配布経由でスキルが読めるか | install後、無関係なディレクトリで `probe` が `OK` を返す |
| version更新が利用者に届くか | `version` を上げて返答文字列を `OK2` に変え、update後に `OK2` になるか |

## 使い方

リポジトリを直接開く場合はそのまま `probe` スキルを呼ぶ。

プラグインとして入れる場合:

```sh
# Claude Code
claude plugin marketplace add cti1650/claude-plugin-probe
claude plugin install plugin-probe@plugin-probe-marketplace

# Codex CLI
codex plugin marketplace add cti1650/claude-plugin-probe
codex plugin add plugin-probe@plugin-probe-marketplace
```

外すとき:

```sh
claude plugin uninstall plugin-probe@plugin-probe-marketplace
codex plugin remove plugin-probe@plugin-probe-marketplace
```

更新を配る側は、変更のたびに `.claude-plugin/plugin.json` の `version` を上げること。
上げないと利用者に更新が届かない。

検証結果は [NOTES.md](NOTES.md) に置いてある。
