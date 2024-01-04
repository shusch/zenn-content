---
title: "JavaScript でタイムゾーン付きISO 8601形式に日付をフォーマットする"
emoji: "⏰"
type: "tech"
topics: ["javascript", "typescript", "web"]
published: true
---

# タイムゾーン付きのISO 8601形式でフォーマットする

日付の表記方法は世界中で多くのパターンがあります。
地域などで日と月の順序が異なることもあり混乱の原因であるため、統一的な表記形式として ISO 8601 (RFC 3339) というものが存在します。

JavaScript ではこのISO 8601形式の文字列を返すために、 `toISOString()` というメソッド[^1]があります。
ただしこれは常にUTC 協定世界時の時刻を返すため、タイムゾーンを保持したまま扱いたい場合には使い勝手が悪いです。

```js
const now = new Date()
console.log(now)
// Tue Jan 02 2024 19:37:56 GMT+0900 (日本標準時)

console.log(now.toISOString())
// '2024-01-02T10:37:56.324Z'
// タイムゾーンの情報が消失している！
```

タイムゾーン付きとなると、組み込みのメソッドはないため、独自に関数を定義する必要があるため以下のようなコードを書いておくと便利です。

## ISO 8601 (RFC 3339) とは

ここでは ISO 8601 の詳細な説明は行いませんが、UTCの場合とタイムゾーンありの場合とで表記例を記載します。

一般的な日本の時間表記 | タイムゾーン | ISO 8601 形式
--- | --- | ---
2024年1月2日 15時4分5秒 | 協定世界時 (UTC) | 2024-01-02T15:04:05Z
〃 | 日本標準時 (JST) | 2024-01-02T15:04:05+09:00

## 実装するコード

```ts
function toISOStringWithTimezone(date: Date): string {
  const year = date.getFullYear().toString();
  const month = zeroPadding((date.getMonth() + 1).toString());
  const day = zeroPadding(date.getDate().toString());

  const hour = zeroPadding(date.getHours().toString());
  const minute = zeroPadding(date.getMinutes().toString());
  const second = zeroPadding(date.getSeconds().toString());

  const localDate = `${year}-${month}-${day}`;
  const localTime = `${hour}:${minute}:${second}`;

  const diffFromUtc = date.getTimezoneOffset();

  // UTCだった場合
  if (diffFromUtc === 0) {
    const tzSign = 'Z';
    return `${localDate}T${localTime}${tzSign}`;
  }

  // UTCではない場合
  const tzSign = diffFromUtc < 0 ? '+' : '-';
  const tzHour = zeroPadding((Math.abs(diffFromUtc) / 60).toString());
  const tzMinute = zeroPadding((Math.abs(diffFromUtc) % 60).toString());

  return `${localDate}T${localTime}${tzSign}${tzHour}:${tzMinute}`;
}

function zeroPadding(s: string): string {
  return ('0' + s).slice(-2);
}
```

こちらのコードをどこかに定義しておいて、関数 `toISOStringWithTimezone()`` にDateオブジェクトを渡せばフォーマットされた文字列を返します。

```ts
const now = new Date();
const date = toISOStringWithTimezone(now);

console.log(date);
// '2024-01-02T19:47:20+09:00'
// 文字列型で値が返る
```

## Dateオブジェクトに戻す

JavaScriptで扱うにはDateオブジェクトで扱うほうが都合が良い場合もあります。
この場合でも ISO 8601 形式の文字列は簡単にDateオブジェクトに変換できます。

```ts
const dateStr = '2024-01-20T15:04:05+09:00';
const date = new Date(dateStr);

console.log(date);
// Sat Jan 20 2024 15:04:05 GMT+0900 (日本標準時)
```

[^1]: https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Date/toISOString
