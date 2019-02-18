# pcf-developer-workshopアジェンダ
* [簡単なアプリをデプロイする](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/deploy-simple-spring-boot.md)
* [PCFの基本的な機能を利用する](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/pcf-basic.md)
* [バックエンドデータベース(MySQL)を利用する](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/backend-mysql.md)
* [キャッシュストア(Pivotal Cloud Cache)を利用する](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/backend-pcc.md)
* [アプリケーションをモニタリングする](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/monitoring.md)
* [Web UIアプリをデプロイする](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/deploy-ui.md)
* [Circuit Breakerを導入する](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/circuit-breaker.md)
* [セッションをPCCにキャッシュする](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/session-cache-pcc.md)
* [Zipkinを導入する](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/zipkin.md)
* [APMツールを利用する](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/apm.md)
* [ConcourseでCIする](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/concourse.md)


# PCF環境へのログイン
Pivotal Cloud Foundryの環境へログインします。

* APIエンドポイント: api.sys.pcf.pcflab.jp
* ユーザ名: <STUDENT_ID>
* パスワード: 講師より配布


``` shell
cf login -a api.sys.pcf.pcflab.jp  -o handson-<STUDENT_ID> -s development --skip-ssl-validation
```

ログインに成功したら下記のコマンドで確認し、正しく表示されればログイン成功です。

``` console
$ cf target
api endpoint:   https://api.sys.pcf.pcflab.jp
api version:    2.98.0
user:           admin
org:            handson-instructor
space:          development
```

今回は"handson-<STUDENT_ID>"というorganizationと"development"というspaceを利用します。

# ドキュメント
* [進捗確認シート](https://docs.google.com/spreadsheets/d/1pSoTiZ3pL4wzOwpTHmEixJxVbbtfOfNP5_bxAH6KU3w/edit?usp=sharing)
* [説明資料]()
