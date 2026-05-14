---
layout: layout.njk
title: グループ精算アプリをCloudflare WorkersとD1で作った
description: 旅行や飲み会の立替を共有URLに集約し、最後に誰が誰へいくら払えばよいかを計算するWebアプリの技術構成を紹介します。
summary: URL共有、レシート画像登録、最小送金表示までをCloudflare Workers Static Assets + D1で小さく実装したグループ精算アプリの紹介。
tags:
  - post
---

<article class="entry">
  <p class="meta">2026-05-15</p>
  <h1>グループ精算アプリをCloudflare WorkersとD1で作った</h1>

  <p>
    旅行や飲み会のあとに地味に面倒なのが、誰が何を立て替えて、最後に誰が誰へいくら払えばいいのかを整理する作業です。
    メッセージアプリやメモにレシートを投げておく運用でもなんとかはなりますが、人数が増えると
    「合計はいくらか」「1人あたりいくらか」「結局だれに送金するのか」がすぐ曖昧になります。
  </p>

  <p>
    そこで、グループごとに共有URLを作り、参加者・立替金額・レシート画像を登録すると、
    最後の送金額まで自動計算するWebアプリを作りました。
  </p>

  <p>
    公開URL:
    <a href="https://group-settlement-app.rei-hiragram.workers.dev">https://group-settlement-app.rei-hiragram.workers.dev</a>
  </p>

  <h2>できること</h2>

  <p>
    このアプリでは、まず旅行やイベント単位でグループURLを作成します。
    そのURLを参加者に共有すれば、同じ精算ページにアクセスできます。
  </p>

  <ul>
    <li>グループURLの作成</li>
    <li>参加者の追加</li>
    <li>立替内容・金額・レシート画像の登録</li>
    <li>合計金額と1人あたり金額の表示</li>
    <li>誰が誰へいくら払えばよいかの表示</li>
  </ul>

  <p>
    たとえばAさんが多めに払い、BさんとCさんがあまり払っていない場合、
    アプリ側で各参加者の差額を計算し、「CさんがAさんへ4,000円」「BさんがAさんへ1,000円」のように表示します。
  </p>

  <h2>技術構成</h2>

  <p>
    構成はかなり小さくしています。
  </p>

  <ul>
    <li>Frontend: HTML / CSS / vanilla JavaScript</li>
    <li>Backend: Cloudflare Workers</li>
    <li>Hosting: Cloudflare Workers Static Assets</li>
    <li>Database: Cloudflare D1</li>
    <li>CI/CD: GitHub Actions + Wrangler</li>
    <li>Test: Node.js built-in test runner</li>
  </ul>

  <p>
    フロントエンドはSPA風ですが、Reactなどのフレームワークは使っていません。
    今回の用途では画面数も状態も少ないため、vanilla JavaScriptで十分でした。
    依存を増やさないことで、Cloudflare Workersへのデプロイも軽く保てます。
  </p>

  <h2>Cloudflare Workers + D1にした理由</h2>

  <p>
    このアプリは、常時大量アクセスがあるサービスではなく、旅行やイベントの期間だけ使われる小さなツールです。
    そのため、サーバーを常時立てるよりも、Cloudflare Workersのようなエッジ実行環境に寄せたほうが運用が楽です。
  </p>

  <p>
    データはCloudflare D1に保存しています。
    保存しているのは、グループ、参加者、立替レシート情報です。
    レシート画像はクライアント側で縮小し、Data URLとしてD1に保存しています。
    画像ストレージを別に持つ設計も考えられますが、今回のような小規模な精算用途では、構成をシンプルに保つことを優先しました。
  </p>

  <h2>精算ロジック</h2>

  <p>
    精算計算は<code>calculateSettlements</code>に切り出しています。
    処理の流れはシンプルです。
  </p>

  <ol>
    <li>全レシートの合計金額を出す</li>
    <li>参加者数で割って1人あたり負担額を出す</li>
    <li>各参加者について「支払済み金額 - 本来の負担額」を計算する</li>
    <li>プラスの人を受け取り側、マイナスの人を支払い側に分ける</li>
    <li>支払い側から受け取り側へ順番に金額を埋めていく</li>
  </ol>

  <p>
    この方式にすると、「全員が全員に細かく払う」のではなく、必要な送金だけを表示できます。
    精算アプリで一番欲しいのは会計の正確さだけでなく、最後の行動に落ちる表示なので、
    この部分は独立した関数としてテストを書いています。
  </p>

  <h2>デプロイ</h2>

  <p>
    GitHub Actionsでは、<code>main</code>にpushされたら次の流れで自動デプロイします。
  </p>

  <ul>
    <li><code>npm ci</code></li>
    <li><code>npm test</code></li>
    <li>D1 migration</li>
    <li>Worker deploy</li>
  </ul>

  <p>
    CloudflareのAPI tokenはGitHub Secretsに入れ、チャットやログには出さない運用にしています。
    D1のmigrationもCIに含めているので、DBスキーマ変更とアプリのデプロイを同じpushで扱えます。
  </p>

  <h2>作ってみて</h2>

  <p>
    今回のポイントは、フレームワークや外部サービスを増やさず、URL共有だけで使える精算体験に絞ったことです。
    旅行中に使うツールは、ログインや招待フローが重いと使われにくいので、
    「URLを作る」「参加者を入れる」「レシートを登録する」「最後に送金先を見る」だけにしています。
  </p>

  <p>
    今後足すなら、レシート削除、参加者削除、編集履歴、R2への画像保存あたりが自然です。
    ただ、現時点でも小さな旅行や飲み会の精算には十分使える形になりました。
  </p>
</article>

<p class="entry-nav"><a href="/">← home</a></p>
