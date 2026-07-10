# clawd-on-desk — Agent Guide

Electron製のデスクトップペット。各AI coding agentのhook・plugin・logを受け、状態表示、権限バブル、セッションHUD、端末フォーカスを提供する。詳細仕様は `docs/` を正本とし、このファイルには変更時の入口と不変条件だけを置く。

## 最初に読む

- 全体像・対応agent・導入: `README.md`
- agent統合・permission・多会話: `docs/project/agent-runtime-architecture.md`
- 状態・テーマ・Settings UI: `docs/project/theme-state-ui.md`
- 状態マッピング: `docs/guides/state-mapping.md`
- agent導入・Remote SSH: `docs/guides/setup-guide.md`
- 発版: `docs/project/release-process.md`

## 基本コマンド

```bash
npm install
npm start
npm test
npm run build
npm run build:mac
npm run build:win:all
npm run build:linux
npm run create-theme
```

agent別のinstall／uninstallやremote deployは `package.json` scripts と対応する `docs/guides/` を確認して実行する。

## アーキテクチャ

- 状態経路: hook / monitor → `src/server.js` → `src/state.js` → IPC → renderer
- 設定経路: `src/prefs.js` → `src/settings-controller.js` → `src/settings-store.js`。controllerだけが書き込む
- agent定義: `agents/registry.js`、有効化と機能境界: `src/agent-gate.js`
- agent installer／hook: `hooks/*-install.js`、`hooks/*-hook.js`
- Remote SSH: `src/remote-ssh-*.js` と `scripts/remote-deploy.sh`
- UIは表示窓と入力窓の二窓構成。DashboardとSession HUDはsession snapshotを共有する

## 不変条件

- ユーザー既存のhook設定は上書きせず、Clawd管理部分だけを増分更新する
- agentの無効化はhook/pluginを削除しない。同期・event・permission入口を止め、明示Uninstallだけが削除する
- permission hookのstdoutは各agent仕様に厳密に合わせ、未対応agentの権限をClawdが代理決定しない
- hook helper追加時は `scripts/remote-deploy.sh` の `FILES` と `src/remote-ssh-deploy.js` の `HOOK_FILES` の両方へ登録する
- 安定したterminal PIDは `getStablePid()` を使い、`process.ppid` で代用しない
- HTTP portは `127.0.0.1:23333-23337`、実行時情報は `~/.clawd/runtime.json`
- Remote SSHのNode probeは `scripts/remote-deploy.sh` と `src/remote-ssh-node.js` で挙動を揃える
- Windows installerはx64／ARM64を明示し、universal installerへ戻さない
- theme assetは `assets/source/` の保全対象を確認し、元データを直接破壊しない
- Settingsのstoreを唯一の真実、controllerを唯一のwriterとして扱う

agent固有のhook種類、permission対応範囲、session id規則は実装・テスト・対応ガイドを読んでから変更する。

## テスト

- 基本は `npm test`
- hook payload、とくにpermission系はunit testに加えて対象CLIで実測する。自作`curl`だけで完了にしない
- 透明窓、drag、tray、terminal focus、macOS固有挙動は手動QAが必要。実施できない場合は未検証と残余リスクを明記する

## 回帰しやすい箇所

- `hitWin.focusable = true` はWindows drag修正に必要
- `miniTransitioning` 中のwindow位置更新はguardする
- DNDはpermissionを承認・拒否せず、各agentのnative確認へ戻す
- `petHidden` はDNDではなく、新しいpermission bubbleを抑止しない
- Session HUDからidle完了sessionを除外しない
- update bubbleはHUDとpermission stackの両方を避ける
- `contextMenuOwner` の `parent: win` と `closable:false` を維持する
- Claude settings watcherはfileではなくdirectoryを監視し、管理フラグ・installed・enabledの三条件でguardする
- QoderのWindows commandはbash/cmd互換のportable形式を維持し、旧PowerShell encoded形式へ戻さない
- Codex official hookがprimary、JSONL monitorはfallbackとして残す
- SVG URLの `?_t=` cache bustを削除しない

Language submenu下端のWindows表示問題は既知のDWM制約。透明窓方式やalways-on-topの再設計を安易に再開しない。
