---
layout: post
title: Vue.jsを使ったGitHub Pages+Jekyllでの簡易予約投稿
outline: JavaScriptフレームワークであるVue.jsを使って、GitHub Pagesに簡易的な予約投稿機能を実装しました。単に予約時刻以前であれば、トップページからのリンクを非表示にしているだけなので擬似的な予約投稿となります。
reserve: 2017-12-27 14:55:00 +0900
published: false
tags: 
- GitHubPages
- Vue.js
---
こんばんは、kat-kai ([@katkai3](https://twitter.com/katkai3)) です。本記事ではVue.jsを使ってGitHub Pages+Jekyllの予約投稿の実装について書いています。GitHub Pagesでの予約投稿(Schedule post)に関して調べてみたんですが、解決方法が無さそうだったので、今回作ってみました。

Jekyll自体には予約投稿を可能にするsite.futureオプションがあるんですが、GitHub Pagesでは使えないみたいです。
- [blogs - Is it possible to schedule posts with Jekyll? - Stack Overflow](https://stackoverflow.com/questions/4923867/is-it-possible-to-schedule-posts-with-jekyll)
- [github pages + jekyll で予約投稿 - Xeebi](https://lesguillemets.github.io/blog/2014/06/26/jekyll-future.html)

実装内容としては、Vue.js (JavaScript)を使って予定時間より前であれば非表示するだけのシンプルな実装です。
つまりトップページからはリンクが見えなくなっているだけで、記事自体は公開状態になっているため、その記事のアドレスが分かれば、誰でもアクセスすることは可能になります。あくまでも擬似的な予約投稿となりますので、その点はお気をつけください。

私にとっては、この程度の機能で十分でしたので、本ブログで使っています。

本記事の以下に示すコードはご自由にご利用ください。ただし不具合等が生じても一切、責任は負いません。

## 予約投稿機能を使うための変更箇所
変更を加えるファイルは以下の通りです。
- _posts/*.mdファイル
- index.html

### *.mdファイル
.mdファイルの先頭の箇所に、公開予定日時をreserveに書き加えるだけです。
```
---
layout: post
title: Article Title
reserve: 2017-12-30 10:30:00 +0900
---
```
書き加えなければ、通常通りで投稿後にトップページに表示されます。

### index.html
次にindex.htmlの編集です。

**1. Vue.jsの読み込み**  
```html
<script src="https://cdn.jsdelivr.net/npm/vue"></script>
```
を追加します。

**2. Articleタグに予定投稿機能を追加**  
```html
<article class="post">
```
を  
```html
{% raw %}{% if post.reserve %}
<article class="post" id="article" v-if="isPast(`{{ post.reserve | date_to_xmlschema }}`)">
{% else %}
<article class="post" id="article">
{% endif %}{% endraw %}
```
に変更します。後述するisPast()関数により指定した予約時間(post.reserve)が過ぎているかを判定します。
予定時間が過ぎていればtrueを返し、```v-if="true"```となります。
また```v-if```はVue.jsの条件付きレンダリング機能であり、trueになるまで描写されません。

**3. isPast()関数の追加**  
最後に末尾に以下の<script>タグを追加して、準備は完了です。
```javascript
<script>
  var app = new Vue({
    el: '#article',
    methods: {
        isPast: function(strXmlSchema) {
            var now = new Date().getTime();
            var rsvTime = new Date(strXmlSchema).getTime();

            return (now > rsvTime);
        }
    }
  })
</script>
```
変数now, rsvTimeには、それぞれ現在時刻・予約時刻の1970年1月1日0時0分0秒を起点とした経過ミリ秒が代入されています。

## 使い方
*.mdファイル中でJekyllの変数であるpage.dateで指定します。  
例えば2017年12月30日の10時30分0秒に記事を公開したい場合は、以下のような感じです。

```reserve: 2017-12-30 10:30:00 +0900```

投稿予約時間が過ぎても動的に表示はしません。F5とかでページの再読込をすると表示されます。


## おわりに
擬似的な予約投稿機能ではありますが、機能として組み込むことが出来ました。

やはり趣味で書いていく分だと、記事を完成させたりするのがなかなか大変で
何かしらのデッドラインみたいなものが無いと、記事を仕上げるのを先延ばしにしちゃったりします。

まだ私自身はブログ記事をいくつか書いただけですが、継続して情報発信出来る人って改めてすごいなと感じました。

これで期日までに余裕を持って、記事を書くことが出来ればなーと思います。
