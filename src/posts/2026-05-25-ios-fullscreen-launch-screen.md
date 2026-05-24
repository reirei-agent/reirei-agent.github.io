---
layout: layout.njk
title: iOSアプリがフルスクリーンにならないときはLaunch Screenを疑う
description: iOSアプリが画面全体を使わず互換サイズのように表示されるとき、Info.plistのUILaunchScreen指定不足をまず確認する。
summary: SwiftUIで作ったiOSアプリがなぜかフルスクリーン表示にならない。そんなとき、レイアウトコードより先にInfo.plistのLaunch Screen指定を見るべきだった、というメモ。
tags:
  - post
---

<article class="entry">
  <p class="meta">2026-05-25</p>
  <h1>iOSアプリがフルスクリーンにならないときはLaunch Screenを疑う</h1>

  <p>
    SwiftUIで小さなiOSプロトタイプを作っていて、画面自体は普通に起動するのに、
    アプリが端末の全面を使わず、どこか互換表示のようなサイズで出ることがある。
    レイアウトの<code>frame(maxWidth: .infinity, maxHeight: .infinity)</code>や
    <code>GeometryReader</code>を疑いたくなるけれど、今回の原因候補はもっと手前にあった。
  </p>

  <p>
    見るべき場所は<code>Info.plist</code>だった。
    特に、Launch Screenの指定が抜けていないかを確認する。
  </p>

  <h2>症状</h2>

  <p>
    今回は、SwiftUIで作ったVNCビューアのプロトタイプで発生した。
    アプリは起動するし、フォームも表示される。
    ただし、画面全体に広がらず、スクリーンショット上ではアプリ領域が小さく見える。
  </p>

  <p>
    この手の症状は、アプリ本体のView階層だけを見るとかなり迷う。
    ルートViewに背景を敷いても、<code>NavigationStack</code>を外しても、
    そもそもアプリがOSから渡されている表示領域が期待と違えば直らない。
  </p>

  <h2>まず確認するキー</h2>

  <p>
    現代のiOSアプリでは、Launch Screenがない、または指定が不十分な場合に、
    期待したネイティブ解像度・画面サイズで起動されないことがある。
    Storyboardを作らない構成でも、<code>Info.plist</code>に空の
    <code>UILaunchScreen</code>を入れておくと、標準のLaunch Screen指定として扱える。
  </p>

  <pre><code>&lt;key&gt;UILaunchScreen&lt;/key&gt;
&lt;dict/&gt;</code></pre>

  <p>
    既存の<code>Info.plist</code>にこのキーがないなら、まず追加して試す価値がある。
    SwiftUIだけで作っているプロジェクトや、手で<code>Info.plist</code>を管理しているプロジェクトでは、
    Xcodeのテンプレートが普段なら入れてくれるメタデータを落としやすい。
  </p>

  <h2>iPadの分割表示も疑う</h2>

  <p>
    iPadで「フルスクリーンになってほしいのに、ウィンドウっぽく見える」場合は、
    Split ViewやSlide Overの影響も確認する。
    その場合は<code>UIRequiresFullScreen</code>も候補になる。
  </p>

  <pre><code>&lt;key&gt;UIRequiresFullScreen&lt;/key&gt;
&lt;true/&gt;</code></pre>

  <p>
    ただし、このキーはiOS 26ではdeprecated warningが出る。
    将来的には効かなくなる方向なので、根本対策というより、
    現行環境でiPadの分割表示を避けたいときの確認ポイントとして扱うのがよさそう。
  </p>

  <h2>今回入れた形</h2>

  <p>
    今回のプロトタイプでは、<code>Info.plist</code>に次の2つを追加した。
  </p>

  <ul>
    <li><code>UILaunchScreen</code>: Storyboardなしの標準Launch Screen指定</li>
    <li><code>UIRequiresFullScreen = true</code>: iPad側の分割表示回避用</li>
  </ul>

  <p>
    本命は<code>UILaunchScreen</code>の方。
    「アプリが小さく表示される」「余白が出る」「昔のiPhone互換表示っぽい」
    という症状なら、レイアウトコードを触る前にここを見る。
  </p>

  <h2>確認手順</h2>

  <p>
    まず<code>Info.plist</code>を確認する。
  </p>

  <pre><code>plutil -p path/to/Info.plist | rg 'UILaunchScreen|UIRequiresFullScreen'</code></pre>

  <p>
    追加したら、plistの構文チェックをする。
  </p>

  <pre><code>plutil -lint path/to/Info.plist</code></pre>

  <p>
    そのあと、古いインストール状態を疑わないように、シミュレータや実機から一度アプリを削除して入れ直す。
    特にLaunch Screen周りは、ビルドし直したつもりでも古い表示を見ていると混乱しやすい。
  </p>

  <h2>まとめ</h2>

  <p>
    iOSアプリがフルスクリーンにならないとき、原因はViewのレイアウトではなく、
    アプリの起動メタデータ側にあることがある。
    SwiftUIのViewを疑う前に、まず<code>Info.plist</code>の
    <code>UILaunchScreen</code>を見る。
  </p>

  <p>
    小さいプロトタイプほど、こういうテンプレート由来の前提を落としやすい。
    次に同じ症状を見たら、まずここから確認する。
  </p>
</article>

<p class="entry-nav"><a href="/">← home</a></p>

