---
layout: post
title: ESP-IDF ESP32でのBTstackの導入とHID-over-GATTの利用
outline: ESP32/BTstackを使ってHID-over-GATTマウスを動かしてみました。
categories: [ESP32]
---

こんばんは、kat-kaiです。今回はBTstackのHID-over-GATTによるマウスサンプルを使ってみます。

__追記 (2017/11/21):__ BTstackを導入せずに、ESP-IDFでBluetooth HIDとして  
動く[デモコードが公開](https://github.com/espressif/esp-idf/issues/782#issuecomment-342410213)されており、その[紹介記事](https://kat-kai.github.io/ESP-IDF%E3%81%A8ESP32%E3%81%A7Bluetooth-HID/)を書きました。

### BTstackとは
[BTstack](https://github.com/bluekitchen/btstack)とは[BlueKitchen’s](http://bluekitchen-gmbh.com/)によって実装されているBluetoothスタックです。

非営利的な利用であれば無料で使えるため、趣味の電子工作として使う分には適していますね。

### BTstackの導入
それでは、まずセットアップ済みのESP-IDFにBTstack developブランチを導入してみます。

[BTstack Port for the Espressif ESP32 Platform](https://github.com/bluekitchen/btstack/tree/develop/port/esp32)に書かれている導入の方法は、

```btstack/port/esp32/ $ ./integrate_btstack.sh```を実行すれば良いみたいです。

しかしWindows MinGW32に導入する場合、以下の2つのファイルの修正が必要でした。
- btstack/tool/compile_gatt.py
- btstack/port/esp32/integrate_btstack.sh


それでは実際に導入してみます。まず最初にBTstack developブランチをダウンロードします。

```
$ git clone -b develop https://github.com/bluekitchen/btstack.git
$ cd btstack/tool
```

次にbtstack/tool/compile_gatt.pyを修正します。707行目付近の```btstack_root = ...```を以下のように変更します。

```
    #btstack_root = os.path.abspath(os.path.dirname(sys.argv[0]) + '/..')
    btstack_root = os.path.abspath(os.path.dirname(os.path.abspath(sys.argv[0])) + '/..') #こちらに変更
    gen_path = btstack_root + '/src/bluetooth_gatt.h'
```


そしてbtstack/port/esp32/integrate_btstack.shの修正を行います。

integrate_btstack.shではBTstackのライブラリを```${IDF_PATH}/components```にコピーするためにrsyncが使われています。しかしながらEspressifが用意しているMSYSにはrsyncが含まれておりません。

一応、MinGW32にrsyncは導入できますが今回はcpでコピーします。もしrsyncをする場合、以下のページを参考に導入することができます。ただしESP-IDFの環境では、さらに追加でmsys-1.0.dllが必要でした。

[rsyncをGit for Windowsに混ぜる](https://hail2u.net/blog/software/install-rsync-to-git-for-windows.html)
- rsync.exe
- msys-iconv-2.dll
- msys-intl-8.dll
- msys-popt-0.dll
- [msys-1.0.dll](https://sourceforge.net/projects/mingw/files/MSYS/Base/msys-core/msys-1.0.19-1/) (ESP-IDFでは必要)



```
$ cd ../port/esp32
$ sed -i.bak -e 's/rsync/cp/g' integrate_btstack.sh
$ ./integrate_btstack.sh
```

これでBTstackのexampleを使う準備が整いました。以下のような感じで```port/esp32/example```にサンプルコードが用意されていきます。

![btstack_console](https://kat-kai.github.io/images/2017-10-04_btstack_console.png){:style="display:block;margin-left:auto;margin-right:auto;"}

#### HID-over-GATT(HOG)のマウスを使ってみる

次にhog_mouse_exampleを```make flash```してみます。

```
$ cd example/hog_mouse_demo
$ make menuconfig flash monitor
```

menuconfigで```Serial flasher config```の```Default serial port```を指定して保存、ビルド、ESP32に書き込みをします。そしてリセットしてESP32を起動します。

そうすると、Bluetoothのデバイス名HID Mouseとして認識されます。接続時には、パスコードを要求されるので、コンソールのログの少し上にある```Display Passkey: 123456```のように記載されているパスコードを入力することで接続できます。

#### おわりに
BTStack導入に少し手こずりましたが、簡単にHID-over-GATTを使ったマウスを試すことができました！

ESP32でBLEが使えるようになると、電子工作の幅が広がりそうで楽しみです！ではまた！
