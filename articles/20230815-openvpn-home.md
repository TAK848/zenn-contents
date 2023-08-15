---
title: "OpenVPNをLet's Encryptのサーバー証明書とオレオレCAのクライアント証明書で設定[Ubuntu 22.04 LTS]"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["OpenVPN", "LetsEncrypt", "Ubuntu", "Route53"]
published: false
---

## はじめに

手元にあった，昔のMacBook ProのSSDが故障したため，これを機に外付けSSDを接続・Ubuntuをインストールし，OpenVPNサーバーを立ててみました。
過程で，サーバーの証明書にオレオレCAを使うのはやだなーと思い，Let's EncryptをDNS認証で使用することにしました。DNSはRoute53を使用しているので，Route53のみの権限を用意したIAMを用意して，これを使っていきます。
IPv4・IPv6双方のルーティングの設定も行いました。

これらの備忘録として，Ubuntuのインストールも含め，概要をこちらにまとめておきます。

## Ubuntuのインストール

## OpenVPNのインストール

## 証明書の作成

### クライアント証明書

### サーバー証明書

## OpenVPNの設定

### サーバー側の設定

### クライアント用ovpnファイルの作成

## ルーティングの設定

### IPv4

### IPv6

## おわりに
