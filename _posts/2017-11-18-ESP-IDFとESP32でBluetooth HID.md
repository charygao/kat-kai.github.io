---
layout: post
title: ESP-IDF ESP32でのBluetooth HIDの利用
outline: Espressif社より先行公開されたBluetooth HIDデバイスのデモコードを動かしてみました。
categories: [ESP32]
---

こんばんは、kat-kaiです。

最近、ESP32でBluetooth HIDデバイスのデモコードが公開されましたので、その紹介をしたいと思います。

<blockquote class="twitter-tweet" data-lang="en"><p lang="ja" dir="ltr">ちなみにデモコードは、こちらの下の方<a href="https://t.co/KbFsbbXizZ">https://t.co/KbFsbbXizZ</a><br>まだ詳しく内容見れてません</p>&mdash; kat-kai (@katkai3) <a href="https://twitter.com/katkai3/status/927827470409113605?ref_src=twsrc%5Etfw">November 7, 2017</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>


## ESP32 (ESP-WROOM-32)とは
Espressif Systems社によって開発されたWiFi, Bluetoothが搭載されたワイヤレスモジュールです。  
1500～2000円程度で購入可能な開発ボードを使えば、PCとUSB接続するだけでESP32の開発ができます。
- 秋月電子 [ESP32-DevKitC ESP-WROOM-32開発ボード](http://akizukidenshi.com/catalog/g/gM-11819/)
- スイッチサイエンス [ESPr® Developer 32](https://www.switch-science.com/catalog/3210)

ESP32モジュール単体だと、開発には別途USB-シリアル変換モジュールですが600～700円程度で購入可能です。
- [秋月電子](http://akizukidenshi.com/catalog/g/gM-11647/) 700円
- [スイッチサイエンス](https://www.switch-science.com/catalog/3156/) 700円
- [マルツオンライン](https://www.marutsu.co.jp/pc/i/836865/) 640円

### ESP32の開発環境
ESP32の開発環境として、ESP-IDF, Arduino ESP32, Micropythonなどがあります。

- ESP-IDF
    - 提供されている開発環境の中でもESP32の機能を最大限利用することが出来る
- Arduino ESP32
    - Arduinoライクな開発が可能
    - ESP32用のArduinoライブラリが公開されていて便利
- Micropython
    - Python 3.xで開発できる、らしい。

### ESP-IDFでのBluetooth Low Energy (BLE) HIDデバイスとしての利用について
Bluetooth Low Energy (BLE)はBluetooth 4.0以降のことであり、省電力を目的とした規格です。  
BLEでHIDデバイスとして振る舞うにはHOGP (HID-over-GATT Profile)と呼ばれるプロファイルを使う必要があります。

これまでには、Bluekichen’sによって開発された外部のライブラリBTstackを使うことで  
ライブラリの導入が少し面倒ではありますが、Bluetoothキーボードやマウスとして振る舞うことが出来ていました。  
また過去にはBTstackを用いたHOGPのデモコードについて紹介しています。

- [ESP-IDF ESP32でのBTstackの導入とHID-over-GATTの利用](https://kat-kai.github.io/ESP32%E3%81%A8BTStack%E3%82%92%E4%BD%BF%E3%81%A3%E3%81%A6HID-over-GATT/)



今回、Espressif社より公式(現時点ではまだ非公式かも？)のデモが公開されました。  
本デモコードでは、ライブラリを新たに導入する必要がないため、比較的容易に開発を始めることが出来ます。
  
- Bluetooth Low Energy (BLE)によるHID-over-GATT Profileのデモコード    
[[TW#13919] Bluetooth HID implementation progress?](https://github.com/espressif/esp-idf/issues/782#issuecomment-342410213)

ただ問題点として、人によってはパソコン側で認識できないこともあるそうなので  
もし今回のデモコードの動作が不安定そうならBTstackを試してみるのも良いかもしれません。


### 今回のデモコード
今回のデモコードは、ホスト側のスマートフォンのボリュームを上げ下げするコードです。  
まずESP32にデモコードを書き込み、起動してみるとAndroid側からはデバイス名"HID"として表示されます。

![btstack_console](https://user-images.githubusercontent.com/21113258/32978230-4a273414-cc81-11e7-9640-5dd7e3919377.png){:style="display:block;margin-left:auto;margin-right:auto;"}

これをペアリングすると、ボリュームが上がって、下がってを繰り返します。


それではデモコード中のキー入力を操作する箇所を見てみます。  
実際のキー入力の制御は```ble_hidd_demo_main.c``` 233行目の```hid_demo_task```関数で行われています。

```
void hid_demo_task(void *pvParameters)
{
    vTaskDelay(1000 / portTICK_PERIOD_MS);
    while(1) {
        vTaskDelay(2000 / portTICK_PERIOD_MS);
        if (sec_conn) {
            LOG_ERROR("Send the volume");
            send_volum_up = true;
            //uint8_t key_vaule = {HID_KEY_A};
            //esp_hidd_send_keyboard_value(hid_conn_id, 0, &key_vaule, 1);
            esp_hidd_send_consumer_value(hid_conn_id, HID_CONSUMER_VOLUME_UP, true);
            vTaskDelay(3000 / portTICK_PERIOD_MS);
            if (send_volum_up) {
                send_volum_up = false;
                esp_hidd_send_consumer_value(hid_conn_id, HID_CONSUMER_VOLUME_UP, false);
                esp_hidd_send_consumer_value(hid_conn_id, HID_CONSUMER_VOLUME_DOWN, true);
            }
        }
    }
}
```

```esp_hidd_send_consumer_value```関数によって音量を上げ下げしています。  
ここではコメントアウトされていますが、  
```esp_hidd_send_keyboard_value```関数を用いることでキーボード入力できることも分かります。

また```HID_KEY_A```のようなキーコードの詳細は```hid_dev.h```に記載されています。

今回のデモコードでは、ある一定の間隔で単にボリュームを上下させるだけでしたが、  
キースイッチを接続したESP32でGPIO割り込み処理をすることで
キーボードを作製することも出来そうですね！

### おわりに
実は今回紹介したデモコードは、次回アップデートで導入される予定だったようです。ただ需要があったようで、本デモコードのみ先行公開されています。  

今回のデモコードを参考にすることで、ESP32で自作Bluetoothキーボードやマウスを作ることが出来ますね！  
容易にHIDデバイスを自作出来るようになると、色々と夢が広がって楽しいです！それでは！
