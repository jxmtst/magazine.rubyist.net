---
layout: post
title: RubyKaigi 2024 のトーク "Adding Security to Microcontroller Ruby" の日本語解説記事
short_title: Adding Security to Microcontroller Ruby Ja
tags: 0064
post_author: 梶原 龍
created_on: 2024-05-25
---

{% include base.html %}

## はじめに

本記事は発表のスライドに沿って解説していくスタイルを取っています。スライドは以下の URL にありますのでこちらを開きながら読んでください。

[https://speakerdeck.com/sylph01/adding-security-to-microcontroller-ruby](https://speakerdeck.com/sylph01/adding-security-to-microcontroller-ruby)

## 自己紹介 (p.3-5)

梶原 龍 (かじわら-りょう) といいます。インターネットでは sylph01 (しるふ-ぜろいち; だいたい数字の部分が省略して呼ばれがち) と名乗っています。デジタルアイデンティティとセキュリティにフォーカスしたフリーランスの Web プログラマをしております。また、W3C や IETF などの標準化団体にてインターネット標準の編集や実装に携わっていました/います。

趣味では音楽ゲーム (特に DDR) をしたり、オーケストラでファゴットを吹いたり、鉄道に乗ったり、キーボードを作ったりしています。

## 発表に至るまでの経緯 (p.7-16)

暗号技術やそれを利用したプロトコルに関心があり、Ruby に足りていないものの実装を行っていました。RubyKaigi 2024 に先立って開催された RubyConf Taiwan 2023 にて Hybrid Public Key Encryption の実装の話をしました。ここでは Ruby の OpenSSL gem を利用する形での gem 実装と、OpenSSL 3.2 以降についている HPKE の API を利用して Ruby の OpenSSL gem そのものを拡張する形での実装の両方を紹介しました。同じく台湾で登壇されていた PicoRuby の開発者の羽角さんと話しているうちに PicoRuby の暗号機能の話になり、PicoRuby に暗号機能を追加することで TLS で通信できるようになるのではないか？　ということで、その実装を行うことにしました。本発表は PicoRuby における暗号機能と、これまで足りていなかった WiFi を利用したネットワーク機能の実装を扱います。MicroPython にはあるので PicoRuby でできないハズはない...！

なおタイトルは[ASMR](https://ja.wikipedia.org/wiki/ASMR) との掛詞なので、実際は「セキュリティの追加」というよりは「ネットワーキングの追加」のほうが正しいと思われますが、スライドにネタを入れてしまう習性なので...。

対象の環境は Raspberry Pi Pico W + PicoRuby/R2P2 です。Raspberry Pi Pico W は RP2040 というマイコンを搭載しており、これは 133MHz で動作するデュアルコアの ARM Cortex-M0+を持ち、264KB の SRAM と 2MB のフラッシュを持ちます。通常のデスクトップに比べて極めて性能の制限が強いことがわかると思います。Pico W は Pico に加えて CYW43439 チップによる Bluetooth とワイヤレス LAN をサポートするもので、技適認証を通っているので日本でも安心して利用できます。会場付近に「電波法を守ろう」とか「技適マークついてますか？」という広告がたくさんあってギョッとしました。そしてお値段は 4 月時点で 1353 円、発表時点でのレートで 10 ドルを切ります。

## 注意事項 (p.17-18)

暗号 API は誤って用いると簡単にセキュリティを損ないます。ご家庭で真似する分には大丈夫ですが、Production で利用する際には専門家のレビューを得てください。

PicoRuby/R2P2 の master ブランチに入っているものは production-ready であると思いますが、まだ入っていないネットワーク機能に関してはかなり試験的な荒削りな実装になっています。

また、今回私は Pico W を「普通のコンピュータ」として扱ってプログラミングしているため、組み込み固有のバグみたいなものを踏んでいるかもしれません。マイコンを「普通のコンピュータ」として扱えることは全く自明ではなく、これは Pico SDK や PicoRuby/R2P2 によるサポートが非常に強力なためできているといっても過言ではありません。

## Part 1: PicoRuby における暗号 (p.20-37)

21 ページから 23 ページでは CRuby において[SHA256](https://ja.wikipedia.org/wiki/SHA-2) ( ハッシュ関数 ) や[AES](https://ja.wikipedia.org/wiki/Advanced_Encryption_Standard) ( 共通鍵暗号 ) をどのように使えるかを記述しています。CRuby においては暗号機能を利用する場合 OpenSSL gem を使うのが一般的です。では OpenSSL を組み込み環境で利用できるかというと必ずしもそのようには行きません。なぜなら OpenSSL 全体を組み込むとサイズが非常に大きくなってしまうためです。組み込み用の暗号ライブラリ、例えば Mbed TLS や wolfSSL などでは、必要な機能のみを指定してビルドするということができ、最終的なバイナリサイズを小さく抑えることができるようになっています。Pico SDK においては Mbed TLS が使われているので、ここでは Mbed TLS のラッパーを実装していきました。

暗号機能を実装するにあたっては暗号化やハッシュ化の結果であるバイナリ文字列の中身を人間に可読な形で確認したいです。しかし PicoRuby はサイズの制限が厳しいため人間にやさしい機能をいちいち実装しているスペースの余裕がなく、Base16 や Base64 はもともと用意されていませんでした。そこで最初に取り掛かったのが PicoRuby で動作する [Base16](https://github.com/picoruby/picoruby/tree/master/mrbgems/picoruby-base16) と [Base64](https://github.com/picoruby/picoruby/tree/master/mrbgems/picoruby-base64) の mrbgem の実装です。これらの gem は単機能の実装で非常にコンパクトな実装になっているため、PicoRuby において C 拡張を持つ mrbgem を実装したい人にとってわかりやすい入口になると思われます。

実際に実装できた AES と SHA256 の PicoRuby での利用例が 26 ページと 27 ページにあります。PicoRuby にはもともと Mbed TLS を利用した CMAC の実装があり、これをベースに最もよく利用される共通鍵暗号アルゴリズムの AES とその中で最も利用される[暗号利用モード](https://ja.wikipedia.org/wiki/%E6%9A%97%E5%8F%B7%E5%88%A9%E7%94%A8%E3%83%A2%E3%83%BC%E3%83%89) の CBC と GCM、最も利用されていて現在安全であると知られている SHA256 の実装を追加しました。SHA-1 や MD5 を足していないのはこれらのアルゴリズムは危殆化したため使ってほしくないからです。[最終的に Mbed TLS mrbgem がどうなったかはこちらから見れます。](https://github.com/picoruby/picoruby/tree/master/mrbgems/picoruby-mbedtls)

30 ページから先は CRuby と mruby/c における C 拡張の実装の比較を行っています。C ライブラリのラッパーを書く場合 C の値を Ruby オブジェクトに包みそれをインスタンスで引き回すということをよく行います。mruby/c では `mrbc_instance_new()` 関数でインスタンスを作成する際に関連づける値のサイズを指定して領域を確保し、インスタンスの `data` 要素でこの領域を利用します。また、CRuby の `rb_define_method` で定義するメソッドは指定した関数ポインタで示される関数の引数から直接メソッドの引数を取得できるのに対して、mruby/c の `mrbc_define_method` で指定する関数ポインタで示される関数はシグネチャが決まっており可変長の引数を扱えるようになっていません。mruby/c でメソッドの引数を取得するには `GET_ARG()` マクロで取得することになります。

(補足: スライドでは CRuby における C の値のラッピングについてかなり大雑把に説明していますが、実際はマクロ 1 個で済むわけではありません。CRuby における C の値のラッピングの詳細はこの発表の範囲を飛び越えるので、[ruby/ruby の extension に関するドキュメント](https://github.com/ruby/ruby/blob/master/doc/extension.ja.rdoc#c%E3%81%AE%E3%83%87%E3%83%BC%E3%82%BF%E3%82%92ruby%E3%82%AA%E3%83%96%E3%82%B8%E3%82%A7%E3%82%AF%E3%83%88%E3%81%AB%E3%81%99%E3%82%8B-) や[The Definitive Guide to Ruby's C API の第 10 章](https://silverhammermba.github.io/emberb/c/#data) を参照してください。)

この情報をもとに、32 ページと 33 ページの例を見てみましょう。

32 ページではハッシュ (Digest) のインスタンスを生成する例を示しています。アルゴリズム ID を `GET_ARG(1)` で取得し、インスタンスの生成を `mrbc_instance_new` で行っています。ここで確保する領域は Mbed TLS におけるメッセージダイジェストの状態を保持する値の型である `mbedtls_md_context_t` のサイズです。確保した領域 `self.instance->data` へのポインタを取得し、それに対して `mbedtls_md_init()` を呼ぶことで、インスタンスに C の値を関連づけています。

33 ページでは Digest のインスタンスメソッドである update の実装を示しています。入力文字列を `GET_ARG(1)` で取得し、 `v->instance->data` でインスタンスに関連付けられた状態へのポインタを取得し、これらを利用して Mbed TLS の関数 `mbedtls_md_update()` を呼んでいます。また、関数末尾にて `mrbc_incref()` をインスタンスに対して呼ぶことでこのインスタンスの参照カウントを追加し、関数の終了時に deallocate されることを回避しています。これを呼ばないとこのメソッドの末尾で該当のインスタンスは寿命を終えたものとして deallocate されるため、次にインスタンスメソッドを呼んだ際には segmentation fault となってしまいます。

今回 PicoRuby の暗号機能を実装するにあたって、複数回 `update` を呼ぶ形の API しか実装しておらず、バッファをひとまとめに暗号化する API(one-shot API) を実装していません。これは one-shot API は暗号化対象のバッファと同じサイズの書き込み先バッファが必要であるため、メモリが小さい環境では小さいバッファに対して処理を繰り返し処理が終わった領域を解放していくことができる形の API のほうが有利であると考えられるからです。

また、多くの暗号処理においては乱数が必要ですが、パソコンと違って乱数生成器が事前に用意されておらず、ハードウェアの機能を使って自分で実装する必要があります。Raspberry Pi Pico には Ring Oscillator という NOT ゲートを輪っかのようにつないだものがあり、呼び出すタイミングによって 0 と 1 のどちらかが得られる、というものがあります。これを C で利用できるようにしたのが 37 ページの実装です。ここでは単にハードウェアから取り出したビット `rosc_hw->randombit` をそのまま利用しているのではなく、01 を 0 に、10 を 1 に、00 と 11 を捨てるという処理 (von Neumann debiasing) を行うことで、ハードウェアから取り出したビットの偏りを低減させる処理を行っています。

(補足: 0 が得られる確率が p としたとき、1 が得られる確率は (1-p) であるため、01 が得られる確率は p(1-p)、10 が得られる確率も同じく p(1-p)、00 と 11 は捨てるので、結果として 01 と 10 をそれぞれ 0 と 1 にした場合元の値を利用するよりも偏りが低減していることになります)

## Part 2: ネットワーク (p.39-60)

ネットワーク機能の実装とは、具体的に以下のことを目指します:

- [802.11](https://ja.wikipedia.org/wiki/IEEE_802.11) を利用して無線 LAN 接続を行う (レイヤー 2)
- IP アドレスを取得する (レイヤー 3)
- サーバーに対して TCP で接続する (レイヤー 4)
- 可能であれば TLS で暗号化する (レイヤー 5)
- アプリケーション層のプロトコルとして HTTP を利用する (レイヤー 6/7)

それぞれのレイヤーに対して、Pico SDK の対応するものが以下のように存在します:

- CYW43439 のドライバ: レイヤー 2
- lwIP: レイヤー 3 ・ 4
- Mbed TLS: レイヤー 5

また、CYW43439 ドライバと lwIP の連携、lwIP と Mbed TLS の連携が Pico SDK で用意されています。HTTP は自分で実装することになります。よって、今回実装したのは、CYW43439 ドライバの WiFi 関連機能の Ruby インターフェース、lwIP の Ruby インターフェースと簡易な HTTP ライブラリです。Mbed TLS の機能は lwIP から透過的に利用されるので、実は Part 1 で実装した暗号機能は TLS 接続には使われていません。

R2P2 から Pico SDK に用意されているネットワーキングのライブラリを追加する CMakeLists.txt の記述が 43 ページにあります。CYW43439 ドライバには `pico_cyw43_arch_lwip_poll` と `pico_cyw43_arch_lwip_threadsafe_background` の 2 つのモードがあり、今回は後者を利用しています。前者はメインのプログラムから定期的にドライバをポーリングする必要があるのに対して、後者はその処理をバックグラウンドでやってくれます。リアルタイム OS(RTOS) を利用するモードもあるのですが、WiFi のためだけに RTOS を導入するのはオーバーキルなのでここでは利用していません。

CYW43439 ドライバの機能を使って WiFi アクセスポイントに接続するための Ruby コードが 46 行目に記載されています。

WiFi アクセスポイントにつながるようになったあと、まず lwIP の DNS 機能を利用した名前解決のラッパーを実装しました。lwIP の多くの関数は、ネットワーク処理を実行してデータが準備できた際に呼び出されるコールバックを関数ポインタで設定する形で動作します。48 ページのコードがその実例を示しています。lwIP の `dns_gethostbyname()` 関数は第 3 引数にコールバック関数を設定します。名前解決が終了し IP アドレスが得られたとき、`dns_found` コールバックが呼ばれ、その第 3 引数 `void *arg` に結果の IP アドレスが格納されるので、コールバック関数内で結果を格納したい領域のポインタに対して名前解決の結果をコピーしています。

ところでデモ動画では HTTPS 接続の 1 回目が失敗していました。これは何ででしょう？　というわけで Raspberry Pi を WiFi ホットスポットとして立ち上げ、その間の通信を Wireshark で覗いてみることにしました。設定の方法は[公式のチュートリアルに記載されています](https://www.raspberrypi.com/tutorials/host-a-hotel-wifi-hotspot/)。実際のパケットキャプチャの結果が 50 ページに記載されています。失敗した例では `3.0.0.0` という IP アドレスに対して TCP 接続を試みて失敗していることがわかります。これは DNS の解決結果を格納する IP アドレスの領域が適切に初期化されていないために名前解決の結果を待たずに意図しない IP アドレスに対して TCP 接続をしようとしていた結果でした。適切な初期化を行う修正をしたところ DNS の名前解決を待ってその IP アドレスに対してリクエストを行っていることがわかります。

続いて TCP クライアントの実装です。スライドでは紙面の関係でかなり端折った説明になっていますがここでは[実際のコード](https://github.com/sylph01/picoruby/blob/wlan/mrbgems/picoruby-net/ports/rp2040/net.c) とセットで見ていきましょう。

- Ruby コードに対するインターフェースが[`TCPClient_send()`](https://github.com/sylph01/picoruby/blob/wlan/mrbgems/picoruby-net/ports/rp2040/net.c#L220-L241) です。
  - 223 行目の `ip4_addr_set_zero()` が上記の「適切な初期化を行う修正」に相当します。
  - 230 行目の `TCPClient_connect_impl()` にて TCP 接続を初期化し接続を行います。
    - [`TCPClient_new_connection()`](https://github.com/sylph01/picoruby/blob/wlan/mrbgems/picoruby-net/ports/rp2040/net.c#L127-L141) が TCP 接続を作成している部分です。`altcp_new()` で Protocol Control Block(PCB) を作成し、続く `altcp_recv()`, `altcp_sent()`, `altcp_err()`, `altcp_poll()` で状態に応じて呼ばれるべきコールバックを設定しています。コールバック関数に渡される引数として接続状態を示す値を渡してほしいので `altcp_arg()` にて設定しています。ここで特に重要なのがデータを け取ったときに呼ばれるコールバックを設定する `altcp_recv()` です。
  - `TCPClient_send` 内の while ループで呼んでいる [`TCPClient_poll_impl()`](https://github.com/sylph01/picoruby/blob/wlan/mrbgems/picoruby-net/ports/rp2040/net.c#L177-L218) は TCP 接続の状態に応じて行うべき処理が記述されています。接続が完了した状態 `NET_TCP_STATE_CONNECTED` になったとき、 `altcp_write()` を利用して `send_data` という mruby/c 文字列の中身をネットワークに書き出し、 `altcp_output()` で書き出しの完了を待ちます。
  - `altcp_recv()` で設定した [`TCPClient_recv_cb()`](https://github.com/sylph01/picoruby/blob/wlan/mrbgems/picoruby-net/ports/rp2040/net.c#L75-L97) の中身を見ていきましょう。受信したデータは `pbuf` と呼ばれる構造体に入って渡ってきます。この `pbuf` は linked list のデータ構造を持っているので、82 行目〜86 行目でこのリストを順番にたどることでデータを一時バッファにコピーしています。この一時バッファの中身を `mrbc_string_append_cbuf()` を使って受信データを示す mruby/c の `String` に対して結合してあげることで受信したデータの `String` を生成しています。
  - 空の `pbuf` を受信したり接続が idle になった場合 `NET_TCP_STATE_FINISHED` という状態に移行します。 `TCPClient_poll_impl()` の中で PCB を閉じたり接続状態を管理する変数を解放したりしています。この結果 while ループを脱し、 `recv_data` オブジェクトが準備できるので、それを返り値として返却しています。

TCP クライアントができれば HTTP はそんなに難しくありません。Ruby の文字列で HTTP リクエストを組み立ててあげればよく、それを TCPClient に対して渡せばよいです。最も原始的な HTTP GET のクライアントを 55 ページに記載しています。執筆時点での実装ではこれに加えて、得られたレスポンスからステータスコードとヘッダとボディを分割するところまで実現しています。

TLS は lwIP の Application Layered TCP の機能を使えば少々の書き換えで実現できます。[`TCPClient_new_tls_connection()`](https://github.com/sylph01/picoruby/blob/wlan/mrbgems/picoruby-net/ports/rp2040/net.c#L143-L160) の `TCPClient_new_connection()` に対する差分を説明すると、

- `altcp_tcp_create_client_config()` で設定を用意し
- `altcp_new()` の代わりに `altcp_tls_new()` で PCB を作成し
- `mbedtls_ssl_set_hostname` でホスト名を設定する

だけの差分しかありません。

何が起こっているのかというと、TLS の PCB が渡されたとき、lwIP はデータ送信時には TCP の送信用関数が呼び出されたあとに TLS のコールバックを呼ぶことで暗号化を行い、逆に受信時には TLS のコールバックが先に呼ばれて復号されてから平文が TCP のコールバックに対して渡されることで、透過的に暗号処理が行われるため、平文の場合と TLS の場合でコードを共用することができています。

スライドには「TLS は比較的自明に実装できる」と書いていたのですが、実は TLS を追加したときに突然謎のハングアップに見舞われたため、そんなに自明ではありませんでした。これは lwIP のメモリ管理と mruby/c VM のメモリ管理が独立に動いており、TLS を追加したことでメモリが足りなくなったためでした。そのため PicoRuby 側のヒープメモリサイズを元の 194KB から 96KB と半分近く削っています。また、別のところでは `mrbc_free()` でメモリ解放するべきところを、存在しない `free()` で解放しようとして謎のハングアップで 3 時間ほど失ったこともありました。

## Part 3: 今後の開発について (p.62-65)

今回は TCP クライアントを lwIP の機能を使って愚直に実装しましたが、CRuby では `TCPSocket` や `Socket.tcp` によって BSD Socket API に沿った形の API が提供されています。MicroPython でも同様のソケット API が存在しています。MicroPython も同様に Pico SDK の lwIP を利用しているため、これをポーティングすることで PicoRuby でもソケットのような API でネットワーキングが実現できるかもしれません。

また、クライアントを実装したということはサーバー機能も欲しくなります。ソケット API があれば比較的やりやすいでしょう。実は lwIP の機能をそのまま使う形で実装を進めているのですが、mruby/c のほうに足りないと思われる機能があり Ruby で実現したい API デザインが実現できずに保留しています。

今回実装したネットワーク機能はブロッキング IO を利用していますが、ノンブロッキング IO が必要かといわれると現時点では確証を持てていません。

## まとめ (p.67-79)

ここまでの実装を通して、PicoRuby を使って HTTPS のリクエストを送る機能が実現できました。発表時点では Base16/64、SHA256、AES と乱数生成が PicoRuby/R2P2 の master ブランチに入っており、ネットワーク機能は今後いくつかの修正を経て pull request を出す予定です。

しかし PicoRuby で TLS が本当に必要かと言われるとまた別の問題があります。似たような環境のベンチマーク (STM32L562E, Cortex-M33 @ 110MHz on wolfSSL) では 2048bit の RSA は 1 秒に 0.155 回しか実行できない、つまり 6 秒前後かかる、という結果が得られているように、公開鍵暗号は組み込み機器で実行するには非常に高コストな演算です。また、TLS のルート証明書をインストールしていないため、TLS の暗号化部分を得ることはできますが、接続先が本当に意図した接続先であるかの認証を得ることはできません。そのため、セキュリティ要件によってはゲートウェイとの間をアプリケーションレベルの共通鍵暗号で通信し、ゲートウェイから TLS 接続する、というような構成も考えられるでしょう。この場合共通鍵は Pico の上に存在しているため Pico が物理的に侵害された場合は共通鍵が奪われてしまいますが、ネットワークに接続するための WiFi パスワードも Pico の上に存在しているので、物理的侵害を本気で気にするならばもっと別の方法を使う必要があるでしょう。

もっと普通のコンピュータに近いプログラミング体験が欲しいならば Raspberry Pi Zero 2 W というもう少し強力なハードウェアを使うのも手でしょう。これは Linux を実行でき SSH することもできるので、なんなら PicoRuby すら使う必要もありません。Pico を使うのは組み込みやすいからです。実際の IoT 環境ではこれらの違ったスペックを持つハードウェアは協調して動作させるので、PicoRuby が WiFi に繋がるようになったことで、「Ruby での真の IoT」は現実に近づいたといってよいでしょう。IoT に関するプロトコルは数多く存在しており、これらを PicoRuby で扱えるようにすることで、Ruby での IoT の可能性は広がっていくものと考えられます。このことから、Ruby での IoT は「ブルーオーシャン」であるといえます。コミュニティは皆さんのコントリビュートをお待ちしています。

## 謝辞 (p.80-81)

まずは PicoRuby/R2P2 の作者である羽角さん ([Twitter: @hasumikin](https://twitter.com/hasumikin)) に最大限の感謝をしたいと思います。このプロジェクト自体 PicoRuby/R2P2 なしには全く成立し得ないものです。また、実装の過程で多くのアドバイスをいただきました。

また、本発表は RubyKaigi でネットワークに関するトークを行った先駆者であるうなすけさん ([Twitter: @yu_suke1994](https://twitter.com/yu_suke1994)) としおいさん ([Twitter: @coe401\_](https://twitter.com/coe401_)) に大きなインスピレーションを受けています。今後も Ruby におけるネットワークプロトコル実装をやっていきましょう！　 ( 読者向け: ruby-jp Discord の `#ietf` チャンネルにて IETF Meeting の前に最新の Internet-Draft を読んでいく会をやっていますので興味のある方はぜひご参加ください。IETF 自体膨大なワークを扱っているので Ruby でカバーできる範囲が増えることはコミュニティのためにもなります！　)