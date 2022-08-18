---
title: "Info.plistのミスでFlutter×Firebase×Googleログイン時にクラッシュした問題の解決"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter","firebase","Xcode"]
published: true
---

# 発生した問題
Flutter勉強中，以下の記事
https://zenn.dev/kazutxt/books/flutter_practice_introduction/viewer/chapter4_authentication#google%E3%82%A2%E3%82%AB%E3%82%A6%E3%83%B3%E3%83%88%E3%81%AB%E3%82%88%E3%82%8B%E8%AA%8D%E8%A8%BC
を参考に，手順通りFirebaseを使用したGoogleログインを試していたつもりだったのが，iOSでの検証で
```
*** Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: 'Your app is missing support for the following URL schemes: com.googleusercontent.apps.xxxxxxxxxxxxxxxxxx'
```
とエラーが出てクラッシュしました。（Androidでは正常に動作した）

# 解決方法
Info.plistに`REVERSED_CLIENT_ID`をきちんと`CFBundleURLSchemes`に設定したはずなのに何故だろうと調べていたところ，GithubのIssue
https://github.com/googlesamples/google-services/issues/81#issuecomment-302976984
を発見しました。これに従い直接XcodeからURLの設定をしたところ，正常にログインできました！
![](/images/flutter-firebase-google-url-error/settings.png)

# 問題があった点の確認
Info.plistを再度確認してみると，
```
<dict>
    ...
    <array>
        <dict>
            <key>CFBundleTypeRole</key>
            <string>Editor</string>
            <key>CFBundleURLSchemes</key>
            <array>
                <string>com.googleusercontent.apps.xxxxxxxxxxxxxxxxxxxxx</string>
            </array>
        </dict>
    </array>
    ...
</dict>
```
のように生成されていましたが，僕が手動で行った設定だと，
```
<dict>
    ...
    <key>CFBundleTypeRole</key>
    <string>Editor</string>
    <key>CFBundleURLSchemes</key>
    <array>
        <string>com.googleusercontent.apps.xxxxxxxxxxxxxxxxxxxxx</string>
    </array>
    ...
</dict>
```
このように全体をarrayで囲うのを忘れてしまっていて，読み込まれていなかったことが原因でした......

# 謝辞

無料でここまで充実した記事を公開していただいていることに，感謝するばかりです...
https://zenn.dev/kazutxt/books/flutter_practice_introduction
