---
title: "Laravel でストリームを使った効率の良いCSVダウンロード"
emoji: "🤠"
type: "tech"
topics: ["laravel", "php"]
published: true
---

## はじめに

Laravel でDBから取得したデータをCSVダウンロードするには色々な方法があり、たまに実装することになると混乱するので良い方法をまとめておきます。

## 環境

Laravel 10, 11

動作環境は Laravel 10, 11 ですが、6系などでも動作するはずです。

## CSVダウンロードするコード

最初にサンプルコードを記載します。

```php
Route::get('/export', function () {
    // 出力用のストリームを作成
    $stream = fopen('php://output', 'w');
    // Excelで文字化けしないように Shift_JIS で保存
    // UTF-8 で良ければ不要
    stream_filter_prepend($stream, 'convert.iconv.utf-8/cp932');

    $csvHeader = [
        // CSV ヘッダーのカラム名を定義する
        //
        // 'ユーザーID',
        // etc.
    ];
    fputcsv($stream, $csvHeader);

    $filename = 'ダウンロードファイル名.csv';

    return response()->streamDownload(function () use ($stream) {
        Post::query()
            ->select(['id', 'user_id', 'foo', 'bar'])
            ->chunkById(1000, function (\Illuminate\Support\Collection $posts) use ($stream) {
                foreach ($posts as $post) {
                    $row = [
                        // 書き込みたいカラムのデータ
                        $post->user_id,
                        $post->foo,
                        $post->bar,
                    ];
                    fputcsv($stream, $row);
                }
            });
        fclose($stream);
    }, $filename, [
        'Content-Type' => 'text/csv',
    ]);
});
```

## 実装のポイント

### ストリームを作成

```php
    // 出力用のストリームを作成
    $stream = fopen('php://output', 'w');
```

`php://output` を使い出力バッファに直接書き込みを行います。

よく `php://temp` のストリームに書き込み、最後に `fpassthru` で全データを出力している例も見かけます。
これはテンポラリファイルにデータを一度溜め込んだあと出力していますが、都度出力する `php://output` のほうが処理も早く、CSVが大容量になった場合でも問題が起きません。

ブラウザにダウンロードさせるのであれば `php://output` のほうを使いましょう。

### streamDownload を使用

Laravelには `response()->streamDownload()` という便利なメソッドがあります。
これを使ってブラウザでファイルダウンロードができるようなレスポンスを返します。

似たようなものに `response()->stream()` というメソッドもありますが、こちらはより汎用的なストリームレスポンス用です。
streamDownload を使うと、ダウンロード用のヘッダー `Content-Disposition: attachment` を自動で付与してくれるので、ダウンロード用には streamDownload を使いましょう。
https://laravel.com/docs/11.x/responses#streamed-responses

### chunkById でカーソルページネーション

```php
    Post::query()
        ->select(['id', 'user_id', 'foo', 'bar'])
        ->chunkById(1000, function (\Illuminate\Support\Collection $posts) use ($stream) {
```

`chunkById` を使い1000件ずつカーソルページネーションでデータを取得しています。

`->get()` を使った取得では一括で全データをDBから取得するため、大量のデータを扱うとメモリ落ちする可能性があります。
chunkById を使うと分割してデータを取得するため、メモリ消費を抑えながらデータを出力できます。

カーソルページネーションはメモリ効率と処理速度の面から最もおすすめですが、指定したユニークキー順にデータが取得されるので、ユニークキーなどに限らず `ORDER BY` を柔軟に指定したいときなどには使えません。
その場合は `chunk()` を使いましょう。

chunk() は limit と offset を使いページネーションするため、件数が多くなるほど遅くなりますが、chunkById() より柔軟に並べ替えや絞り込みができます。

Laravel で使えるデータ取得のメソッドは以下で詳しくまとめられています。
https://qiita.com/nekohan/items/eba0816fe8c21e3cc077

### レスポンスヘッダー

```php
        }, $filename, [
            'Content-Type' => 'text/csv',
        ]);
```

streamDownload は第3引数に渡した配列をレスポンスヘッダーに追加してくれます。
今回はCSVダウンロードなので `'Content-Type' => 'text/csv'` を返しています。

ファイルダウンロードを強制するために `Content-Type: application/octet-stream` や `Content-Type: application/force-download` としている例もありますが、どちらも正しい使い方とはいえないので避けるべきでしょう。

ブラウザにファイルダウンロードをさせたければ、 `Content-Disposition: attachment` を返すのが適切です。
そして先述したように Laravel の streamDownload はこのヘッダーを自動で付与してくれます。

今回はCSVを返していることが自明なので、あくまで Content-Type は **text/csv** が適切です。

#### 補足: application/octet-stream と application/force-download について

application/octet-stream は未知のバイナリ形式を表す MIME タイプで、これを指定するとブラウザは普通ダウンロードを実行します。
ただしファイルダウンロードのためであれば Content-Disposition: attachment とするのが筋で、今回のようにあらかじめ MIME タイプがわかっているのであれば、わざわざ octet-stream を使う必要はないでしょう。

https://developer.mozilla.org/ja/docs/Web/HTTP/MIME_types#applicationoctet-stream

application/force-download も同様に、Content-Type に指定することでブラウザにダウンロードを実行させられます。

しかし MIME タイプに application/force-download というものは存在せず、あくまで慣習的に使われているものになります。
これは未知の MIME タイプを受け取ると、ファイルを実行せずダウンロードするというブラウザの挙動によるものです。
先ほどと同じく、Content-Disposition: attachment を使うのが筋なので使う必要はありません。
