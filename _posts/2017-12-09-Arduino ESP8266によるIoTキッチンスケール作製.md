---
layout: post
title: Arduino ESP8266を使ったIoTキッチンスケールの作製
outline: Arduino IDEで開発可能なWiFiモジュール ESP8266を使って、市販のキッチンスケールをIoT化しました。
tags: 
- Arduino
- ESP8266
- IoT
---

こんにちは、kat-kaiです。[Arduino Advent Calendar 2017](https://qiita.com/advent-calendar/2017/arduino)の9日目です。  
今回は、WiFi経由で使えるキッチンスケールを作ってみたいと思います。

## はじめに
### なにを作ったの？
<blockquote class="twitter-tweet tw-align-center" data-lang="en"><p lang="ja" dir="ltr">Arduino Advent Calendar 2017 9日目で紹介するIoTキッチンスケールの動画！スマートフォンからでも重さを取得できるようになりました！ <a href="https://t.co/bdcMemnujy">pic.twitter.com/bdcMemnujy</a></p>&mdash; kat-kai (@katkai3) <a href="https://twitter.com/katkai3/status/939388375613587457?ref_src=twsrc%5Etfw">December 9, 2017</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>


今回は、インターネット経由で重さを取得できるIoTキッチンスケールを作りました！

### ESP-WROOM-02/ESP8266ってなに？
ESP-WROOM-02はEspressif systems社で製造されているWiFiモジュールです。  
そして、このモジュールに搭載されているマイコンがESP8266です。  
このESP8266は、統合開発環境Arduino IDEを使って開発することができます。  

![esp8266](https://user-images.githubusercontent.com/21113258/34217434-d8543c38-e5ee-11e7-9e91-819493321328.jpg){:style="display:block;margin-left:auto;margin-right:auto;"}

中央のチップがESP-WROOM-02 (ESP8266)でWiFiのみ、右側のチップが後続品ESP-WROOM-32 (ESP32)でWiFi/Bluetoothが使える。

ただモジュール単独だと、開発には別途USBシリアル変換モジュールも必要になるので  
USBシリアル変換が搭載されている開発ボードを用いると、開発が容易になります。

今回は開発ボードを使わずに、ESP8266をキッチンスケールに組み込むことでIoT化したいと思います。

## イメージ図
まず全体のイメージ像について説明します。  

市販のキッチンスケールの中にESP8266を組み込みます。  
ESP8266で重さを取得できるようにしつつ、Webサーバとして動かします。  
そしてWiFi経由でESP8266にアクセスすることでスマートフォンでも重さを取得できるようにします。  

![whole_img](https://user-images.githubusercontent.com/21113258/33708063-c3565c5c-db7c-11e7-8ce3-a51f4d731551.png)

今回は赤枠の中で示すようにローカルな環境に留まっていますが  
ネット上のレシピ等で連携できたら面白そうだなーって思います。

それでは以下のような手順でしていきたいと思います。

[使ったもの](#使ったもの){:rel="nofollow"}  
[キッチンスケールの仕組み](#キッチンスケールの仕組み){:rel="nofollow"}  
[Step1. ESP8266で重さを取得する回路を組む](#step1-esp8266で重さを取得する回路を組む){:rel="nofollow"}  
[Step2. Arduino ESP8266/HX711を使った重さの取得/校正](#step2-arduino-esp8266hx711を使った重さの取得校正){:rel="nofollow"}  
[Step3. Arduino ESP8266によるWebサーバーの実装](#step3-arduino-esp8266によるwebサーバーの実装){:rel="nofollow"}  
[Step4. インターフェイスの作成](#step4-インターフェイスの作成){:rel="nofollow"}  

## 使ったもの
- タニタ キッチンスケール KF-100-WH
- Espressif WiFiモジュール ESP8266
- 超小型USBシリアル変換モジュール AE-FT234X (開発ボードを使う場合不要です)
- 重量計りADCモジュール HX711-M
- ハンダごて、ハンダ (キッチンスケールに配線する場合に必要)
- ブレッドボード (試作時に必要)

ちなみに購入場所は、
- キッチンスケール Amazon  
- ESP8266 スイッチサイエンス(ESP-WROOM-02ピッチ変換済みモジュール シンプル版として)  
- シリアル変換モジュール 秋月電子  
- 重量計りADCモジュール HX711-M aitendoです。  

改めて見ると購入場所が見事にバラバラですが、
電子部品に限っては1箇所からの購入で全てが揃うと思います。


## キッチンスケールの仕組み
次に、どうやってESP8266を使ってキッチンスケールから重さを取得するのかについてです。  

その前に、そもそもキッチンスケール自身はどうやって重さを取得しているのかを説明します。  
まずキッチンスケールの中には、「ひずみゲージ」と呼ばれるセンサーが入っています。  

キッチンスケールのフタを開けると、こんな感じで中心の金属の物体が「ひずみゲージ」です(HX711/ESP8266は取り付け済み)  
![scale1](https://user-images.githubusercontent.com/21113258/34217439-dc5ca55e-e5ee-11e7-93ef-c917a59f1c52.jpg){:style="display:block;margin-left:auto;margin-right:auto;"}

この「ひずみゲージ」には赤・黒・緑・白の4本の配線があります。  
![strain_gauge](https://user-images.githubusercontent.com/21113258/34217437-da6b8a12-e5ee-11e7-972f-7fc3f1d7f968.jpg){:style="display:block;margin-left:auto;margin-right:auto;"}


この「ひずみゲージ」に外部から力が加わると、その力の大きさに応じて  
金属材料の抵抗値が変化し、その抵抗値の変化量から重さを求めます。

今回、ESP8266でも同様に重さを取得しますが、ADCモジュール HX711とそのライブラリを使うと  
コードを数行書くだけで、重さを取得することができます！ライブラリ作者の方に感謝です！
- [bogde/HX711](https://github.com/bogde/HX711)

## Step1. ESP8266で重さを取得する回路を組む
それではキッチンスケール-HX711-ESP8266を、以下のように配線したいと思います。  
また電源はキッチンスケールの電源(単4電池x2)を流用します。

![fritzing2](https://user-images.githubusercontent.com/21113258/33792401-fcab9e4e-dce0-11e7-84c1-fdfe04b6bdc1.png)

| ひずみゲージ | HX711 | ESP8266 |
|:-----------:|:-----:|:-------:|
| 赤          | E+    |         |
| 白          | E-    |         |
| 緑          | A-    |         |
| 青          | A+    |         |
|             | DT    | GPIO14  |
|             | SCK   | GPIO13  |

ひずみゲージ、HX711とESP8266を配線し、キッチンスケール内部の隙間にいれます。  
写真ではいきなり配線しているように見えますが、予めブレッドボードを使って  
Step2を試してから配線してます。

![scale2](https://user-images.githubusercontent.com/21113258/34217445-dea40294-e5ee-11e7-944c-89df196dd53e.jpg){:style="display:block;margin-left:auto;margin-right:auto;"}

キッチンスケール-HX711の配線は[こちらのサイト](http://barcelona.lomo.jp/wp/?p=23)を参考にさせて頂きました。  
またFritzing ESP8266は[こちら](https://github.com/houtbrion/fritzing-parts/tree/master/ESP-WROOM-02)から使わせて頂いております。

## Step2. Arduino ESP8266/HX711を使った重さの取得/校正
それでは次にESP8266で重量を取得してみます！  
HX711のライブラリに同梱されているサンプルコードをESP8266に書き込んで校正を行います。

以下のサイトを参考に重さの取得・校正を行いました。
- [ロードセルを使った簡易スケールの製作](http://barcelona.lomo.jp/wp/?p=23)
- [歪みゲージ(ロードセル)と HX711 を使って重量計測する (Arduino)](https://lowreal.net/2016/12/25/1)

今回は、詳細については省きます！

## Step3. Arduino ESP8266によるWebサーバーの実装
次にスマートフォンからも重さを取得できるようにします。  
ここでESP8266はWebサーバーとしても使えるので、取得した重さをWebサーバーで公開し  
そこにスマートフォンでアクセスをして重さを取得する、といった形式にしたいと思います。  

[ESP8266_scale.ino](#esp8266_scaleino)を書き込みます。  
そして```http://(ESP8266のIPアドレス)/weight```にアクセスすると、  
スマートフォンから重量を取得できるようになりました！  
また風袋引きするなら```http://(ESP8266のIPアドレス)/tare```で出来ます。

これはこれで嬉しいんですが、まだ味気ないので  
次のステップでは、もう少し見栄えを良くしたインターフェイスを作ります


## Step4. インターフェイスの作成
次にスマートフォンでアクセスしたときの画面を作ります。  
イメージとしては、下の円グラフのアニメーションのように視覚的にも重さを表示する感じです。  
例えば小麦粉300グラム必要な場合、現在の重さをオレンジ色で、過剰分を赤色で表示したいと思います。  

![movie](https://user-images.githubusercontent.com/21113258/33702976-43da9280-db69-11e7-9214-512102009f17.gif)


今回は[Chart.js](http://www.chartjs.org/)を使って、円グラフを書きます。  
この[index.html](#indexhtml)を、ESP8266の内蔵フラッシュメモリにSPIFFSファイルシステムで書き込みます。



これでシステムは完成しました！これでスマートフォンでアクセスして  
キッチンスケールに載せると、冒頭の動画のように円グラフが変化していきます。


## おわりに
今回はIoTキッチンスケール作製でしたが、いかがでしたか。  
スマートフォン単体では出来ないことを、ESP8266を用いて連携させる形を取りました。  

もう少し作り込んで、レシピと連携が出来るようになると面白そうですね！  

今回は以上となります！次回の12日目では詳細は未定ですが  
ESP32かESP8266を使って、また別のものを作ってみたいと思います！
それでは！

## Appendix
後ほどGitHubにアップロードします。  

### ESP8266_scale.ino
```c
#include "HX711.h"
#include <ESP8266WiFi.h>
#include <ESP8266WebServer.h>
#include <FS.h>

//-------------------------------------------------------------------------
//  WiFi Data
//-------------------------------------------------------------------------
#define WLAN_SSID       "SSID"
#define WLAN_PASS       "PASSWORD"
//-------------------------------------------------------------------------

ESP8266WebServer server(80);

HX711 scale;
String weight = "";

void getLocalFile(String filename, String dataType) {
  SPIFFS.begin();
  File data = SPIFFS.open(filename, "r");
  server.streamFile(data, dataType);
  data.close();
}

void setup() {
  ESP.wdtDisable();
  Serial.begin(115200);
  
  WiFi.begin(WLAN_SSID, WLAN_PASS);
  Serial.println("");
  while(WiFi.status() != WL_CONNECTED){
    delay(1000);
    Serial.print(".");
  }
  Serial.println("");
  Serial.print("IP Address: ");
  Serial.println(WiFi.localIP());

  ESP.wdtFeed();
    
  Serial.println("start");
  scale.begin(14, 13); //DT:14, SCK:13
  
  Serial.print("read:");
  Serial.println(scale.read());

  scale.set_scale();
  scale.tare();

  ESP.wdtFeed();
  
  Serial.print("calibrating...");
  delay(5000);
  Serial.println(scale.get_units(10));

  
  scale.set_scale(-1542.00);
  scale.tare();

  Serial.print("read (calibrated):");
  Serial.println(scale.get_units(10));
   
  server.on("/", [](){ getLocalFile("/index.html", "text/html"); });
  server.on("/weight", [](){ server.send(200, "text/html", weight); });
  server.on("/tare", [](){ scale.tare(); server.send(200, "text/html", "tare()"); });
  
  server.begin();
}


void loop() {
  ESP.wdtFeed();
  server.handleClient();

  weight = String(scale.get_units(10), 1);
  delay(50);
}
```

### index.html
```html
<head>
    <title>ESP8266 scale</title>
    <script src="https://code.jquery.com/jquery-3.2.1.min.js"></script>    
    <script src="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/2.7.1/Chart.bundle.js"></script>
</head>

<body>
    <div id="canvas-holder" style="width:65%; display:inline-block">
        <canvas id="chart-area" />
    </div>

    <div id="weight-label" style="display:inline">test</div>
    <script>
        
    var targetWeight = 300.0; //量り取りたい重さを300グラムとする

    var pieWeightData = function(w) {
        var weight = parseFloat(w);
        var excessWeight, currentWeight, shortageWeight;

        if (weight > targetWeight*2) {
            excessWeight = targetWeight;
            currentWeight = 0;
            shortageWeight = 0;
        } else if (weight > targetWeight) {
            excessWeight = weight - targetWeight;
            currentWeight = targetWeight - excessWeight;
            shortageWeight = 0;
        } else {
            excessWeight = 0;
            currentWeight = weight;
            shortageWeight = targetWeight - weight;
        }

        return [excessWeight, currentWeight, shortageWeight];
    }
    
    var config = {
        type: 'pie',
        data: {
            datasets: [{
                data: [ 0, 0, 0 ],
                backgroundColor: [
                    'rgb(255, 99, 132)',    //過剰分用の黄色パート
                    'rgb(255, 205, 86)',    //現在の重量用の赤色パート
                    'rgb(255,255,255)',     //不足分用の白色パート
                ],
            }],
        },
        options: {
            animation: {
                duration: 500
            },
            events: [], //eventsを何も指定しないことで、マウスオーバーイベントなくす
            responsive: true
        }
    };



    window.onload = function() {
        var ctx = document.getElementById("chart-area").getContext("2d");
        window.myPie = new Chart(ctx, config);

        var timer = setInterval(function() {
        $.get("/weight", function(w){
                $("#weight-label").text(w);
                config.data.datasets[0].data = pieWeightData(w);
                window.myPie.update()
                
            });
        }, 500);
    };



    </script>
</body>

</html>
```
