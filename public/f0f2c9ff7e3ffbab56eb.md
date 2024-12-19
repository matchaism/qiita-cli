---
title: Adventarの投稿をSlackやDiscordに通知する / UEC Advent Calendar 2024
tags:
  - JavaScript
  - GoogleAppsScript
  - TypeScript
  - Slack
  - discord
private: false
updated_at: '2024-12-16T18:13:57+09:00'
id: f0f2c9ff7e3ffbab56eb
organization_url_name: null
slide: false
ignorePublish: false
---
## はじめに (本題とは無関係)

こんにちは，matchaism/抹茶です．

こちらは[UEC Advent Calendar 2024](https://adventar.org/calendars/10127)の8日目の記事です．([maccha Advent Calendar 2024](https://adventar.org/calendars/10199)の記事でもある)

昨日の記事はYちゃんさんの[歌奏絆(かなでつぐ)を生み出して](https://y-chan.dev/blog/kanade-tsugu-2024-12/)です．

https://y-chan.dev/blog/kanade-tsugu-2024-12/

先月のMIKUEC Remixにて，ついに歌奏絆のライブが披露されました[^1]．昨日の記事を読むとYちゃんさん視点でのドラマや，歌奏絆の成長に携わった人々の存在が現れ，VLLが掲げる「広がれ、創作の輪。」[^2]を垣間見た気がします．

さて，私は2020年から5年連続でUEC Advent Calendarに参加しておりますが，今回で最後の参加になります．色々語りたい話や，歌奏絆に関するオタクトークは[UEC Advent Calendar 2024 Day8 あとがき](https://macchanism.hateblo.jp/entry/uec_advent_calendar_2024_appendix)に書きました．興味がある方はご覧ください．

最後は電通大生らしく，襟足正してTechな記事で締めたいと思います．

## この記事について

ここから本題です．今回，[Adventar](https://adventar.org/)で開催されるAdvent Calendarに記事が投稿/公開された旨を，SlackやDiscordで通知してくれるツールを作りました．

AdventarはユーザがAdvent Calendarを自由に立ち上げ，投稿記事をまとめることができるサイトです．例えば我々のように，同じ大学の学生を集めてアドカレを開催，寄稿するといった使われ方をしています．(図は[maccha Advent Calendar 2024](https://adventar.org/calendars/10199))

<img width="80%" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/503263/af9ab99c-9f8b-686f-90f9-c8bece22b802.png">

開発した理由は2つあります．1つ目はAdventarの記事投稿を見逃さないためです．執筆者は必ず担当日に記事を公開してくれるとは限りません．時間が経過してから投稿されることもあります．そういった記事が読まれずに年越ししてしまうのは避けたいです．

2つ目の理由は，友人がいるSlackやDiscordのチャンネルに，記事投稿の通知が来ると盛り上がりそうだからです．これは私の友人[azarasing](https://azarasing.hatenablog.com/)くんの要望でもあります．

## 実装

先に使われた技術をまとめると :
- プログラムはJavaScript
- 最新のAdvent Calendarの情報はAdventarからスクレイピング
- スクレイピングした情報はGoogleスプレッドシートに記録
- 実行環境はGoogle Apps Script (Googleスプレッドシートの拡張機能Apps Scriptから)

### 概要

更新前のAdventarの情報はGoogleスプレッドシートに記録されます．最新のAdventarの情報はWebサイトから直接スクレイピングし，抽出されます．両者の差分を見つけ，新たに投稿/公開された記事の情報がSlack/Discordに投稿されます．一連の処理はGoogle Apps Scriptで動作します．

<img width="70%" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/503263/a85a54e8-11ab-4cf3-43a6-39457deb1dd6.png">

この記事では説明のため，実際のコードから改変したものを掲載しています．ソースコードの全貌を見たい方は[こちら(GitHub)](https://github.com/matchaism/adventar_bell/tree/v2.0.1/src)を確認してください．

### Adventarからのスクレイピング

`Cheerio`ライブラリでAdventarからスクレイピングをしています．スクレイピングする内容は :

- 各日の投稿者名
- 記事リンク
- 投稿状態 (`registered`:登録のみ、`posted`:記事投稿済み)

日ごとの情報を抽出し，`CalendarEntry`(下記)のインスタンスに結果をすべて格納します．

```javascript
class CalendarEntry {
  constructor(title, url) {
    this.title = title; // Adventarのタイトル
    this.url = url; // AdventarのURL
    this.calendarStatus = Array(config.DAYS_IN_CALENDAR); // 'null', 'registered', 'posted', 'no_change'
    this.authors = Array(config.DAYS_IN_CALENDAR); // 各日の投稿者
    this.articles = Array(config.DAYS_IN_CALENDAR); // 各日の記事URL
    ~~~~~ (中略) ~~~~~
  }
},
```

以上の処理を`getCalendarEntryFromWeb`関数で行います．

<details>
<summary>getCalendarEntryFromWeb関数</summary>

```javascript
// Webから最新のカレンダー情報をスクレイピング
function getCalendarEntryFromWeb(row) {
  const calendarEntry = new adventarBell.CalendarEntry(row[0], row[1]); // CalendarEntryクラスの宣言
  const html = UrlFetchApp.fetch(row[1]).getContentText(); // Adventarの取得
  const $ = Cheerio.load(html); // ライブラリCheerio
  const entryList = $('ul.EntryList').find('li'); // リストアップ
  entryList.each(function() {
    const date = $(this).find('div.head > div.date').text();
    const day = parseInt(date.split('/')[1]);
    const author = $(this).find('div.head > div.user > a').text();
    const articleLink = $(this).find('div.article > div.left > div.link > a').attr('href');
    // 記録
    calendarEntry.authors[day - 1] = author;
    calendarEntry.articles[day - 1] = articleLink;
    if (typeof articleLink === 'undefined') { // 登録されているが、記事が投稿されていない
      calendarEntry.calendarStatus[day - 1] = 'registered';
    } else { // 記事が投稿されている
      calendarEntry.calendarStatus[day - 1] = 'posted';
    }
  });
  return calendarEntry;
}
```

</details>

### Googleスプレッドシートからの取得

`getCalendarEntryFromSpreadsheet`関数では，スプレッドシートにある各日の投稿/更新情報(`registered`または`posted`，空白・不正値は`null`)をプログラムで扱いやすい形式に変換します．結果をまとめ，`CalendarEntry`インスタンスに格納します．

<img width="100%" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/503263/6444b25c-dab8-448c-4dfa-3914853e5a1c.png">

<details>
<summary>getCalendarEntryFromSpreadsheet関数</summary>

```javascript
// スプレッドシートからカレンダー情報を抽出
function getCalendarEntryFromSpreadsheet(row) {
  const calendarEntry = new adventarBell.CalendarEntry(row[0], row[1]); // CalendarEntryクラスの宣言
  for (let day = 1; day <= config.DAYS_IN_CALENDAR; day++) { // 各日の情報を取得
    let status = row[day + 1];
    if (status !== 'registered' && status !== 'posted') status = null; // 'registered'でも'posted'でもないとき，null
    calendarEntry.calendarStatus[day - 1] = status; // 記録
  }
  ~~~~~ (中略) ~~~~~
  return calendarEntry;
}
```

</details>

### 差分・更新検出

前2つの処理で前回のAdvetarの情報(`prevEntry`)と，現在の情報(`currentEntry`)が手に入りました．両者を比較することで，投稿/更新情報の変化を確認します．

<details>
<summary>getCalendarDifference関数</summary>

```javascript
// カレンダー情報の差分を取得
function getCalendarDifference(prevEntry, currentEntry) {
  ~~~~~ (中略) ~~~~~
  const diffEntry = new adventarBell.CalendarEntry(currentEntry.title, currentEntry.url); // CalendarEntryクラスの宣言
  for (let i = 0; i < config.DAYS_IN_CALENDAR; i++) { // 差分検出と記録
    if (prevEntry.calendarStatus[i] !== currentEntry.calendarStatus[i]) {
      diffEntry.calendarStatus[i] = currentEntry.calendarStatus[i];
      diffEntry.authors[i] = currentEntry.authors[i];
      diffEntry.articles[i] = currentEntry.articles[i];
    } else {
      diffEntry.calendarStatus[i] = 'no_change';
    }
  }
  ~~~~~ (中略) ~~~~~
  return diffEntry;
}
```

</details>

この後，検出された差分はスプレッドシートに反映します．

### Slack/Discordへ通知

手に入った差分の情報をSlack/Discordへ通知します．作成したPayloadをWebhookを使ってPOSTします．

<details>
<summary>Payload</summary>

```slack.json
{
  "username": "UEC Advent Calendar 2024",
  "icon_emoji": ":christmas_tree:",
  "unfurl_links": true,
  "unfurl_media": true,
  "blocks": [{
    "type": "section",
    "text": {
      "type": "mrkdwn",
      "text": "matchaism posted </uru/to/article|Day 8 Article>!!"
    }
  }]
}
```

```discord.json
{
  "username": "UEC Advent Calendar 2024",
  "content": "matchaism posted Day 8 Article!!\r/uru/to/article"
}
```

</details>

<img width="50%" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/503263/43d84aa5-aa3e-4381-686c-fd7d1e17a6b3.png"><img width="50%" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/503263/79988856-fb6e-c6bf-da5c-b92a29197775.png">

## 最後に

今回の開発のリポジトリはGitHubで公開しています．(後述しますが，後でTypeScriptで書き直しました．こちらはJavaScript版のリンクです．)

https://github.com/matchaism/adventar_bell/tree/v2.0.1

余談ですが，私はこのコードをGitHubにpush後，[このやり方](https://macchanism.hateblo.jp/entry/maccha_advent_calendar2024_day5)でGitHub Actionsにより自動でデプロイさせています．(内部的には`clasp push`&`clasp deploy`)

以上となります．

### 追記

同プロジェクトをTypeScriptで書き直しました．こちらが最新のリンクになります．

https://github.com/matchaism/adventar_bell

---

明日はこう(昼飯)さんの記事です．

https://adventar.org/calendars/10127

[^1]: https://www.youtube.com/watch?v=rJNUfNG4FIE
[^2]: https://mikuec.com/
