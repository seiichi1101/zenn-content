---
title: "Cosmic OSでCapsLockを変換キーにする方法"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["CosmicOS", "xremap", "CapsLock"]
published: true
---

最近 Cosmic OS の安定版が出たことに伴い、DE を Cosmic に変更しました。

以前の PopOS は GNOME ベースでしたが、新しい COSMIC は Rust 言語で書かれた独自の Wayland ネイティブな環境であるため、いろいろと設定方法が変わっています。

普段 CapsLock キーにかな変換を割り当てているのですが、以前使っていた xmodmap が使えなくなったので、別の方法を調べてみました。

最終的には、`xremap` を使って CapsLock キーを F13 キーにリマップし、fcitx のホットキー設定で F13 キーにかな変換を割り当てる方法で実現しました。

## xremap をインストール

`cargo` でインストールします。

`cargo install xremap --features cosmic`

cargo がインストールされていない場合は、`rustup` を使って Rust 環境をインストールすれば`cargo`も一緒にインストールされます。

https://rust-lang.org/tools/install/

## xremap の設定

設定ファイルを作成します。

`~/.config/xremap/config.yml`

```yaml
modmap:
  - name: caps to f13
    remap:
      CapsLock: F13
```

## fcitx の Hotkey の Trigger Input Method を F13 に設定

![alt text](/images/cosmicos-capslock/image.png)

`Tools` が F13 キーのことです。

## xremap を自動起動する設定

`~/.config/autostart/xremap.desktop` というファイルを作成し、下記の内容を記述します。
デスクトップの自動起動アプリケーションに登録されます。

```ini
[Desktop Entry]
Type=Application
Exec=/home/<username>/.cargo/bin/xremap /home/<username>/.config/xremap/config.yml
Hidden=false
NoDisplay=false
X-GNOME-Autostart-enabled=true
Name[en_US]=xremap
Name=xremap
Comment[en_US]=
Comment=
```

`<username>` の部分は自分のユーザー名に置き換えてください。
