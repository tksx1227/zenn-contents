---
title: "Debian11で簡易的なDNSサーバーを立てる"
emoji: "🖥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Debian", "DNS", "Docker"]
published: true
---

# はじめに
最近DNSサーバーを触る機会があったので、その復習も兼ねて、ローカル上で簡易的なDNSサーバーを立てる手順を解説してみます。

# 開発環境
DNSサーバー用のOSにはDebian11を使用します（特に深い意味はないです）。

また、仮想環境はDocker上に立てていきます。

必要なパッケージ等を記述したDockerfileはこちらになります。

```Dockerfile
FROM debian:11
RUN apt update && apt install -y \
    systemctl \
    bind9 \
    dnsutils
```

簡単にそれぞれ説明すると

* `systemctl`: システムのサービスを管理・設定するためのソフトウェア
* `bind9`: DNS本体
* `dnsutils`: `nslookup` や `dig` などが入ったパッケージ

# 構成
今回構築するDNSサーバーでは、ドメイン名「tksx1227.dns-server.com」を、「192.168.200.42」に解決するという処理を行えるように設定していこうと思います。

当然、逆引きにも対応させます。

![参考画像](/images/DNS_server/img1.png)

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

:::message
ゾーンとはDNSが管理する範囲のことを言います。
ここでは ゾーン＝ドメイン と考えても差し支えないでしょう。
:::

設定ファイルのイメージです。

```:named.conf.sample
options {

};

zone <Zone Name> {

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

再帰問い合わせを有効化することで、自前DNSサーバーで解決できない名前は既存のDNSサーバーに問い合わせるといった設定ができますが、今回は特に必要ないのでこのままでいきます。

以下のコマンドで設定ファイルの構文チェックができるので、適宜確認することをおすすめします。

```bash
$ named-checkconf [named.conf's path]
```

## ゾーンの設定
DNS本体の設定が終わったので、ここからはゾーンの設定をしていきます。

ゾーンの設定は、`named.conf` の `zone` ステートメントの記述と、そのゾーン固有の設定ファイルの２つを編集する必要があります。

早速 `named.conf.local` に追記していきます。

下の例では、１つ目が正引き、２つ目が逆引きのゾーン設定となっています。

```:named.conf.local
zone "tksx1227.dns-server.com" {
        type master;
        file "tksx1227.dns-server.com";
};

zone "200.168.192.in-addr.arpa." {
        type master;
        file "tksx1227.dns-server.com.rev";
};
```

:::message
逆引きは名前空間の都合よりIPアドレスを逆順に記述していきます。
`.in-addr.arpa.` は名前空間の上位層にあたります。
:::

上で指定したアドレスは 192.168.200.xxx というアドレスに対するゾーン設定ということですね。
もちろん、この時点で 192.168.200.42 ただ一つに対するゾーンの設定もできるのですが、範囲で設定することの方が多いようなので、そちらを真似ています。

`file` にはゾーン固有の設定ファイルのパスを指定します。

これは絶対パスでもいいのですが、`options` ステートメントで指定したディレクトリ配下にあるファイルの場合は、ファイル名だけでも大丈夫です。

`type` には以下の３種類がありますが、基本的には `master` で大丈夫でしょう。

|  type  |  説明  |
| ---- | ---- |
|  hint  |  DNSのルートゾーンを表す  |
|  master  |  ネームサーバがこのゾーンの権威DNSサーバであることを定義する  |
|  slave  |  ネームサーバがこのゾーンのセカンダリDNSサーバであることを定義する  |

:::message
権威DNSサーバーとは、実際に該当ゾーンの情報を管理しているDNSサーバーのことです。
外部のDNSサーバーに委託せずに自分自身で名前解決を行うことができます。
:::

では、`file` で指定した `tksx1227.dns-server.com`, `tksx1227.dns-server.com.rev` というファイルを `/var/cache/bind/` に作成していきます。

```:tksx1227.dns-server.com
tksx1227.dns-server.com.      86400      IN      SOA      localhost. sample.mail.com. (
               2022050800   ; Serial
               10800        ; Refresh
               600          ; Retry
               2419200      ; Expire
               86400 )      ; Minimum

tksx1227.dns-server.com.      86400      IN      NS      localhost.
tksx1227.dns-server.com.      86400      IN      A       192.168.200.42
```

```:tksx1227.dns-server.com.rev
200.168.192.in-addr.arpa.      86400      IN      SOA      localhost. sample.mail.com. (
               2022050800   ; Serial
               10800        ; Refresh
               600          ; Retry
               2419200      ; Expire
               86400 )      ; Minimum

200.168.192.in-addr.arpa.      86400      IN      NS      localhost.
42                             86400      IN      PTR     tksx1227.dns-server.com.
```

ゾーンファイルは、DNSレコードを複数行並べることで構成されています。

上の例では、可読性の観点から途中改行を入れていますが、それぞれ３つのレコードで構成されていることになります。

`;` に続く文字列はコメントです。

各レコードは次のフォーマットに沿って記述していきます。

```
[ラベル] [TTL] [クラス] [タイプ] [データ]
```

|  項目  |  説明  |
| ---- | ---- |
|  ラベル  |  ラベルの名前  |
|  TTL  |  キャッシュの生存時間  |
|  クラス  |  DNSレコードのクラス<br>現在は IN（Internetの略らしい） 以外を使うことは一切無いと考えてOK  |
|  タイプ  |  DNSレコードの種類を表す  |
|  データ  |  実際の設定データを指定する<br>タイプによって書き方が変わる  |

上の例はかなり冗長に記述したものであるため、少し簡略化してみましょう。

ラベルは `@` で置き換えることができ、`@` を指定すると、`named.conf` のゾーンステートメントで指定した名前が使用されます。
今回の例だと `@` は `tksx1227.dns-server.com` や `200.168.192.in-addr.arpa.` として解釈されます。

また、２つ目のレコード以降はラベルを省略することができ、省略した場合は１つ上のレコードと同じラベルとして解釈されます。

TTLはファイルの頭で一括で指定することもでき、次のように記述することで各レコードのTTLも省略することができます。

```
$TTL 86400
```

上記を取り入れてゾーンファイルを書き直すと多少スッキリと記述できます。

```:tksx1227.dns-server.com
$TTL 86400
@      IN      SOA      localhost. sample.mail.com. (
               2022050800   ; Serial
               10800        ; Refresh
               600          ; Retry
               2419200      ; Expire
               86400 )      ; Minimum

       IN      NS       localhost.
       IN      A        192.168.200.42
```

```:tksx1227.dns-server.com.rev
$TTL 86400
@      IN      SOA      localhost. sample.mail.com. (
               2022050800   ; Serial
               10800        ; Refresh
               600          ; Retry
               2419200      ; Expire
               86400 )      ; Minimum

       IN      NS       localhost.
42     IN      PTR      tksx1227.dns-server.com.
```

:::message alert
DNSサーバー名やメールアドレスの末尾には `.` を付ける必要があるので、忘れないようにしましょう。
`.` を忘れても構文エラーにはなりませんが、別の文字列として解釈されるためデバッグが大変になります。
:::

では各レコードの役割を簡単に説明していきます。

### SOAレコード
SOAレコードは、該当するゾーンの情報を記述するレコードになります。

データのフィールドには、DNSサーバー名と管理者のメールアドレス、DNSサーバーの各種設定の３項目を記述します。

:::message
ゾーンファイルはSOAレコードを一番最初に記述する必要があります。
:::

DNSサーバー名は、このゾーンを管理するDNSサーバーの名前を記述する必要があるので、今回は `localhost` となります。

DNSサーバーにドメインを振っている場合は、`xxx.my-dns-server.jp.` などの独自の値を記述してください。

メールアドレスは、DNSの管理者のメールアドレスを入力する必要があります。
 今回は適当な連絡先を指定します。

:::message
メールアドレスは `@` を `.` で置き換える必要があるので注意してください。
:::

DNSサーバーの各種設定項目は以下の通りです。

|  項目  |  説明  |
| ---- | ---- |
|  Serial  |  適当なシリアルナンバー  |
|  Refresh  |  セカンダリサーバーへの転送間隔  |
|  Retry  |  リフレッシュ失敗時の再送間隔  |
|  Expire  |  リフレッシュ失敗時のセカンダリサーバの有効期間  |
|  Minimum  |  存在しないドメインのキャッシュを保持する期間  |

それぞれ適当な値を設定します。

### NSレコード
NSレコードは、該当するゾーンを管理しているDNSサーバーの名前を記述するレコードになります。

こちらはデータフィールドにDNSサーバーの名前だけを記述すればOKです。

今回は、`tksx1227.dns-server.com` というゾーンを `localhost` で管理することになるので、`localhost.` とだけ記述します。

```
       IN      NS       localhost.
```

### Aレコード
Aレコードは、正引きのドメインとIPアドレスの紐付けを行うレコードになります。

こちらはデータフィールドに紐づけるIPアドレスを指定すればOKです。

今回は、`192.168.200.42` とだけ記述します。

```
       IN      A        142.250.207.3
```

### PTRレコード
PTRレコードは、逆引きのドメインとIPアドレスの紐付けを行うレコードになります。

こちらは、ゾーンを範囲で設定している場合、ラベルに可変部のアドレスを記述する必要があります。

今回の場合だと逆引きのゾーン設定は、`192.168.200.xxx` の設定を行うようにしており、可変部 `xxx` には `42` が入るので、ラベルには `42` とだけ記述します。

`42` 以外にも設定したい場合には適宜追加するといいでしょう。

```
42     IN      PTR      tksx1227.dns-server.com.
```

これで正引き、逆引きの設定ができたので、動作確認に移りましょう。

# 動作確認
DNSを起動して動作確認を行います。

```bash
$ systemctl start named
$ systemctl enable named
```

動作確認には `nslookup` コマンドを使用します。

`nslookup` は以下のようにして使用できます。

DNSサーバーを指定しない場合、既存のDNSサーバーに問い合わせに行ってしまうため、こちらは `localhost` を指定するようにしましょう。

```bash
$ nslookup <IP Address or Domain Name> [DNS Server]
```

では早速確認してみましょう。

```bash
$ nslookup tksx1227.dns-server.com localhost
Server:         localhost
Address:        127.0.0.1#53

Name:   tksx1227.dns-server.com
Address: 192.168.200.42

$ nslookup 192.168.200.42 localhost
42.200.168.192.in-addr.arpa     name = tksx1227.dns-server.com.

```

これで無事設定通りに名前解決が実行できていることが確認できました。

やったぜ ٩(ˊᗜˋ*)و

# おわりに
DNSサーバーなんて普段触らないので、自分で構築するのも楽しいですね。

ただ、まだまだ自分の知らない設定が数多くあるようなので、時間があればもう少しセキュリティなどを意識した構成も調べてみようと思います。

技術的な間違いのご指摘は大歓迎なので、是非コメントいただけると幸いですmm

# 参考にした記事
@[card](https://nanbu.marune205.net/2021/12/debian-dns-bind.html?m=1)
@[card](https://digitalmania.info/linux/debian-11/build-dns-server/)
@[card](http://park12.wakwak.com/~eslab/pcmemo/linux/bind/bind2.html)
@[card](https://qiita.com/leomaro7/items/d6907d0a2aee890fe52b#%E3%82%BE%E3%83%BC%E3%83%B3%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB)
@[card](https://atmarkit.itmedia.co.jp/fnetwork/dnstips/031.html)
@[card](https://www.eis.co.jp/bind9_src_build_3/)
@[card](https://wa3.i-3-i.info/word12285.html)

