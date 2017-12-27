---
layout: post
title: Vue.jsを使ったGitHub Pages+Jekyllでの簡易予約投稿
outline: JavaScriptフレームワークであるVue.jsを使って、GitHub Pagesに簡易的な予約投稿機能を実装しました。単に予約時刻以前であれば、トップページからのリンクを非表示にしているだけなので擬似的な予約投稿となります。
reserve: 2017-12-27 18:50:00 +0900
published: true
tags: 
- GitHubPages
- Vuejs
---
こんばんは、kat-kai ([@katkai3](https://twitter.com/katkai3)) です。本記事ではVue.jsを使ってGitHub Pages+Jekyllの予約投稿の実装について書いています。GitHub Pagesでの予約投稿(Schedule post)に関して調べてみたんですが、解決方法が無さそうだったので、今回作ってみました。

Jekyll自体には予約投稿を可能にするsite.futureオプションがあるんですが、GitHub Pagesでは使えないみたいです。
- [blogs - Is it possible to schedule posts with Jekyll? - Stack Overflow](https://stackoverflow.com/questions/4923867/is-it-possible-to-schedule-posts-with-jekyll)
- [github pages + jekyll で予約投稿 - Xeebi](https://lesguillemets.github.io/blog/2014/06/26/jekyll-future.html)

実装内容としては、Vue.js (JavaScript)を使って予定時間より前であれば非表示するだけのシンプルな実装です。つまりトップページからはリンクが見えなくなっているだけで、記事自体は公開状態になっているため、その記事のアドレスが分かれば、誰でもアクセスすることは可能になります。あくまでも擬似的な予約投稿となりますので、その点はお気をつけください。

私にとっては、この程度の機能で十分でしたので、本ブログで使っています。

本記事の以下に示すコードはご自由にご利用ください。ただし不具合等が生じても一切、責任は負いません。

## 仕組み
まず投稿する.mdファイルのFront-matterで、投稿予定時刻を示す変数reserveをセットします。
```reserve: 2017-12-30 10:30:00 +0900```

そして変数reserveの値を元に、index.htmlで記事毎に以下のように出力します。
```
<article class="post" v-if="isPast(`2017-12-30T10:30:00+0900`)">
  (記事の日付, タイトル等)
</article>
```
後述するisPast関数では、予定時刻を過ぎているかどうかをtrue/falseで返します。
そして表示・非表示の制御には、Vue.jsの条件付きレンダリング機能である```v-if```を利用します。

Vue.jsでは、v-ifの属性値の式をJavaScriptとして評価することが出来るため、
isPast関数の戻り値によって表示・非表示を制御することができます。


## 予約投稿機能を使うための変更箇所
変更を加えるファイルは以下の通りです。
- _layouts/default.html
- _posts/*.mdファイル
- index.html

### _layouts/default.html  

body要素内の末尾に以下を追加します。  
```javascript
<script src="https://cdn.jsdelivr.net/npm/vue"></script>
<script>
  var app = new Vue({
    el: '#articles',
    methods: {
      isPast: function(strXmlSchema) {
        return (new Date().getTime() > new Date(strXmlSchema).getTime());
      }
    }
  })
</script>
```
Vue.js (執筆時点ではv2.5.13) を読み込み、isPast()関数を追加します。


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
次にindex.htmlの編集です。Articleタグに予定投稿機能を追加するために、以下のように変更します。

```html
{% raw %}<div class="posts">
  {% for post in site.posts %}
    <article class="post">{% endraw %}
```  
を  
```html
{% raw %}<div class="posts" id="articles">
  {% for post in site.posts %}
    {% if post.reserve %}
    <article class="post" v-if="isPast(`{{ post.reserve | date_to_xmlschema }}`)">
    {% else %}
    <article class="post">
    {% endif %}{% endraw %}
```  
に変更します。

## 使い方

上記の変更さえしてしまえば、あとは*.mdファイル中のFront-matterでreserveをセットするだけです。
例えば2017年12月30日の10時30分0秒に記事を公開したい場合は、以下のような感じです。

```
---
layout: post
title: Article Title
reserve: 2017-12-30 10:30:00 +0900
---
```
投稿予約時間が過ぎても動的な表示はしません。F5等でページの再読込をすると表示されます。


## おわりに
擬似的な予約投稿機能ではありますが、機能として組み込むことが出来ました。

趣味として自分のペースでブログを書いていると、仕上げを先延ばししちゃったりします。
今回は、期日っぽいのを設定して先延ばしを改善できればなーと思って取り組みました。それでは！
