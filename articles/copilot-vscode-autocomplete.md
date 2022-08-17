---
title: "VSCodeでcopilot+neovimを使っていると，IntelliSenseがうまく使えない問題の回避策"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vscode","copilot","neovim","vim","IntelliSense"]
published: false
---

# 発生した問題
VSCodeにてcopilotを使用していると，意図していないのに補完が出てきてしまうことがよくありました。これだけならまだ許せたのですが，IntelliSense（エディタとかその他の拡張機能による補完，定義あってる？）が出なくなってしまっていて，class名とかの補完が効かなくなり結構困っていました。
![oi](/images/copilot-vscode-autocomplete/omg.png)

Copilotを無効化せずにCopilot自身のの補完を無視するには，Escを押すしかないみたいなのですが，Neovimを併用している都合上，Escを押すと問答無用でCopilot終了どころかinsertモードまで終了してしまい，エディタ補完の出番なくノーマルモードに戻り困っていました。
![ノーマルモードに戻ってしまった](/images/copilot-vscode-autocomplete/omg-vim.png)
（↑ノーマルモードに戻ってしまう……）

# その時の環境
- VSCode 1.70.1
    - Github Copilot 1.41.6508
    - VSCode Neovim 0.0.89
    - Flutter拡張などなど


# 解決策
copilotのEscをそれ以外のキーに変えられれば本来はよかったのですが，ダメそうだったので，Neovim拡張（vimでも同じだと思う）からEscを無効化することにしました。
cmd+k→cmd+sでキーボードショートカット一覧を表示し，「neovim esc」で検索しました。その上で，KeybindingがEscになっている箇所を，ctrl+[に変更することで，EscでCopilotの打消のみを行うようにしたことで，無事にIntelliSenseが動くようになりました。
![fix](/images/copilot-vscode-autocomplete/fix.png)
（↑SourceがUserとなっている部分が，いじった箇所）

![わーい](/images/copilot-vscode-autocomplete/lgtm.png)

# 一言
僕はVimのEscをctrl+[で普段から行っていたためよかったのですが，そうではないときついかなと思います…。根本的な解決策はあるのでしょうか…？
