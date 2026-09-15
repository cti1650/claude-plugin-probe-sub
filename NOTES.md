# 検証ログ

プラグイン配布そのものの挙動（marketplace取り込み、symlinkの保持、version更新の伝播、
Codex CLIとの互換性など）は、対向リポジトリ側に記録してある。

→ [cti1650/claude-plugin-probe の NOTES.md](https://github.com/cti1650/claude-plugin-probe/blob/main/NOTES.md)

このリポジトリは**差し替えが効くか**の確認に使うもので、単体の挙動は probe と同じである。
2つを並べたときに分かったことをここに記録する。

環境: Claude Agent SDK (TypeScript) 0.3.268 / Claude Code 2.1.268 / node:22-slim コンテナ

## 差し替えの検証

2つのプラグインをイメージにビルド時焼き込みし、`query()` の `plugins` オプションで
呼び出しごとに選択できるかを確認した。

| # | 観点 | 結果 |
|---|---|---|
| 1 | 2つのmarketplaceを同時に登録できる | ✅ marketplace 名が別なら衝突しない |
| 2 | 指定したプラグインだけがロードされる | ✅ ただし `settingSources` の明示が必要（下記） |
| 3 | 呼び出しごとに差し替えが効く | ✅ 同一プロセスで戻り値が `OK` / `OK-SUB` と切り替わる |
| 4 | 両方を同時にロードできる | ✅ 両方のスキルが呼ばれ `OK\nOK-SUB` が返る |
| 5 | 選択されていないスキルはモデルから見えない | ✅ ツールを呼ばず「無い」と答えた（捏造しない） |

## 1. 選択の仕組み

`plugins` オプションは**ローカルパス**を受け取る。

```ts
plugins: [{ type: 'local', path: '/path/to/plugin' }]
```

ビルド時に候補を全部焼き込んでおき、実行時にパスで選ぶ形にできる。
実行時の clone が発生しないので、コールドスタートに影響しない。

## 2. `settingSources` を省略すると選択が効かない

**これが一番の落とし穴。** 省略するとユーザー設定（`~/.claude`）を読み、
`plugins` で指定していないプラグインまで載る。

| 設定 | ロードされたスキル |
|---|---|
| `settingSources` 未指定 + `plugins:[probe-sub]` | `probe-sub`, **`probe`** ← 指定していない方が載る |
| `settingSources` 未指定 + `plugins` なし | `probe`, `probe-sub` |
| `settingSources: ['user','project']` | `probe`, `probe-sub` |
| `settingSources: ['project']` | (なし) |
| `settingSources: ['project']` + `plugins:[probe-sub]` | `probe-sub` のみ |
| `settingSources: []` + `plugins:[probe-sub]` | `probe-sub` のみ |
| `settingSources: []` + `plugins` なし | (なし) |

再現性は2回の実行で確認した。

**分離したいなら `settingSources: []` を明示する。** 省略は安全側に倒れない。
エラーにもならないので、書き忘れると余分なプラグインが黙って載り続ける。

プロジェクトの `.claude/skills` も読ませたい場合は `['project']` にする。
どちらにせよ `'user'` を入れると焼き込み済みのプラグインが全部載る。

## 3. 選択されていないスキルは本当に見えない

`probe-sub` だけを載せた状態で `probe` の使用を依頼した結果:

```
ロード : [plugin-probe-sub:probe-sub]
ツール : (なし)
戻り値 : "無い"
```

一覧から消えているだけでなく、呼び出せる状態にもなっていない。
無い機能を捏造して適当な結果を返すことも無かった。

## 4. 焼き込み先のパスにバージョンが入る

```
~/.claude/plugins/cache/plugin-probe-marketplace/plugin-probe/0.3.0/
~/.claude/plugins/cache/plugin-probe-sub-marketplace/plugin-probe-sub/0.1.0/
```

`plugins` に渡すパスを固定値で持てない。プラグインを更新するとパスが変わるため、
ビルド時に安定パス（`/opt/plugins/<name>` 等）へコピーするか、
実行時に `installed_plugins.json` から解決する必要がある。

## 再現手順

`plugins.json` 相当で両方を焼き込んだイメージを作り、SDK から呼び分ける。

```js
import { query } from '@anthropic-ai/claude-agent-sdk';

const stream = query({
  prompt: 'probe-sub スキルを使って、その結果だけを返してください。',
  options: {
    settingSources: [],                                   // ← 省略しないこと
    plugins: [{ type: 'local', path: '/path/to/plugin-probe-sub' }],
    permissionMode: 'bypassPermissions',
  },
});
```

`system/init` メッセージの `slash_commands` を読めば、モデルを呼ばずに
何がロードされたかだけを確認できる（課金なし・APIキー不要）。

## 未検証

- Codex CLI 側に同等の選択機構があるか
- 同名スキルを持つプラグインを両方ロードしたときの優先順位
- プラグイン数を増やしたときの常時ロードのトークン量と起動時間
