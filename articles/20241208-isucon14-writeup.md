---
title: "学生最後のISUCON14に一人で出て30位以内に入ったと思ったら失格だった！😇"
emoji: "💣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["isucon", "go", "nginx", "mysql"]
published: false
---

こんにちは。いや，こんいす。
東京理科大学工学部電気工学科の学部4年，25卒で株式会社LayerX入社予定の[@TAK848](https://github.com/TAK848)です。
12月8日に開催されたISUCON14に，学生最後の記念として（？），1人でチーム名「tak」の「tak」として参加しました。

計測ログを消してしまったので，誰向けかはさておき忘れないうちにWrite-Upを書き殴っておきます。

## 結果

28677点を終了前に出していたんですが，28100点くらいの点数が30位以内の賞に選ばれているのにもかかわらず，結果発表は配信のTop30には名前無し。
![Top30](/images/20241208-isucon14-writeup/top30.png)

つまり失格ということですね！！！😅

ただ結果発表後のLeaderboardを見たところ，27位，学生5位でした！
![Leaderboard](/images/20241208-isucon14-writeup/leaderboard.png)

いや，失格は失格です！

対戦ありがとうございました！！精進します！！

## 失格の原因

計測に，ISUCON12・13優勝してISUCON14で作問をしているNaruseJunチーム特製のツール，pproteinを使用していました。
すごく使いやすい計測ツールでした。ありがとうございました！来年も使えたら使いたい。

ただこの計測サーバーのインスタンスを同じVPC内に配置していたことが原因で，多分失格になったっぽいです。
というのも，集計終了後にenvcheckを試してみたら，ちゃんと失敗したため…。
pproteinのインスタンスを削除してから試したところ成功。これだ…。

![Envcheck](/images/20241208-isucon14-writeup/envcheck.png)

pproteinのデプロイ記事に，Cloudformationで同じVPCに！と書いてあったので完全に油断してEC2起動したままでした…残念

## やったこと

気を取り直してやったことを書いていきます！
とはいっても，計測関係をpprotein用のEC2に全て載せていて，Envcheck検証時に勢いでpproteinのインスタンスのバックアップをとらずに削除してしまったので，スコアやメトリクス記録がありません！！
なんとなく思い出しながら書いていきます！

スコア推移はこんな感じでした。
![Scoreboard](/images/20241208-isucon14-writeup/scoreboard.png)

リポジトリはこちらです。
https://github.com/TAK848/isucon14

### 爆速デプロイ&焦り

portalとcloudformationのデプロイページをどっちもArcで開いておいて，爆速デプロイを完了しました。
そしてportalからベンチマークを実行したんですが，何回やってもスコアが0で最初焦りました。
ベンチの番号，#1をとっていたので，正常に動いていたら一瞬一位だったのかなとか思ったり。

何分たっても誰もスコアあがらないからそういうことか。と思ったあたりで質問・回答コーナーにバグだと出たので，とりあえず計測環境のセットアップをはじめました。

### pproteinのセットアップ

前日に，[@KOBA789](https://github.com/koba789)さん作の[ISUNARABE](https://isunarabe.org)でISUCON13の素振りをしようと思い立ちセットアップしました。
https://diary.hatenablog.jp/entry/2023/09/27/002428

ですが，pproteinのサーバーのAMIだけ作成してほぼ何も素振りせず終わっていました。ISUNARABE，本番さながら演習ができてめっちゃ良いです。推し。
ここまで前日。

とりあえず同じVPCにそのAMIを使ってpproteinをホストしました。
以下のページが良い感じにまとまっていて，基本コピペで動きました。
https://hackmd.io/@to-hutohu/pprotein-getting-started

https://zenn.dev/team_soda/articles/20231206000000#pprotein-agent%E3%81%AEservice%E3%82%92%E8%B5%B7%E5%8B%95

ただ今回，chiが使われていて，[echo・gin・gorilla/mux](https://github.com/kaz/pprotein/tree/main/integration)はあったけどchiのやつはintegrationが用意されていなかったので，雑に書きました(ChatGPTが)。

https://github.com/TAK848/isucon14/blob/main/go/chiinteg/chiinteg.go

良い感じに設定したり，openapiをChatGPTにぶん投げてalpのpattern書いてもらったりしていたら，計測環境ができました。ただ既に10:40でした。素振りしてれば半分にはできたかな…

### マニュアル読みつつGPTs作った

とりあえずドメインに詳しいGPTs作って，分からないこと雑に聞いていました。
![GPTs](/images/20241208-isucon14-writeup/gpts.png)
![GPTs2](/images/20241208-isucon14-writeup/gpts2.png)
べんり。

### インデックスを貼っていた

pproteinからslpの結果をみて，インデックスを貼っていきました。
https://github.com/TAK848/isucon14/commit/86a74021ad9dcba892ea7d04c6f140745f309c2f
https://github.com/TAK848/isucon14/commit/4614a6bb69538f8f42f18b4799258bff65236358
https://github.com/TAK848/isucon14/commit/5c033de7d950ec5db0dc583155f861277627924d
https://github.com/TAK848/isucon14/commit/b8c566978d93151098b173fb57c9c843754d3fea
https://github.com/TAK848/isucon14/commit/3bcbcc7c08138fc37c7d1b70474482c529ef0e1c

また，ISUCON特権でPreparedStatementをオフにしました。
https://github.com/TAK848/isucon14/commit/771482c3a011fdebf7af8219f968bc85073eb9cc

もうこの時点で13時くらい。（コミットログによると）
4000点くらいだったみたいです。
定石クリアに3時間。練習してれば2時間にはできた気が…。

### けたたましいnotificationをなだめた

alpの結果を見るとapp・chairのnotificationエンドポイントがボトルネックになっていました。
ブラウザからアプリを開いてNetworkタブを見たら，notoficationエンドポイントにけたたましい勢いでDOS攻撃していて最初絶望しました。

ただ，マニュアルにSSE（Server-Sent Events）を使って良いと書いてあったので，調べながら実装をはじめたんですが，ChatGPTとも話して進めながらも何も知らなすぎてこの時間でやるのは無理じゃね？と思ってきました。
そこでよく見ると，通知のリクエスト間隔をレスポンスのRetryAfterから設定できることを知り，みたら30msに固定になっていたので一気に10倍の300msにしました。そしたら最終的にスコアが7000くらいへと1.7倍くらいになりました。
ISUCONのことだからどうせこれやらなくても，他潰すとスコアあがるだろうなーと思い，次に行くことにしました。

### 観光名所のSQLを消した

ownerのchairs取得時に，過去位置をすべて参照する豪快なクエリが走っていて，slp（slow queryログ）のトップに来ていたので，これをシンプルな1テーブル参照に変更しました。
もともとはchair_locationsテーブルで履歴全体から距離を算出していましたが，その計算結果をchairsテーブル側にキャッシュ的に持たせることで，履歴参照を不要にしました。
chair_locationsの履歴を，total_distanceとtotal_distance_updated_at取得のためだけに使われていたので，chairsテーブルにlongtitudeとlatitude，total_distanceとtotal_distance_updated_atを全て追加し，もともとのマスタデータを以下のクエリでごにょっと更新してやることで，スコアが9000くらいになりました。

やりながら多少，別テーブルにしても良かった感を感じていました。chairsのmiddlewareで不要なときも取得するようになっていたので…。

:::details 雑に初期データをUPDATEして入れたクエリ
```sql
UPDATE chairs
JOIN (
	SELECT
		id,
		owner_id,
		name,
		access_token,
		model,
		is_active,
		created_at,
		updated_at,
		IFNULL(distance_table.total_distance, 0) AS total_distance,
		distance_table.total_distance_updated_at
	FROM
		chairs
		LEFT JOIN (
			SELECT
				chair_id,
				SUM(IFNULL(distance, 0)) AS total_distance,
				MAX(created_at) AS total_distance_updated_at
			FROM
				(
					SELECT
						chair_id,
						created_at,
						ABS(
							latitude - LAG(latitude) OVER (
								PARTITION BY
									chair_id
								ORDER BY
									created_at
							)
						) + ABS(
							longitude - LAG(longitude) OVER (
								PARTITION BY
									chair_id
								ORDER BY
									created_at
							)
						) AS distance
					FROM
						chair_locations
				) tmp
			GROUP BY
				chair_id
		) distance_table ON distance_table.chair_id = chairs.id
) AS summary ON summary.id = chairs.id
SET
	chairs.total_distance = summary.total_distance,
	chairs.total_distance_updated_at = summary.total_distance_updated_at;
```

```sql
UPDATE chairs
JOIN (
    SELECT 
        cl.chair_id, 
        cl.longitude, 
        cl.latitude
    FROM 
        chair_locations cl
    JOIN (
        SELECT 
            chair_id, 
            MAX(created_at) AS max_created_at
        FROM 
            chair_locations
        GROUP BY 
            chair_id
    ) latest ON cl.chair_id = latest.chair_id AND cl.created_at = latest.max_created_at
) latest_location ON chairs.id = latest_location.chair_id
SET 
    chairs.longitude = latest_location.longitude,
    chairs.latitude = latest_location.latitude;
```
:::

唯一ブランチ切りました。一人だったんで…。
https://github.com/TAK848/isucon14/pull/1/files
https://github.com/TAK848/isucon14/commit/315ee4a7a288bd30ca648fa4eff566fc4513dc30

### 重いエンドポイントチューニング

https://github.com/TAK848/isucon14/commit/9f6ce913e7b6c4e1ab65d2b31e893778e51e78ae

で必要な時だけupdateかけるようにするなどのチューニングをしました。

その後さらにslpを見てindex足したりMaxConn調整したりしました。

### db分割

17:30になり，流石にわけんとまずいとなり，dbを別サーバーにしました。
これでスコアが23,000くらいになりました。
その後，internalエンドポイントの処理を別サーバーになんとなく分けたつもりができてませんでした。

### ログなどを切る

ログなどを切ったら28000が出ました。
ただしこれは最終的なもので，notificationのRetryAfterを短くするなどするとベンチが失敗することが希に発生するようになったので，600msに上げた結果です。
焦ってたのでベンチのログをよく覚えていません。
このスコアで終了を迎えました。

### その他: PGO

pproteinのパワーでpgoのデータを簡単に取れたんですが，スコアがそもそも安定しきって無くて効果が見えきらなかった&3台構成にして，準備不足によりコマンド打つのがめんどかったのでやりませんでした。

## こころのこり

- マッチングのアルゴリズム，DB負荷を無駄に上げていただけだったのでアプリ演算に変えたかった。距離/speedが一番近いやつをマッチするようにしたかった。DBに謎クエリ投げてボトルネックになってたし。
- マッチング間隔，雑に0.5s->0.25にしたらスコア少しあがって満足してたけど，もう少し下げても良かったかも？感想戦で試さないと分からない。
- SSE，やろうとおもったけど断念した。
  - DOS攻撃，Retry時間レスポンスで変えられることに気付いて広げたらめっちゃ点数あがってそのままでﾖｼ!と判断した。一人だしまあ間違ってはいなかったかとは思う
- Envcheckを実行するようアナウンスあったので，余裕のあるうちにやっておけば良かった。


## 個人的に便利だったアプリやツール

アプリやツールの活用により，色々効率化できました。

### tableplus

https://tableplus.com/

DBのスキーマやデータをサッと見たり編集したり書き出したりSQL実行したりと何にでも便利です。

![tableplus](/images/20241208-isucon14-writeup/tableplus.png)

### SSH Config Editor

https://setapp.com/apps/ssh-config-editor

`.ssh/config` をGUIで編集できて，便利でした。
良い感じに編集してcmd+sで保存してくれます。IPアドレス以外埋めておきました。Port Forwardingのオプションとかいちいち覚えてられないので，すごく良かった。

pprotein，普通にローカルでホストしてRemote Forwardingすればよかった。そしたら順位出てたかな〜

![ssh-config-editor](/images/20241208-isucon14-writeup/sshconfigeditor.png)

![ssh-config-editor2](/images/20241208-isucon14-writeup/sshconfigeditor2.png)

### chatgpt

いわずもがなです。必需品。上述のドメイン知識答えてくれるGPTsからo1, o1-miniまで。たくさん活用しました。

### Cursor

複数行補完最強。

### Arc

今更感ありますが。
よく見るページは全てPinしておくと便利でした。
![Arc](/images/20241208-isucon14-writeup/arc.png)

### Warp

Warpである必要はありませんが，1タブに全てのサーバーのシェルを出して，入力をSynchronize Inputで同期していました。これにより，全てのサーバーで設定まで同じ物を保っていたので簡単にサーバー分割できました。
![warp](/images/20241208-isucon14-writeup/warp.gif)

```bash
git pull && go build -o isuride && sudo systemctl restart isuride-go
```

や

```bash
wget http://192.168.0.35:9000/api/pprof/data/latest -O ~/pgo.pb.gz && git pull && go build -o isuride -pgo=/home/isucon/pgo.pb.gz && sudo systemctl restart isuride-go
```

を，Warpの機能で何文字か売ってctrl+e，enterで実行することで，Makefile作成を完全にサボりました。makeコマンド一切使わなかった…準備不足。
/etc/以下のファイルはgit管理もせず，neovimで同時編集すれば全て同じ！と言って雑管理していました。
まあ一人参加でしたし…（main直push万歳でやった）。

## 謝辞

ISUCON12で先輩に誘われて初参加し，そこから技術力の向上に留まらず色んな縁に恵まれました。
非常に思い入れ深いイベントです。
今回は新たに失格になるという思い出もできました。最近忙しすぎて，今回はサクッと出て終わるつもりが，失格前のスコアが30位以内だったのは嬉しかったです。
とはいえ失格でくやしい！改め精進します！！

pproteinというすごいツールを公開してくださり作問もしてくださったNaruseJunチーム，そしてポケットサインの皆さんはじめ，運営の皆さん，参加者の皆さん，ありがとうございました！！
問題，結構分岐ポイントがあってボリューミーで面白かったです。当然ですが，複数人で出たら効率良いだろうなと思いました。
おついす〜
