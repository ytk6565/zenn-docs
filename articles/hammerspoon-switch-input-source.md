---
title: "Hammerspoon で英数・かなの切り替えを行う"
emoji: "📟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kitty", "hammerspoon"]
published: true
---

## はじめに

US配列のMacBook Proを利用しているのですがJIS配列と同じようにスペースキーの左右のキー（US配列では `command` キー）で入力ソース（英数・かな）の切り替えのために [Hammerspoon](https://www.hammerspoon.org/) を利用しているので設定と試したことをまとめる

## 前提

- macOS Monterey 12.4 を利用
- ターミナルのアプリケーションに [kitty](https://sw.kovidgoyal.net/kitty/) を利用
- Hammerspoon, kitty のインストール方法は記載しない

## 1. 左右の `command` キーを押したときに、「英数キー」「かなキー」を押すイベントを実行する

Hammerspoon の設定ファイル（`~/.hammerspooon/init.lua`)に下記を記述し、設定を読み込む（Hammerspoon で `Reload Config` を実行）
この設定で左右の `command` キーを押せば入力ソースの切り替えが可能になる

```lua:~/.hammerspooon/init.lua
local map = hs.keycodes.map
local keyDown = hs.eventtap.event.types.keyDown
local flagsChanged = hs.eventtap.event.types.flagsChanged
local keyStroke = hs.eventtap.keyStroke

local isCmdAsModifier = false

local function switchInputSourceEvent(event)
    local eventType = event:getType()
    local keyCode = event:getKeyCode()
    local flags = event:getFlags()
    local isCmd = flags['cmd']

    if eventType == keyDown then
        if isCmd then
            isCmdAsModifier = true
        end
    elseif eventType == flagsChanged then
        if not isCmd then
            if isCmdAsModifier == false then
                if keyCode == map['cmd'] then
                    keyStroke({}, 0x66, 0) -- 英数キー
                elseif keyCode == map['rightcmd'] then
                    keyStroke({}, 0x68, 0) -- かなキー
                end
            end
            isCmdAsModifier = false
        end
    end
end

eventTap = hs.eventtap.new({keyDown, flagsChanged}, switchInputSourceEvent)
eventTap:start()
```

## 2. 左右の `command` キーを押したときに、入力ソースの切り替えのショートカットを押すイベントを実行する

1 の方法で基本的には動作する
しかし kitty の利用中はこれが実行できなかった

おそらく原因は kitty が「英数キー」「かなキー」などの入力を監視・干渉することによってイベントが入力ソースの切り替えまで届いていないのだと理解している

そのため今回は入力ソースの切り替えのショートカットを押す方法で実現する

### 手順

#### 「入力ソースを切り替えるショートカット」を確認する

`システム環境設定 > キーボード > ショートカット > 入力ソース > 入力メニューの次のソースを選択` から確認する
デフォルトは `ctrl+alt+space` が設定されていた

[](/images/hammerspoon-switch-input-source/shortcut.png)

#### 「英数・かなの入力ソースのID」を確認する

入力ソースには ID があるので確認する

1. Hammerspoon の設定ファイルに下記を記述し、設定を読み込む

```lua:~/.hammerspooon/init.lua
local keyDown = hs.eventtap.event.types.keyDown
local flagsChanged = hs.eventtap.event.types.flagsChanged

eventTap = hs.eventtap.new({keyDown, flagsChanged}, function(event)
    local flags = event:getFlags()
    if flags['cmd'] then
        print(hs.keycodes.currentSourceID())
    end
end)
eventTap:start()
```

1. 入力ソースを「英数」に設定する
2. Hammerspoon のコンソールを開く（Hammerspoon で `Console...` を実行）
3. 左右どちらかの `command` キーを押す
4. コンソールに表示される「英数」の入力ソースIDを確認する
5. 入力ソースを「かな」に設定する（「入力ソースを切り替えるショートカット」を押すでも可）
   1. 自分の環境では `com.apple.keylayout.ABC`
6. 4 を実施する
7. コンソールに表示される「かな」の入力ソースIDを確認する
   1. 自分の環境では `com.apple.inputmethod.Kotoeri.RomajiTyping.Japanese`

#### 設定ファイルに反映する

入力された `command` キーで変更したい入力ソースと現在の入力ソースが異なるときだけ「入力ソースを切り替えるショートカット」を実行する

```lua:~/.hammerspooon/init.lua
local map = hs.keycodes.map
local keyDown = hs.eventtap.event.types.keyDown
local flagsChanged = hs.eventtap.event.types.flagsChanged

local SOURCE_ID_EN = "com.apple.keylayout.ABC" -- 「英数」の入力ソースID
local SOURCE_ID_JA = "com.apple.inputmethod.Kotoeri.RomajiTyping.Japanese" -- 「かな」の入力ソースID

-- 「入力ソースを切り替えるショートカット」を押す
local function switchInputSource()
    hs.eventtap.keyStroke({"ctrl", "alt"}, "space", 0)
end

local isCmdAsModifier = false

local function switchInputSourceEvent(event)
    local eventType = event:getType()
    local keyCode = event:getKeyCode()
    local flags = event:getFlags()
    local isCmd = flags['cmd']

    if eventType == keyDown then
        if isCmd then
            isCmdAsModifier = true
        end
    elseif eventType == flagsChanged then
        if not isCmd then
            if isCmdAsModifier == false then
                local currentSourceID = hs.keycodes.currentSourceID()

                -- 入力された command キーの入力ソースと現在の入力ソースが異なるときだけ実行
                if keyCode == map['cmd'] and currentSourceID ~= SOURCE_ID_EN then
                    switchInputSource()
                elseif keyCode == map['rightcmd'] and currentSourceID ~= SOURCE_ID_JA then
                    switchInputSource()
                end
            end
            isCmdAsModifier = false
        end
    end
end

eventTap = hs.eventtap.new({keyDown, flagsChanged}, switchInputSourceEvent)
eventTap:start()
```

## 試して上手く動作しなかったこと

### kitty で「英数キー」「かなキー」の監視・干渉をしないように設定

https://sw.kovidgoyal.net/kitty/conf/?highlight=discard_event#keyboard-shortcuts

```kitty.conf
map 0x66 discard_event
map 0x68 discard_event
```

### Hammerspoon の [`hs.keycodes.currentSourceID(sourceID)`](https://www.hammerspoon.org/docs/hs.keycodes.html#currentSourceID)

「かな」への変更はできるが、「英数」への変更でエラーが発生する

### Hammerspoon の [`hs.keycodes.setMethod(methodName)`](https://www.hammerspoon.org/docs/hs.keycodes.html#setMethod)

「かな」への変更はできるが、「英数」への変更でエラーが発生する