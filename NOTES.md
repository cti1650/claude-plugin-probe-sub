# 検証ログ

環境: macOS / Claude Code / Codex CLI 0.146.0

## 結果

| # | 観点 | 結果 |
|---|---|---|
| 1 | `claude plugin validate` | ✅ 通る（marketplace / plugin / skills） |
| 2 | プライベートリポジトリをcloneできるか | ✅ `marketplace add` が `git@github.com:` 形式でSSH cloneする |
| 3 | Claude Code: 配布経由でスキルが読める | ✅ 無関係なディレクトリで `/plugin-probe:probe` → `OK` |
| 4 | Claude Code: symlinkが配信キャッシュで保たれる | ✅ `.claude/skills/probe -> ../../skills/probe` のまま残る |
| 5 | Claude Code: version更新が利用者に届く | ✅ 0.1.0 → 0.2.0 で返答が `OK` → `OK2` に変わった |
| 6 | Codex: `.claude-plugin/` だけで取り込める | ✅ marketplace add / plugin add とも通り、versionも読まれる |
| 7 | Codex: 配布経由でスキルが読める | ✅ `skills/probe/SKILL.md` を直接読んで `OK2` を返した |
| 8 | Codex: symlinkが配信キャッシュで保たれる | ❌ **保たれない**（下記） |
| 9 | Codex: `.codex-plugin/plugin.json` の自動生成 | ❌ **生成されなかった**（下記） |

## 8. Codexはsymlinkを落とす

marketplaceのsnapshot（git clone）にはsymlinkが残るが、`plugin add` でインストールキャッシュへ
展開する段階で落ちる。

```
~/.codex/.tmp/marketplaces/<mp>/.claude/skills/probe -> ../../skills/probe   # 残る
~/.codex/plugins/cache/<mp>/<plugin>/<ver>/.claude/skills/                   # 空になる
```

このリポジトリは実体が `skills/` にあり `.claude/skills` がsymlink、という向きなので影響を受けない。
逆向き（実体が `.claude/skills/`、`skills/` がsymlink）にするとCodexでスキルが消える。
両CLIで配布するなら **実体は `skills/` 側に置く** こと。`.claude/skills` のsymlinkは
Codex側では単に無視される（無害）。

## 9. `.codex-plugin/plugin.json` は自動生成されなかった

`.claude-plugin/marketplace.json` 経由で取り込んだプラグインには生成されない。
一方、`.agents/plugins/marketplace.json` 形式のmarketplaceから入れたプラグインには
`"skills": "./skills/"` 込みで生成されていた。生成条件はmarketplaceの形式に依存する。

ただし生成されなくても `.claude-plugin/plugin.json` の version は読まれ、スキルも動く。
Codex用のマニフェストを別途保守する必要はないが、
**「Codexが自動生成してくれるから不要」ではない**点に注意。

## 5. 配信キャッシュはバージョンごとに別ディレクトリ

```
~/.claude/plugins/cache/<mp>/<plugin>/0.1.0/
~/.claude/plugins/cache/<mp>/<plugin>/0.2.0/   # updateでこちらが使われる
```

更新後も旧バージョンのディレクトリは残るが参照されない。
プラグインディレクトリ配下へ出力を書くと、更新のたびに見えなくなる。
出力先は実行時のカレントディレクトリ基準にすべき。Codex側も
`~/.codex/plugins/cache/<mp>/<plugin>/<ver>/` と同じ構造。

## 未検証

- `codex plugin marketplace upgrade` だけで新バージョンへ切り替わるか
- Organization配下のプライベートリポジトリからのインストール

## 再現手順

```sh
claude plugin marketplace add cti1650/claude-plugin-probe
claude plugin install plugin-probe@plugin-probe-marketplace
cd /tmp && claude -p "/plugin-probe:probe"      # OK が返る

codex plugin marketplace add cti1650/claude-plugin-probe
codex plugin add plugin-probe@plugin-probe-marketplace
cd /tmp && codex exec --skip-git-repo-check "probeスキルを使って"   # OK が返る
```

versionを上げて `skills/probe/SKILL.md` の返答文字列を変えれば、更新の伝播を再確認できる。
