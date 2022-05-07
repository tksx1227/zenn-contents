---
title: "Debian11で簡易的なDNSサーバーを立てる"
emoji: "🖥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Debian", "DNS", "Docker"]
published: false
---

# はじめに
最近DNSサーバーを触る機会があったので、その復習も兼ねて、ローカル上で簡易的なDNSサーバーを立てる手順を解説してみます。
# 開発環境
DNSサーバー用のOSにはDebian11を使用します（特に深い意味はないです）。
また、仮想環境はDocker上に立てていきます。

必要なパッケージ等を記述したDockerfileはこちらになります。

```Dockerfile
FROM debian:11
RUN apt update && apt install \
    systemctl \
    bind9 \
    dnsutils
```

簡単にそれぞれ説明すると

* `systemctl`: システムのサービスを管理・設定するためのソフトウェア
* `bind9`: DNS本体
* `dnsutils`: `nslookup` や `dig` などが入ったパッケージ

# 構成
今回扱うDNSサーバーでは、ドメイン名を「tksx1227.dns-server.com」とし、その解決先となるIPアドレスをこちらの記事のアドレスに設定していこうと思います。
また、DNSサーバーの確認には、`nslookup` コマンドとwebブラウザを使用していこうと思います。

![](/images/DNS_server/img1.png)

# 各種設定
設定ファイルに変更を加える前にDNSサーバーを停止しておきましょう。
`named` はDNSサーバーのことです。

```bash
$ systemctl stop named
```

DNSの設定ファイルはデフォルトで `/etc/bind/named.conf` が読み込まれるようになっているので、基本的にはこちらに設定を記述していく形になります。

`named.conf` はいくつかのステートメントを使用できますが、今回は必要最小限の `options`, `zone` だけを使用します。

* `options`: DNS本体の設定を記述する
* `zone`: ゾーンに関する設定を記述する

※ ゾーンとはDNSが管理する範囲のことを言います。
ここでは ゾーン＝ドメイン と考えても差し支えないでしょう。

設定ファイルのイメージです。

```:named.conf.sample
options {

};

zone <zone name> {

};
```

`options` ステートメントで設定した項目は全てのゾーンへ反映されるので、共通の設定などをここに記述することで冗長さを削減することができます。

## DNS本体の設定
最初に `/etc/bind/named.conf` を確認してみると、ここには設定は記述されておらず、外部のファイルを読み込む形になっていることがわかります。

また、ゾーンを追加する場合は `/etc/bind/named.conf.local` に追記してくれ、とあるのでそちらに従っていこうと思います。

```:named.conf
// This is the primary configuration file for the BIND DNS server named.
//
// Please read /usr/share/doc/bind9/README.Debian.gz for information on the
// structure of BIND configuration files in Debian, *BEFORE* you customize
// this configuration file.
//
// If you are just adding zones, please do that in /etc/bind/named.conf.local

include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
include "/etc/bind/named.conf.default-zones";
```

次に `named.conf` でインクルードされている `named.conf.options` を確認してみます。

ここには、`options` ステートメントのみ記述されており、DNS本体の設定をしているファイルだということがわかります。

早速、こちらに必要な項目を追記していきましょう。

```diff:named.conf.options
 options {
         directory "/var/cache/bind";
 
         // If there is a firewall between you and nameservers you want
         // to talk to, you may need to fix the firewall to allow multiple
         // ports to talk.  See http://www.kb.cert.org/vuls/id/800113
 
         // If your ISP provided one or more IP addresses for stable
         // nameservers, you probably want to use them as forwarders.
         // Uncomment the following block, and insert the addresses replacing
         // the all-0's placeholder.
 
         // forwarders {
         //      0.0.0.0;
         // };
 
         //========================================================================
         // If BIND logs error messages about the root key being expired,
         // you will need to update your keys.  See https://www.isc.org/bind-keys
         //========================================================================
         dnssec-validation auto;
 
-        listen-on-v6 { any; };
+        listen-on-v6 { none; };
+        listen-on port 53 { localhost; };
+        allow-query { localhost; };
 };
```

オプションがいくつかあるので、必要に応じて追加で設定します。
これ以外にもまだまだオプションは存在するので、気になった方は調べてみてください。

|  オプション  |  説明  |
| ---- | ---- |
|  directory  |  ゾーンファイル等を置くディレクトリ  |
|  listen-on [port <port_num>]  |  問い合わせを受け付けるインターフェース<br>DNSサーバーが静的IPアドレスを持つ場合はそのアドレスも追記するといいでしょう  |
|  allow-query  |  問い合わせを許可するホスト、ネットワーク  |
|  blackhole  |  問い合わせを許可しないホスト、ネットワーク  |
|  recursion  |  再帰的に問い合わせをするかどうか  |
|  allow-recursion  |  再帰的な問い合わせを許可するホスト、ネットワーク  |
|  forwarders  |  解決できない名前を転送するDNSサーバー  |

これでローカルホストからのみ問い合わせを受け付けるという設定が完了しました。

再帰問い合わせを有効化することで、自前DNSサーバーで解決できない名前は既存のDNSサーバーに問い合わせるといった処理が設定できますが、今回は特に必要ないのでこのままでいきます。

以下のコマンドで設定ファイルの構文チェックができるので、適宜確認することをおすすめします。
```bash
$ named-checkconf [named.conf's path]
```

## ゾーンの設定
DNS本体の設定が終わったので、ここからはゾーンの設定をしていきます。

ゾーンの設定は、`named.conf` の `zone` ステートメントの記述と、そのゾーン固有の設定ファイルの２つを編集する必要があります。

`localhost` の例が `named.conf.default-zones` に記述されているので、これを真似て `named.conf.local` に追記していきます。

```:named.conf.default-zones
// prime the server with knowledge of the root servers
zone "." {
        type hint;
        file "/usr/share/dns/root.hints";
};

// be authoritative for the localhost forward and reverse zones, and for
// broadcast zones as per RFC 1912

zone "localhost" {
        type master;
        file "/etc/bind/db.local";
};

...

```

`file` にはゾーン固有の設定ファイルのパスを指定します。
これは絶対パスでもいいのですが、`options` ステートメントで指定したディレクトリ配下にあるファイルの場合は、ファイル名だけでも大丈夫です。
```:named.conf.local
zone "tksx1227.dns-server.com" {
        type master;
        file "tksx1227.dns-server.com"
};
```

`type` には以下の３種類がありますが、基本的には `master` で大丈夫でしょう。

|  type  |  説明  |
| ---- | ---- |
|  hint  |  DNSのルートゾーンを表す  |
|  master  |  ネームサーバがこのゾーンのプライマリDNSサーバであることを定義する  |
|  slave  |  ネームサーバがこのゾーンのセカンダリDNSサーバであることを定義する  |


では、`file` で指定した `tksx1227.dns-server.com` というファイルを `/var/cache/bind/` に作成していきます。

```:tksx1227.dns-server.com
$TTL 86400
@     IN      SOA     tksx1227.dns-server.com. sample.dns-server.com. (
                  2          ; Serial
                  10800      ; refresh
                  600        ; retry
                  2419200    ; expire
                  86400 )    ; Negative

@       IN      NS      tksx1227.dns-server.com.
@       IN      A       142.250.207.3
```

ゾーンファイルは、レコードを複数行並べることで構成されています。

上の例では、途中改行を入れていますが、＠マークから始まるブロックが１行にあたるので、合計３行のレコードが記述されていることになります。

`;` に続く文字列はコメントです。

各レコードは次のフォーマットに沿って記述していきます。

```
[ラベル] [TTL] [クラス] [タイプ] [データ]
```

|  項目  |  説明  |
| ---- | ---- |
|  ラベル  |  ドメインの名前  |
|  TTL  |  キャッシュの生存時間  |
|  クラス  |  レコードのクラス<br>現在は IN 以外を使うことは一切無いと考えてOK  |
|  タイプ  |  このレコードがなんの設定を行うのかを表す  |
|  データ  |  実際の設定データを指定する<br>タイプによって省略可能  |


# 動作確認
動作確認には `nslookup` コマンドを使用します。

`nslookup` は以下のようにして使用できます。

DNSサーバーを指定しない場合、既存のDNSサーバーに問い合わせに行ってしまうため、こちらは `localhost` を指定するようにしましょう。

```bash
$ nslookup <IP Address or Domain Name> [DNS Server]
```

では早速確認してみましょう。

```bash
$ nslookup tksx1227.dns-server.com localhost
$ nslookup xxxx.xxxx.xxxx.xxxx localhost
```

これで無事名前解決が実行できることが確認できました。

# おわりに

# 参考にした記事
@[card](https://nanbu.marune205.net/2021/12/debian-dns-bind.html?m=1)
@[card](https://digitalmania.info/linux/debian-11/build-dns-server/)
@[card](http://park12.wakwak.com/~eslab/pcmemo/linux/bind/bind2.html)
@[card](https://wa3.i-3-i.info/word12285.html)

