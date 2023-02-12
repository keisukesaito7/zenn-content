---
title: "Warp の Hotkey が効かない / 特定のデスクトップでしか開かない に対応"
emoji: "⌨️"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["Warp", "Raycast"]
published: false
---

## Warp を使いたいのに困ったことがあった

最近流行りの Warp。

https://www.warp.dev/

とても体験が良さそうなので [iTerm2](https://iterm2.com/) から乗り換えたのですが、困りごとがありました。

1. Hotkey で Warp が開かない！
2. Warp を開いたデスクトップでしか Warp が開かない！

おそらく「私の環境が悪いのだろう」「何か設定を見落としているだろう」という前提で、もし似た状況の方がいたときのためにメモしておきます。

## 動作環境

| アプリケーション・デバイス | version |
| ---- | ---- |
| MacBook Pro | macOS Monterey v12.6.1 (Intel Core i7) |
| Warp | v0.2023.02.07.08.03.stable_01 |
| Raycast | v1.47.3 |

## 困りごと 1. Hotkey で Warp が開かない！

Warp には、Hotkey が設定できます。

私の場合、iTerm2 を`⌥` `SPACE`で開くようにしていたので、これを Warp でも行いたいと思いました。

公式に手順が載っていたので、そのとおりにやってみました。

### やってみたこと

https://docs.warp.dev/features/windows/hotkey-window

> 1. Open Settings > Features and tick the Hotkey Window and enable the feature.
> 「設定」→「機能」を開き、「ホットキーウィンドウ」にチェックを入れ、機能を有効にする。
> 2. There you can configure the keyboard shortcut and the windows position, screen, and relative size.
> そこで、キーボードショートカットやウィンドウの位置、画面、相対的な大きさを設定することができます。
> 3. Toggle off "Autohide Hotkey Window on loss of focus", the Hotkey Window will stay on top when triggered regardless of mouse or keyboard focus.
> "Autohide Hotkey Window on loss of focus"をオフにすると、マウスやキーボードのフォーカスに関係なく、ホットキーウィンドウが一番上に表示されるようになります。

![warp-config-hotkey-window](/images/warp-hotkey-window/warp-config-hotkey-window.png)

こんな感じで Hotkey を ON にし、`⌥` `SPACE` を設定！

> Note: If the window does not open after pressing the registered hotkey, check under System Preferences > Security & Privacy > Accessibility and tick the checkbox to grant Warp access. Also,ESC, BACKTICK, TAB , SHIFT, CAPS are not supported keyboard shortcuts.
> 注意：登録したホットキーを押してもウィンドウが開かない場合は、システム環境設定＞セキュリティとプライバシー＞アクセシビリティを確認し、ワープへのアクセスを許可するチェックボックスにチェックを入れてください。また、ESC、BACKTICK、TAB、SHIFT、CAPSはサポートされていないキーボードショートカットです。

という注意もあるので、ちゃんとアクセス許可も行いました！

![access-allowed](/images/warp-hotkey-window/access-allowed.png)

設定が反映されてないかもなので、一度 Warp を終了して、もう一度立ち上げました。

いざ、`⌥` `SPACE`！

...

が、ダメ！！
MacBook を再起動してもダメ！！

一応、他の方も Hotkey の設定をされていたので参考にしてみましたが、それでもダメ。。なぜだ。。

### 解決

たぶん私の事象とは違う理由ですが、Warp を Raycast のホットキーで起動している方がいらっしゃったので、参考にさせてもらいました。

https://zenn.dev/toono_f/articles/03cc961bfd64c1#%E3%83%9B%E3%83%83%E3%83%88%E3%82%AD%E3%83%BC%E3%81%AE%E8%A8%AD%E5%AE%9A

Raycast 側で `⌥` `SPACE`を設定。

![raycast-config-warp-hotkey](/images/warp-hotkey-window/raycast-config-warp-hotkey.png)

実行！

1. Warp が起動していないことを確認
2. `⌥` `SPACE`
3. 開いた！！

![warp-open-1](/images/warp-hotkey-window/warp-open-1.gif)

## 困りごと 2. Warp を開いたデスクトップでしか Warp が開けない！

`⌥` `SPACE`で開けたのはよかったのですが、今度は「Warp を開いたデスクトップでしか Warp が開けない」という事象が起きました。

1. Warp が起動していないことを確認
2. `⌥` `SPACE`で Warp を起動
3. 別のデスクトップに移る
4. `⌥` `SPACE`を実行（期待: 同じデスクトップで Warp が開く）
5. Warp を開いたデスクトップに戻る
6. `⌥` `SPACE`で Warp を Hide する
7. 別のデスクトップに移る
8. `⌥` `SPACE`を実行（期待: 同じデスクトップで Warp が開く）
9. Warp を開いたデスクトップで Warp が開く

![warp-open-same-window](/images/warp-hotkey-window/warp-open-same-window.gif)

### 解決

これは、Warp の割り当て先のデスクトップを「すべてのデスクトップ」にしたら解決できました！

![warp-assing-all-desktop](/images/warp-hotkey-window/warp-assing-all-desktop.png)

- どのデスクトップでも Warp が開いた状態になる
- `⌥` `SPACE`で一番上に持ってこれる

![warp-open-all-desktop](/images/warp-hotkey-window/warp-open-all-desktop.gif)

## おわりに

Raycast と組み合わせることで、Warp の Hotkey が想定通りに動くようになりました。
同じ悩みを持っている方の一助となれれば幸いです✨
