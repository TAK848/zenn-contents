---
title: "OpenVPNをオレオレCAのサーバー・クライアント証明書で設定[Ubuntu 22.04 LTS]"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["OpenVPN", "Ubuntu", "Route53", "easyrsa"]
published: true
---

## はじめに

手元にあった，昔のMacBook ProのSSDが故障したため，これを機に外付けSSDを接続・Ubuntuをインストールし，OpenVPNサーバーを立てて，iOSなどのクライアントから接続できるようにしてみました。

IPv4・IPv6双方のルーティングの設定・DNSの設定も行いました。

これらの備忘録として，Ubuntuのインストールも含め，備忘録として概要をこちらにまとめておきます。

## Ubuntuのインストール

OSのインストールは，詳しい人がいろんな記事を書いてくれていると思うので，流れだけメモしておきます。

1. 適当なディスクを用意します（僕の場合は，外付けSSDをそのまま使用しましたが，USBメモリなどでももちろんOK）
2. [公式](https://jp.ubuntu.com/download)もしくは[ミラーサイト](https://www.ubuntulinux.jp/download/ja-remix)から，`ubuntu-ja-22.04-desktop-amd64.iso`を落とします。
3. `shasum -a 256 ubuntu-ja-22.04-desktop-amd64.iso`でハッシュ値を確認します（任意だけどやったほうが良いと思う）。
4. [Etcher](https://www.balena.io/etcher/)などを使って，インストールディスクを作成します。
5. インストールディスクをMBPに挿入し，optionを押しながら起動します。
6. 指示に従ってインストールします。SSDをそのままvolumeとして使うように設定できました。
7. 完了したら，ネットワークの設定や，SSHの設定などを行います。SSHのパスワード認証は早めに無効にしておきましょう。
8. https://github.com/t2linux/T2-Ubuntu-Kernel をいれる

:::details sshの設定

インストール・接続

```bash
$ sudo apt update
$ sudo apt install openssh-server
$ sudo systemctl is-enabled ssh
enabled

$ sudo apt install neovim # 好みで
$ mkdir .ssh
$ vim ~/.ssh/authorized_keys # 公開鍵を入れる
```

パスワード認証無効化

```bash
sudo vim /etc/ssh/sshd_config
```

以下のように編集

```diff conf
To disable tunneled clear text passwords, change to no here!
- #PasswordAuthentication yes
- #PermitEmptyPasswords no
+ PasswordAuthentication no
+ PermitEmptyPasswords no
```

再起動

```bash
sudo systemctl restart ssh
```

:::

:::details T2-Ubuntu_kernelのインストール

```bash
sudo rm -r /usr/src/apple-bce*
sudo rm -r /usr/src/apple-ibridge*
sudo rm -r /var/lib/dkms/apple-bce
sudo rm -r /var/lib/dkms/apple-ibridge

sudo apt install curl

curl -s --compressed "https://adityagarg8.github.io/t2-ubuntu-repo/KEY.gpg" | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/t2-ubuntu-repo.gpg >/dev/null
sudo curl -s --compressed -o /etc/apt/sources.list.d/t2.list "https://adityagarg8.github.io/t2-ubuntu-repo/t2.list"
sudo apt update

exec $SHELL -l

sudo apt install t2-kernel-script

update_t2_kernel
```

@[card](https://github.com/t2linux/T2-Ubuntu-Kernel)

:::

@[card](https://qiita.com/miriwo/items/798dbbcf2d37c7ef2c3c)

## OpenVPNのインストール

証明書の作成には，`easy-rsa`を使用します。

```bash
sudo apt install openvpn easy-rsa
```

## 各証明書の作成

```bash
$ make-cadir ~/openvpn-ca
$ cd ~/openvpn-ca
$ vim vars
set_var EASYRSA_REQ_COUNTRY     "JA"
set_var EASYRSA_REQ_PROVINCE    "Tokyo"
set_var EASYRSA_REQ_CITY        "Shinjuku"
set_var EASYRSA_REQ_ORG "example org"
set_var EASYRSA_REQ_EMAIL       "sample@example.com"
set_var EASYRSA_REQ_OU          "example org"

$ ./easyrsa init-pki

$ ./easyrsa build-ca
# passphraseを入力しないとエラーになるので，入力する。忘れると最初からやり直しになるので注意。CAのCommon Nameも設定。
```

### クライアント証明書作成

以下で完了します。`client-example`は任意の名前でOKです。クライアント証明書を端末ごとに使い分けたいなどあれば，それぞれの端末ごとに名前を変えて全て作成してください。

```bash
./easyrsa build-client-full client-sample nopass # ここで，先ほどのCAのpassphraseを入力する
# ./easyrsa gen-req client-example nopass
# ./easyrsa sign-req client client-example # ここで，passphraseを入力する
```

### サーバー証明書作成

以下で完了します。`server`は任意の名前でOKです。

```bash
./easyrsa build-server-full server-example nopass
# ./easyrsa gen-req server-example nopass
# ./easyrsa sign-req server server-example # ここで，passphraseを入力する
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
│   │   ├── XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX.pem
│   │   └── YYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYY.pem
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
sudo openvpn --genkey secret /etc/openvpn/ta.key # /home/username配下ならsudoいらないかも
```

こちらはサーバー側・クライアント側ともに使用します。

## 設定ファイルの作成

### サーバー側confの作成

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

#### topology subnetの設定

```diff conf
- ;topology subnet
+ topology subnet
```

@[card](https://qiita.com/tomoki0sanaki/items/740aacea16ab7f1241a0#openvpn%E3%82%B5%E3%83%BC%E3%83%90%E3%81%AE%E8%A8%AD%E5%AE%9Atun%E3%81%AE%E5%A0%B4%E5%90%88topology-subnet%E3%81%AB%E3%81%A4%E3%81%84%E3%81%A6)

のページにもあるように，別のサブネットとなって通信ができなくなるため，コメントアウトする必要があるみたいです。

#### 鍵の位置の変更

ここまで作成した鍵のパスを指定します。`/home/user/`のような，ユーザーフォルダは参照できないので注意。相対パスでもOK。

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

- ;push "redirect-gateway def1 bypass-dhcp"
+ push "redirect-gateway def1 bypass-dhcp" # クライアントからの通信をすべてVPN経由にする
```

IPv6も使用したい場合は，さらに以下も追記しておきましょう。

```diff conf:IPv6向けの設定
push "route 10.0.0.0 255.252.0.0" # 自分のネットワークのルーティング
+ server-ipv6 fd00:1234:5678::/64

push "redirect-gateway def1 bypass-dhcp" # クライアントからの通信をすべてVPN経由にする
+ push "redirect-gateway ipv6" # IPv6もVPN経由にする
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

試していませんが，ipv6も含め複数設定できるみたいです。

```conf:DNS設定
push "dhcp-option DNS 192.168.1.1"
```

#### 最終的なconfファイル例


:::details server.confの例

```conf
#################################################
# Sample OpenVPN 2.0 config file for            #
# multi-client server.                          #
#                                               #
# This file is for the server side              #
# of a many-clients <-> one-server              #
# OpenVPN configuration.                        #
#                                               #
# OpenVPN also supports                         #
# single-machine <-> single-machine             #
# configurations (See the Examples page         #
# on the web site for more info).               #
#                                               #
# This config should work on Windows            #
# or Linux/BSD systems.  Remember on            #
# Windows to quote pathnames and use            #
# double backslashes, e.g.:                     #
# "C:\\Program Files\\OpenVPN\\config\\foo.key" #
#                                               #
# Comments are preceded with '#' or ';'         #
#################################################

# Which local IP address should OpenVPN
# listen on? (optional)
;local a.b.c.d

# Which TCP/UDP port should OpenVPN listen on?
# If you want to run multiple OpenVPN instances
# on the same machine, use a different port
# number for each one.  You will need to
# open up this port on your firewall.
port 1194

# TCP or UDP server?
;proto tcp
proto udp

# "dev tun" will create a routed IP tunnel,
# "dev tap" will create an ethernet tunnel.
# Use "dev tap0" if you are ethernet bridging
# and have precreated a tap0 virtual interface
# and bridged it with your ethernet interface.
# If you want to control access policies
# over the VPN, you must create firewall
# rules for the the TUN/TAP interface.
# On non-Windows systems, you can give
# an explicit unit number, such as tun0.
# On Windows, use "dev-node" for this.
# On most systems, the VPN will not function
# unless you partially or fully disable
# the firewall for the TUN/TAP interface.
;dev tap
dev tun

# Windows needs the TAP-Win32 adapter name
# from the Network Connections panel if you
# have more than one.  On XP SP2 or higher,
# you may need to selectively disable the
# Windows firewall for the TAP adapter.
# Non-Windows systems usually don't need this.
;dev-node MyTap

# SSL/TLS root certificate (ca), certificate
# (cert), and private key (key).  Each client
# and the server must have their own cert and
# key file.  The server and all clients will
# use the same ca file.
#
# See the "easy-rsa" directory for a series
# of scripts for generating RSA certificates
# and private keys.  Remember to use
# a unique Common Name for the server
# and each of the client certificates.
#
# Any X509 key management system can be used.
# OpenVPN can also use a PKCS #12 formatted key file
# (see "pkcs12" directive in man page).
ca ca.crt
cert imac-ubuntu-server.crt
key imac-ubuntu-server.key  # This file should be kept secret

# Diffie hellman parameters.
# Generate your own with:
#   openssl dhparam -out dh2048.pem 2048
dh dh.pem

# Network topology
# Should be subnet (addressing via IP)
# unless Windows clients v2.0.9 and lower have to
# be supported (then net30, i.e. a /30 per client)
# Defaults to net30 (not recommended)
topology subnet

# Configure server mode and supply a VPN subnet
# for OpenVPN to draw client addresses from.
# The server will take 10.8.0.1 for itself,
# the rest will be made available to clients.
# Each client will be able to reach the server
# on 10.8.0.1. Comment this line out if you are
# ethernet bridging. See the man page for more info.
server 10.8.0.0 255.255.255.0
server-ipv6 fd00:1234:5678:::/64

# Maintain a record of client <-> virtual IP address
# associations in this file.  If OpenVPN goes down or
# is restarted, reconnecting clients can be assigned
# the same virtual IP address from the pool that was
# previously assigned.
ifconfig-pool-persist /var/log/openvpn/ipp.txt

# Configure server mode for ethernet bridging.
# You must first use your OS's bridging capability
# to bridge the TAP interface with the ethernet
# NIC interface.  Then you must manually set the
# IP/netmask on the bridge interface, here we
# assume 10.8.0.4/255.255.255.0.  Finally we
# must set aside an IP range in this subnet
# (start=10.8.0.50 end=10.8.0.100) to allocate
# to connecting clients.  Leave this line commented
# out unless you are ethernet bridging.
;server-bridge 10.8.0.4 255.255.255.0 10.8.0.50 10.8.0.100

# Configure server mode for ethernet bridging
# using a DHCP-proxy, where clients talk
# to the OpenVPN server-side DHCP server
# to receive their IP address allocation
# and DNS server addresses.  You must first use
# your OS's bridging capability to bridge the TAP
# interface with the ethernet NIC interface.
# Note: this mode only works on clients (such as
# Windows), where the client-side TAP adapter is
# bound to a DHCP client.
;server-bridge

# Push routes to the client to allow it
# to reach other private subnets behind
# the server.  Remember that these
# private subnets will also need
# to know to route the OpenVPN client
# address pool (10.8.0.0/255.255.255.0)
# back to the OpenVPN server.
;push "route 192.168.10.0 255.255.255.0"
;push "route 192.168.20.0 255.255.255.0"
push "route 10.0.0.0 255.252.0.0"

# To assign specific IP addresses to specific
# clients or if a connecting client has a private
# subnet behind it that should also have VPN access,
# use the subdirectory "ccd" for client-specific
# configuration files (see man page for more info).

# EXAMPLE: Suppose the client
# having the certificate common name "Thelonious"
# also has a small subnet behind his connecting
# machine, such as 192.168.40.128/255.255.255.248.
# First, uncomment out these lines:
;client-config-dir ccd
;route 192.168.40.128 255.255.255.248
# Then create a file ccd/Thelonious with this line:
#   iroute 192.168.40.128 255.255.255.248
# This will allow Thelonious' private subnet to
# access the VPN.  This example will only work
# if you are routing, not bridging, i.e. you are
# using "dev tun" and "server" directives.

# EXAMPLE: Suppose you want to give
# Thelonious a fixed VPN IP address of 10.9.0.1.
# First uncomment out these lines:
;client-config-dir ccd
;route 10.9.0.0 255.255.255.252
# Then add this line to ccd/Thelonious:
#   ifconfig-push 10.9.0.1 10.9.0.2

# Suppose that you want to enable different
# firewall access policies for different groups
# of clients.  There are two methods:
# (1) Run multiple OpenVPN daemons, one for each
#     group, and firewall the TUN/TAP interface
#     for each group/daemon appropriately.
# (2) (Advanced) Create a script to dynamically
#     modify the firewall in response to access
#     from different clients.  See man
#     page for more info on learn-address script.
;learn-address ./script

# If enabled, this directive will configure
# all clients to redirect their default
# network gateway through the VPN, causing
# all IP traffic such as web browsing and
# and DNS lookups to go through the VPN
# (The OpenVPN server machine may need to NAT
# or bridge the TUN/TAP interface to the internet
# in order for this to work properly).
push "redirect-gateway def1 bypass-dhcp"
push "redirect-gateway ipv6"

# Certain Windows-specific network settings
# can be pushed to clients, such as DNS
# or WINS server addresses.  CAVEAT:
# http://openvpn.net/faq.html#dhcpcaveats
# The addresses below refer to the public
# DNS servers provided by opendns.com.
;push "dhcp-option DNS 208.67.222.222"
;push "dhcp-option DNS 208.67.220.220"
push "dhcp-option DNS 10.2.1.1"

# Uncomment this directive to allow different
# clients to be able to "see" each other.
# By default, clients will only see the server.
# To force clients to only see the server, you
# will also need to appropriately firewall the
# server's TUN/TAP interface.
;client-to-client

# Uncomment this directive if multiple clients
# might connect with the same certificate/key
# files or common names.  This is recommended
# only for testing purposes.  For production use,
# each client should have its own certificate/key
# pair.
#
# IF YOU HAVE NOT GENERATED INDIVIDUAL
# CERTIFICATE/KEY PAIRS FOR EACH CLIENT,
# EACH HAVING ITS OWN UNIQUE "COMMON NAME",
# UNCOMMENT THIS LINE OUT.
;duplicate-cn

# The keepalive directive causes ping-like
# messages to be sent back and forth over
# the link so that each side knows when
# the other side has gone down.
# Ping every 10 seconds, assume that remote
# peer is down if no ping received during
# a 120 second time period.
keepalive 10 120

# For extra security beyond that provided
# by SSL/TLS, create an "HMAC firewall"
# to help block DoS attacks and UDP port flooding.
#
# Generate with:
#   openvpn --genkey tls-auth ta.key
#
# The server and each client must have
# a copy of this key.
# The second parameter should be '0'
# on the server and '1' on the clients.
tls-auth ta.key 0 # This file is secret

# Select a cryptographic cipher.
# This config item must be copied to
# the client config file as well.
# Note that v2.4 client/server will automatically
# negotiate AES-256-GCM in TLS mode.
# See also the ncp-cipher option in the manpage
cipher AES-256-CBC

# Enable compression on the VPN link and push the
# option to the client (v2.4+ only, for earlier
# versions see below)
;compress lz4-v2
;push "compress lz4-v2"

# For compression compatible with older clients use comp-lzo
# If you enable it here, you must also
# enable it in the client config file.
;comp-lzo

# The maximum number of concurrently connected
# clients we want to allow.
;max-clients 100

# It's a good idea to reduce the OpenVPN
# daemon's privileges after initialization.
#
# You can uncomment this out on
# non-Windows systems.
user nobody
group nogroup

# The persist options will try to avoid
# accessing certain resources on restart
# that may no longer be accessible because
# of the privilege downgrade.
persist-key
persist-tun

# Output a short status file showing
# current connections, truncated
# and rewritten every minute.
status /var/log/openvpn/openvpn-status.log

# By default, log messages will go to the syslog (or
# on Windows, if running as a service, they will go to
# the "\Program Files\OpenVPN\log" directory).
# Use log or log-append to override this default.
# "log" will truncate the log file on OpenVPN startup,
# while "log-append" will append to it.  Use one
# or the other (but not both).
;log         /var/log/openvpn/openvpn.log
;log-append  /var/log/openvpn/openvpn.log

# Set the appropriate level of log
# file verbosity.
#
# 0 is silent, except for fatal errors
# 4 is reasonable for general usage
# 5 and 6 can help to debug connection problems
# 9 is extremely verbose
verb 3

# Silence repeating messages.  At most 20
# sequential messages of the same message
# category will be output to the log.
;mute 20

# Notify the client that when the server restarts so it
# can automatically reconnect.
explicit-exit-notify 1
```

:::

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

#### 最終的なovpnファイルの例


:::details client.ovpnの例

```conf
##############################################
# Sample client-side OpenVPN 2.0 config file #
# for connecting to multi-client server.     #
#                                            #
# This configuration can be used by multiple #
# clients, however each client should have   #
# its own cert and key files.                #
#                                            #
# On Windows, you might want to rename this  #
# file so it has a .ovpn extension           #
##############################################

# Specify that we are a client and that we
# will be pulling certain config file directives
# from the server.
client

# Use the same setting as you are using on
# the server.
# On most systems, the VPN will not function
# unless you partially or fully disable
# the firewall for the TUN/TAP interface.
;dev tap
dev tun

# Windows needs the TAP-Win32 adapter name
# from the Network Connections panel
# if you have more than one.  On XP SP2,
# you may need to disable the firewall
# for the TAP adapter.
;dev-node MyTap

# Are we connecting to a TCP or
# UDP server?  Use the same setting as
# on the server.
proto tcp
;proto udp

# The hostname/IP and port of the server.
# You can have multiple remote entries
# to load balance between the servers.
remote et.imac.4xq8.in.tak848.net 61443
;remote my-server-2 1194

# Choose a random host from the remote
# list for load-balancing.  Otherwise
# try hosts in the order specified.
;remote-random

# Keep trying indefinitely to resolve the
# host name of the OpenVPN server.  Very useful
# on machines which are not permanently connected
# to the internet such as laptops.
resolv-retry infinite

# Most clients don't need to bind to
# a specific local port number.
nobind

# Downgrade privileges after initialization (non-Windows only)
;user nobody
;group nobody

# Try to preserve some state across restarts.
persist-key
persist-tun

# If you are connecting through an
# HTTP proxy to reach the actual OpenVPN
# server, put the proxy server/IP and
# port number here.  See the man page
# if your proxy server requires
# authentication.
;http-proxy-retry # retry on connection failures
;http-proxy [proxy server] [proxy port #]

# Wireless networks often produce a lot
# of duplicate packets.  Set this flag
# to silence duplicate packet warnings.
;mute-replay-warnings

# SSL/TLS parms.
# See the server config file for more
# description.  It's best to use
# a separate .crt/.key file pair
# for each client.  A single ca
# file can be used for all clients.
;ca ca.crt
<ca>
-----BEGIN CERTIFICATE-----

-----END CERTIFICATE-----
</ca>
;cert client.crt
<cert>
-----BEGIN CERTIFICATE-----

-----END CERTIFICATE-----
</cert>
;key client.key
<key>
-----BEGIN PRIVATE KEY-----

-----END PRIVATE KEY-----
</key>

# Verify server certificate by checking that the
# certificate has the correct key usage set.
# This is an important precaution to protect against
# a potential attack discussed here:
#  http://openvpn.net/howto.html#mitm
#
# To use this feature, you will need to generate
# your server certificates with the keyUsage set to
#   digitalSignature, keyEncipherment
# and the extendedKeyUsage to
#   serverAuth
# EasyRSA can do this for you.
remote-cert-tls server

# If a tls-auth key is used on the server
# then every client must also have the key.
;tls-auth ta.key 1
<tls-auth>
-----BEGIN OpenVPN Static key V1-----

-----END OpenVPN Static key V1-----
</tls-auth>
key-direction 1

# Select a cryptographic cipher.
# If the cipher option is used on the server
# then you must also specify it here.
# Note that v2.4 client/server will automatically
# negotiate AES-256-GCM in TLS mode.
# See also the data-ciphers option in the manpage
cipher AES-256-CBC

# Enable compression on the VPN link.
# Don't enable this unless it is also
# enabled in the server config file.
#comp-lzo

# Set log file verbosity.
verb 3

# Silence repeating messages
;mute 20
```

:::

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

受信と送信を許可し，NATを設定します。

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

IPv4と同様に，受信と送信を許可し，NATを設定します。

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

### 保存

マシンを再起動すると消えてしまうので永続化！

```bash
sudo /etc/init.d/netfilter-persistent save
sudo /etc/init.d/netfilter-persistent reload
```

## サーバー起動

ここまで完了したら，OpenVPNのサーバーを起動します。`/etc/openvpn/server-example.conf`という設定ファイルを作成した場合，以下のようにして起動します。

```bash
sudo systemctl start openvpn@server-example
```

ここでエラーが出た場合は，`journalctl -xeu openvpn@server-tcp.service`を実行してみたり，OpenVPNのログファイル`/var/log/openvpn/openvpn.log`などを確認してみてください。

## iOSなどからの接続

OpenVPN Connectアプリをインストールします。

そして作成したovpnファイルを，airdropなどの好きな手段で転送し，OpenVPN Connectアプリで開きます。

プロファイル名を入力して，接続・通信できれば完了です。

@[card](https://test-ipv6.com/)
などをみたり，ローカルのサービスにアクセスするなどしてニヤニヤしましょう。

## おわりに

ここまでで，ひとまずOpenVPNサーバーの設定は完了です。
無効な証明書を使えなくするには，CRLを使う必要があるのですが，これはまた別の機会にやってみたいと思います。

ちなみに，この記事を書いている過程で，ブレーカーが一度落ちてディスクがやられてしまい，このフローを試したのですが，一通り動作することを幸い（？）確認しなおすことができました。

裏で色々な試行錯誤はあったので，時間があれば追記もしくは別記事にしたいと思います。
