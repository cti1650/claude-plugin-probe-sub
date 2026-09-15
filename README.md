# claude-plugin-probe-sub

[claude-plugin-probe](https://github.com/cti1650/claude-plugin-probe) の**対向**となる検証用プラグイン。
`probe-sub` スキルを呼ぶと `OK-SUB` と返るだけで、実処理は持たない。

単体では probe と同じことしか確認できない。**2つを並べて使うためのもの**である。

## 何のためにあるか

プラグインを1つだけ載せた場合、「ロードされたか」は分かるが「**切り替えが効いたか**」は分からない。
元のままでも差し替え後でも同じ `OK` が返るためである。

そこで名前と戻り値をすべてずらしてある。

| | probe | probe-sub |
|---|---|---|
| marketplace | `plugin-probe-marketplace` | `plugin-probe-sub-marketplace` |
| plugin | `plugin-probe` | `plugin-probe-sub` |
| skill | `probe` | `probe-sub` |
| 戻り値 | `OK` | `OK-SUB` |

これにより、**出力の文字列だけでどちらがロードされたかを判定できる**。

marketplace 名までずらしているのは、同名だと2つ目の `marketplace add` が衝突するため。

## 確認できること

| 観点 | 確認方法 |
|---|---|
| 複数marketplaceを同時に登録できるか | 両方 `marketplace add` が通る |
| 指定したプラグインだけがロードされるか | 片方だけ指定して、スキル一覧に他方が出ないこと |
| 呼び出しごとに差し替えが効くか | 指定を替えて、戻り値が `OK` / `OK-SUB` と切り替わること |
| 両方を同時にロードできるか | スキル一覧に `probe` と `probe-sub` が並ぶこと |

検証した結果は [NOTES.md](NOTES.md) にある。

## 使い方

```sh
# Claude Code
claude plugin marketplace add cti1650/claude-plugin-probe-sub
claude plugin install plugin-probe-sub@plugin-probe-sub-marketplace

# Codex CLI
codex plugin marketplace add cti1650/claude-plugin-probe-sub
codex plugin add plugin-probe-sub@plugin-probe-sub-marketplace
```

外すとき:

```sh
claude plugin uninstall plugin-probe-sub@plugin-probe-sub-marketplace
codex plugin remove plugin-probe-sub@plugin-probe-sub-marketplace
```

## 構成

```
.claude-plugin/
  marketplace.json    このリポジトリ自身を1プラグインとして公開する
  plugin.json         プラグインのメタデータ（version はここが正本）
skills/probe-sub/
  SKILL.md            スキルの実体
.claude/skills/probe-sub  -> ../../skills/probe-sub （相対symlink）
```

実体を `skills/` 側に置き `.claude/skills` を symlink にする向きは probe と同じ。
逆向きにすると Codex でスキルが消える（[probe側のNOTES.md](https://github.com/cti1650/claude-plugin-probe/blob/main/NOTES.md) 参照）。

更新を配る側は、変更のたびに `.claude-plugin/plugin.json` の `version` を上げること。
