---
title: "OpenVPNをLet's Encryptのサーバー証明書とオレオレCAのクライアント証明書で設定[Ubuntu 22.04 LTS]"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["OpenVPN", "LetsEncrypt", "Ubuntu", "Route53"]
published: false
---

## はじめに

手元にあった，昔のMacBook ProのSSDが故障したため，これを機に外付けSSDを接続・Ubuntuをインストールし，OpenVPNサーバーを立てて，iOSなどのクライアントから接続できるようにしてみました。

過程で，サーバーの証明書はLet's EncryptをDNS認証で使用してみることにしました。DNSはRoute53を使用しているので，Route53のみの権限を用意したIAMを用意して，これを使っていきます。（失効の厳密管理などを考えるとあまりよくないかも）

IPv4・IPv6双方のルーティングの設定も行いました。

これらの備忘録として，Ubuntuのインストールも含め，備忘録として概要をこちらにまとめておきます。

## Ubuntuのインストール

OSのインストールは，詳しい人がいろんな記事を書いてくれていると思うので，流れだけメモしておきます。

1. 適当なディスクを用意します（僕の場合は，外付けSSDをそのまま使用しましたが，USBメモリなどでももちろんOK）
2. [公式](https://jp.ubuntu.com/download)もしくは[ミラーサイト](https://www.ubuntulinux.jp/download/ja-remix)から，`ubuntu-ja-22.04-desktop-amd64.iso`を落とします。
3. `shasum -a 256 ubuntu-ja-22.04-desktop-amd64.iso`でハッシュ値を確認します（任意だけどやったほうが良いと思う）。
4. [Etcher](https://www.balena.io/etcher/)などを使って，インストールディスクを作成します。
5. インストールディスクをMBPに挿入し，optionを押しながら起動します。
6. 指示に従ってインストールします。SSDをそのままvolumeとして使うように設定できました。
7. https://github.com/t2linux/T2-Ubuntu-Kernel をいれる
8. 完了したら，ネットワークの設定や，SSHの設定などを行います。SSHのパスワード認証は早めに無効にしておきましょう。

https://qiita.com/miriwo/items/798dbbcf2d37c7ef2c3c

## OpenVPNのインストール

```bash
sudo apt install openvpn easy-rsa
```

## 各証明書の作成

```bash
$ make-cadir ~/openvpn-ca
$ cd ~/openvpn-ca
$ vim vars
set_var EASYRSA_REQ_COUNTRY     "JA"
set_var EASYRSA_REQ_PROVINCE    "Kanagawa"
set_var EASYRSA_REQ_CITY        "Yokohama"
set_var EASYRSA_REQ_ORG "tak848"
set_var EASYRSA_REQ_EMAIL       "tak848@tak848.net"
set_var EASYRSA_REQ_OU          "tak848"

$ ./easyrsa init-pki

$ ./easyrsa build-ca
# passphraseを入力しないとエラーになるので，入力する。忘れると最初からやり直しになるので注意。
```

### クライアント証明書作成

以下で完了します。`client-example`は任意の名前でOKです。クライアント証明書を端末ごとに使い分けたいなどあれば，それぞれの端末ごとに名前を変えて全て作成してください。

```bash
./easyrsa gen-req client-example nopass
./easyrsa sign-req client client-example # ここで，passphraseを入力する
```

### サーバー証明書作成

以下で完了します。`server`は任意の名前でOKです。

```bash
./easyrsa gen-req server-example nopass
./easyrsa sign-req server server-example # ここで，passphraseを入力する
```

### dhパラメータの作成

```bash
./easyrsa gen-dh
```

### easy-rsaのファイル確認

ここまでで，`~/openvpn-ca`に以下のようなファイルができているはずです。

```text
├── easyrsa -> /usr/share/easy-rsa/easyrsa
├── openssl-easyrsa.cnf
├── pki
│   ├── ca.crt
│   ├── certs_by_serial
│   │   ├── 5030B7440D3E05F9884BF46F9E986318.pem
│   │   └── B02873EF63832381E62DF4084FAE7467.pem
│   ├── dh.pem
│   ├── index.txt
│   ├── index.txt.attr
│   ├── index.txt.attr.old
│   ├── index.txt.old
│   ├── issued
│   │   ├── client-example.crt
│   │   └── server-example.crt
│   ├── openssl-easyrsa.cnf
│   ├── private
│   │   ├── ca.key
│   │   ├── client-example.key
│   │   └── server-example.key
│   ├── renewed
│   │   ├── certs_by_serial
│   │   ├── private_by_serial
│   │   └── reqs_by_serial
│   ├── reqs
│   │   ├── client-example.req
│   │   └── server-example.req
│   ├── revoked
│   │   ├── certs_by_serial
│   │   ├── private_by_serial
│   │   └── reqs_by_serial
│   ├── safessl-easyrsa.cnf
│   ├── serial
│   └── serial.old
├── vars
└── x509-types -> /usr/share/easy-rsa/x509-types
```

以降，以下のファイルを使用します。

* `~/openvpn-ca/pki/ca.crt`を，検証用のCAとして，サーバー・クライアント両方に登録
* `~/openvpn-ca/pki/dh.pem`を，TLSのパラメータとして，サーバーに登録
* `~/openvpn-ca/pki/issued/client-example.crt`を，クライアント側の証明書として，クライアントに登録
* `~/openvpn-ca/pki/private/client-example.key`を，クライアント側の証明書の秘密鍵として，クライアントに登録
* `~/openvpn-ca/pki/issued/server-example.crt`を，サーバー側の証明書として，サーバーに登録
* `~/openvpn-ca/pki/private/server-example.key`を，サーバー側の証明書の秘密鍵として，サーバーに登録

`/etc/openvpn/`配下に移動しておくと，後で楽です。

### ta.keyの作成

```bash
openvpn --genkey --secret /etc/openvpn/ta.key
```

こちらはサーバー側・クライアント側ともに使用します。

### サーバー証明書

#### 前提条件

* `~/.aws`に，Route53を操作可能なIAMユーザーのクレデンシャルが保存されていること（今回は，openvpn-route53という名前で作成した）
  * `aws configure --profile openvpn-route53`などで設定すればOK。



#### dockerを使わない場合

##### 必要パッケージインストール

```bash
sudo apt install certbot python3-certbot-dns-route53
```

##### 作成

```bash
sudo AWS_PROFILE=openvpn-route53 certbot certonly --dns-route53 -d ca.openvpn.example.com
```

* `/etc/letsencrypt/live/ca.openvpn.example.com/fullchain.pem`
* `/etc/letsencrypt/live/ca.openvpn.example.com/privkey.pem`

以上の2つを，サーバー証明書として使用します。
自動更新については，AWS_PROFILEがdefaultであればsystemctlで簡単に自動更新できるようですが，今回はopenvpn-route53という名前のprofileを使用しているため，cronなどを使って自動更新すると良い気がします。

#### docker composeを使う場合

以下を参考に`compose.yaml`を作成します。

```yaml
version: "3.8"

services:
  certbot:
    image: certbot/dns-route53
    container_name: certbot
    command: certonly --dns-route53 -d openvpn.example.com
    volumes:
      - ./etc/letsencrypt:/etc/letsencrypt
      - ./var/lib/letsencrypt:/var/lib/letsencrypt
      - ~/.aws:/root/.aws
    environment:
      - AWS_PROFILE=route53 # AWSのprofileをここに指定
    stdin_open: true
    tty: true
```

```bash
docker compose run certbot
```

とすることで，カレントディレクトリに

* `etc/letsencrypt/live/openvpn.example.com/fullchain.pem`
* `etc/letsencrypt/live/openvpn.example.com/privkey.pem`

が作成されます。volumesの設定を変えれば，任意の場所に保存することもできます。

自動更新についてはまだあまり考えていないので，後で追記したいと思います。

## OpenVPNの設定

### サーバー側の設定

#### server用テンプレートのコピー

```bash
sudo cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf /etc/openvpn/
```

#### ポート・プロトコル設定

必要に応じて，udp/tcpや，ポート番号を変更してください。

```diff conf
port 1194 # 任意のポート番号
proto udp # tcpにもできる
```

例えば，tcp 443にすると，ファイアウォールを通りやすくなるかもしれません。

### topology subnetの設定

```diff conf
- ;topology subnet
+ topology subnet
```

https://qiita.com/tomoki0sanaki/items/740aacea16ab7f1241a0#openvpn%E3%82%B5%E3%83%BC%E3%83%90%E3%81%AE%E8%A8%AD%E5%AE%9Atun%E3%81%AE%E5%A0%B4%E5%90%88topology-subnet%E3%81%AB%E3%81%A4%E3%81%84%E3%81%A6

のページにもあるように，別のサブネットとなって通信ができなくなるため，コメントアウトする必要があるみたいです。

#### 鍵の位置の変更

ここまで作成した鍵のパスを指定します。`/home/user/`のような，ユーザーフォルダは参照できないので注意。

```diff conf
dev tun
- ca ca.crt
- cert server.crt
- key server.key  # This file should be kept secret
+ ca /path/to/ca.crt # クライアント証明書のCA（easy-rsaのca.crt）
+ cert /etc/letsencrypt/live/openvpn.example.com/fullchain.pem # サーバー証明書
+ key /etc/letsencrypt/live/openvpn.example.com/privkey.pem # サーバー証明書の秘密鍵

- dh dh2048.pem
+ dh /path/to/dh.pem # dhパラメータ

- tls-auth ta.key 0 # This file is secret
+ tls-auth /etc/openvpn/ta.key 0 # ta.key
```

#### ルーティングの設定

```diff conf:変更箇所
;push "route 192.168.10.0 255.255.255.0"
;push "route 192.168.20.0 255.255.255.0"
+ push "route 10.0.0.0 255.252.0.0" # 自分のネットワークのルーティング

+ push "redirect-gateway def1 bypass-dhcp" # クライアントからの通信をすべてVPN経由にする
```

#### ユーザー・グループの変更

```diff conf
- ;user nobody
- ;group nobody
+ user nobody
+ group nogroup
```

#### TCPの場合の設定※UDPでは不要

TCPの場合，最後の方にある以下をコメントアウトする必要があります。

```diff conf
- explicit-exit-notify 1
+ ;explicit-exit-notify 1
```

#### DNSの設定（任意）

ローカルでホストしているこのDNSを使ってほしい！などあれば，以下のような設定を追加できます。

試していませんが，ipv6も含め複数設定で着るみたいです。

```conf:DNS設定
push "dhcp-option DNS 192.168.1.1"
```

### クライアント用ovpnファイルの作成

#### client用テンプレートのコピー

```bash
sudo cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf /etc/openvpn/client-example.ovpn
```

#### サーバー・ポート・プロトコル設定

以下のように，プロトコルとIPもしくはホスト名，ポート番号を設定します。

IPアドレスが実質固定ならIPでも良いですが，基本的にはDDNSなどを使ってホスト名で指定すると良いと思います。

```conf:設定箇所
proto udp # tcpにもできる

remote myserver.example.com 1194 # サーバーのIPもしくはホスト名とポート番号
```

#### 各証明書の埋め込み

サーバー証明書検証用のCA，クライアント証明書，クライアント証明書の秘密鍵，TLSのパラメータを埋め込みます。

```diff conf:変更箇所
- ca ca.crt
+ ;ca ca.crt
+ <ca>
+ -----BEGIN CERTIFICATE-----
+ ... `~/openvpn-ca/pki/ca.crt`の内容 ...
+ -----END CERTIFICATE-----
+ </ca>

- cert client.crt
+ ;cert client.crt
+ <cert>
+ -----BEGIN CERTIFICATE-----
+ ... `~/openvpn-ca/pki/issued/client-example.crt`の内容 ...
+ -----END CERTIFICATE-----
+ </cert>

- key client.key
+ ;key client.key
+ <key>
+ -----BEGIN PRIVATE KEY-----
+ ... `~/openvpn-ca/pki/private/client-example.key`の内容 ...
+ -----END PRIVATE KEY-----
+ </key>

- tls-auth ta.key 1
+ ;tls-auth ta.key 1
+ <tls-auth>
+ -----BEGIN OpenVPN Static key V1-----
+ ... `/etc/openvpn/ta.key`の内容 ...
+ -----END OpenVPN Static key V1-----
+ </tls-auth>
+ key-direction 1 ### 忘れずに！！！！
```

## iptableによるルーティング設定

これからの設定を永続化するために，`iptables-persistent`をインストールします。

```bash
sudo apt install iptables-persistent net-tools
```

### ネットワークインターフェイス名を確認

```bash
$ ifconfig
...
enp6s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.123  netmask 255.255.255.0  broadcast 192.168.0.255
...
```

`ifconfig`を実行して，inetのところに，自分のサーバーのIPアドレスが表示されているネットワークインターフェイス名を確認します。wifiとethernet双方接続しているなどすると，複数あるので使いたい方を選ぶ。

### ポートフォワーディングの許可

```bash
sudo vim /etc/sysctl.conf
```

```diff conf
- #net.ipv4.ip_forward=1
+ net.ipv4.ip_forward=1

# 以下は任意
- #net.ipv6.conf.all.forwarding=1
+ net.ipv6.conf.all.forwarding=1
```

```bash
sudo sysctl -p
```

### IPv4

set MASQUERADE rule

```bash
$ sudo iptables -A FORWARD -i tun0 -o enp6s0 -j ACCEPT
$ sudo iptables -A FORWARD -i enp6s0 -o tun0 -j ACCEPT
$ sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o enp6s0 -j MASQUERADE
$ sudo iptables -L -v # 確認
...
Chain FORWARD (policy DROP x packets, y bytes)
 pkts bytes target     prot opt in     out     source               destination
...
 3607  400K ACCEPT     all  --  tun0   enp6s0  anywhere             anywhere
 3856 4504K ACCEPT     all  --  enp6s0 tun0    anywhere             anywhere
 ...
$ sudo iptables -t nat -L -v # 確認
...
Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
   23  1380 MASQUERADE  all  --  any    !br-d609a4db1ceb  172.18.0.0/16        anywhere
    0     0 MASQUERADE  all  --  any    !docker0  172.17.0.0/16        anywhere
  605 43679 MASQUERADE  all  --  any    enp6s0  10.8.0.0/24          anywhere
...
```

ちなみに，もしこの辺何度も実行してしまったときは，

```bash:iptablesのリセット（やらかした時のみ）
sudo iptables -t nat -F POSTROUTING
```

で一旦リセットしてからやり直す。(Docker関係のやつがすでに入ってたけど，まあ再起動すれば設定してくれるっぽかったので結果的にﾖｼ!)

### IPv6（任意）

IPv6も使用したい場合は，serverのconfに以下を追加します。

```diff conf
server 10.8.0.0 255.255.255.0
+ server-ipv6 fd00:1234:5678::/64
```

```diff conf
push "redirect-gateway def1 bypass-dhcp" # クライアントからの通信をすべてVPN経由にする
+ push "redirect-gateway ipv6" # IPv6もVPN経由にする
```

その上で，ip6tablesの設定を行います。

```bash
$ sudo ip6tables -A FORWARD -i tun0 -o enp6s0 -j ACCEPT
$ sudo ip6tables -A FORWARD -i enp6s0 -o tun0 -j ACCEPT
$ sudo ip6tables -t nat -A POSTROUTING -s fd00:1234:5678::/64 -o enp6s0 -j MASQUERADE
$ sudo ip6tables -L -v # 確認
...
Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
 1205  171K ACCEPT     all      tun0   enp6s0  anywhere             anywhere
  833  762K ACCEPT     all      enp6s0 tun0    anywhere             anywhere
 ...
$ sudo ip6tables -t nat -L -v # 確認
...
Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
  118 18036 MASQUERADE  all      any    enp6s0  fd00:1234:5678::/64  anywhere
...
```

## iOSなどからの接続

OpenVPN Connectアプリをインストールします。

そして作成したovpnファイルを，airdropなどの好きな手段で転送し，OpenVPN Connectアプリで開きます。

プロファイル名を入力して，接続できれば完了です。

## 詰まったポイント

### UDPだと2分で切れる

### iptablesに使用するネットワークインターフェイス指定

## おわりに

ここまでで，ひとまずOpenVPNサーバーの設定は完了です。
無効な証明書を使えなくするには，CRLを使う必要があるのですが，これはまた別の機会にやってみたいと思います。

