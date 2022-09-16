---
title: "MySQLのDockerイメージがARM64にいつの間にか対応していた"
emoji: "🏃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["docker","mysql","arm64","AppleSilicon","M1"]
published: true
---

# 今まで
つい最近までは，MySQLの公式DockerイメージはプラットフォームがAMD64のみで，ARM64には対応していませんでした。
そのため，普通にイメージをpullしようとしても，ARM64版のイメージが存在しないよ！とエラーが出てきていました。実際，そのような記事がたくさん…！
ゆえに，MySQLの公式イメージをM1 macで使用したい時は，例えば[こちら](https://gihyo.jp/dev/serial/01/mysql-road-construction-news/0167)の記事などのように，
```
services:
  db:
    platform: linux/amd64
    image: mysql:8.0
```
のようにして`platform`を明記する（これによりRosetta2を介して起動していたのかな？）か，mysql serverなどの他のイメージを使用する必要がありました。

# 最近になって…
個人的に，最近コミュニティメンバーとISUCONに参戦したのもあって，DBにPostgreSQLではなくチューニングに慣れたMySQLを使ってみたいなと思ってふと調べてみたら，MySQLの公式イメージがARM64に対応していました。
https://hub.docker.com/_/mysql/tags
![ARM64に対応！](/images/docker-mysql-arm64/arm64.png)
正確に書くと，MySQLイメージにはOracle Linuxとdebianがあるのですが，そのうちのOracle版が対応してくれたみたいです。

全く気づいていなかったのですが，6ヶ月前の8.0.28ですでに対応していたようです。
![6ヶ月前](/images/docker-mysql-arm64/oldest.png)
これからは，
```
services:
  db:
    image: mysql:8.0
```
これだけで良くなりますね。Apple siliconのmacユーザーや，その他のARM64ユーザーにとっては嬉しい改善だと思います。

これによって，今までは使用できないことはないもののネイティブ環境としては動かせず躊躇していましたが，その必要はなくなりました。
そのため現在ハッカソンでの環境構築で喜んで使用させていただいています！

# まとめ
- 今まではMySQLの公式イメージはARM64に対応しておらず，`platform`を明記する必要があった
- 最近になって，MySQLの公式イメージが8.0.28よりARM64に対応した
- これからは，Apple siliconのmacだろうと普通にMySQLをDockerで動かせるようになった，嬉しい
