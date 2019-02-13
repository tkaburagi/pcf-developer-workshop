# PCFの基本的な機能を試す
ここではPCFのプラットフォームとしての代表的な機能である、`Auto Healing`, `Auto Scale`と`Blue-Green Deploy`の三つを試してみます。

## Auto Healing (アプリ)
PCFのオートヒーリングには3つのレイヤがあります。コンテナ、VMとプラットフォームプロセスです。ここでは前者の二つを試してみましょう。

### アプリケーションをスケールアウトする
下準備としてアプリケーションをスケールアウトします。
その前に以下の二つのターミナルを開いてアプリの振る舞いを監視出来るようにして置きましょう

#### ターミナル1
```shell
watch -n 1 'curl https://api-tkaburagi.apps.pcf.pcflab.jp | jq'
```
watchがない場合、`while true;do curl -s ttps://api-tkaburagi.apps.pcfone.io/;echo;sleep 1;done`でも大丈夫です。
このターミナルではアプリケーションのインデックス数やリクエストの負荷分散などの状況を見ます。

#### ターミナル2
```shell
watch -n 1 cf app
```
watchがない場合は上記同様です。このターミナルではアプリケーションの状態を見ます。

別のターミナルでアプリケーションをスケールアウトします。
```shell
cf scale -i 5
```
ターミナル1で負荷分散をしていること、ターミナル2でアプリケーションが5インスタンス`running`になっていることを確認しましょう。

次に以下の設定をアプリケーションに加え、Spring Bootのシャットダウンエンドポイントを有効にします。
```shell
cf set-env api-tkaburagi management.endpoints.web.exposure.include shutdown
cf set-env api-tkaburagi management.endpoint.shutdown.enabled true
cf restart api-tkaburagi
```

アプリが再起動し、0-4の5インスタンスにリクエストが振られていることを確認し、以下のコマンドを実行してください。このコマンドで5つのインスタンスのうち、一つが停止します。ターミナル1,2をよく見ながら何が起こっているかを確認して見てください。
```console
curl -X POST https://api-tkaburagi.apps.pcf.pcflab.jp/actuator/shutdown --insecure
{"message":"Shutting down, bye..."}
```
残りのインスタンスからのみレスポンスがあることしばらくすると自動でアプリが復旧することが確認できるでしょう。見逃した場合は何度か実行して見てください。

**ここまで完了したら進捗シートにチェックをしてください。**

## Auto Healing (VM)
次に、VMを一つ削除しVMとアプリケーションコンテナの自動復旧の様子を見てみます。 現在3インスタンスあるそれぞれのインデックス番号のコンテナがホストされているVMを各自ホワイトボードに記述してください。

記述後講師がVMを一つ削除します。

削除中、コンテナとレスポンスがどのような振る舞いになるかターミナル1,2をよく見てください。

**ここまで完了したら進捗シートにチェックをしてください。**

## PCF Auto-scalerの利用
PCFのライセンスに付随するApp Autoscalerを利用するとアプリケーションのパフォーマンスやカレンダーに基づいてアプリケーションの自動スケールアウト、スケールインを実現できます。


Apps Managerを開き、`api-tkaburagi`の画面に移動し、Autoscalingをオンにします。
![image](https://storage.googleapis.com/pcf-workshop/autoscale.png)

`Manage Autoscaling`を押し、Autoscalerの設定を実施します。

1. 最大と最小のインスタンス数

![image](https://storage.googleapis.com/pcf-workshop/autoscale2.png)

2. CPUの閾値の設定

`SCALING RULES`の`EDIT`を選択し`ADD RULE`で`CPU Utilization`を選びます。
![image](https://storage.googleapis.com/pcf-workshop/autoscale3.png)

設定が完了したら`SAVE`を押します。

![image](https://storage.googleapis.com/pcf-workshop/autoscale4.png)

30秒に一度CPU利用率がチェックされ、閾値を下回るとインスタンスが一つずつ減っていくのでその様子を下記のコマンドで確認します。

```console
cf app api-tkaburagi | grep "%"
#0   running   2018-04-24T07:07:36Z   0.3%   297.1M of 1G   138.1M of 1G
```

数秒に一度実行するとスケールインする様子がわかります。

**ここまで完了したら進捗シートにチェックをしてください。**

## Blue-Green Deploy
[Blue-Greenデプロイ](http://martinfowler.com/bliki/BlueGreenDeployment.html)はBlueとGreenとよばれる2つの環境を用意して、ルーティングを切り替えることによりダウンタイムリスクを低減させる手法です。

Cloud Foundryでは`cf map-route`、`umnap-route`コマンドによりルーティングの設定を行うことでBlue-Greenデプロイを実現できます。

ここではwatchよりwhileでやったほうがわかりやすいので、ターミナル1をwatchで開いている方は一旦止めて以下のコマンドでレスポンスを監視します。
```shell
while true; do curl -s http://api-tkaburagi.apps.pcf.pcflab.jp; echo; sleep 1;done
```

### アプリケーションの修正
```java
jsonObject.put("message","Helloworld V2");
```
アプリケーションの上記の行の文字列を`V1`から`V2`に変更します。(どのような変更でもいいです。)

アプリを変更したらmanifest.ymlのあるディレクトリにいることを確認して、アプリをデプロイします。
```shell
./mvnw package -Dmaven.test.skip=true && cf push api-tkaburagi-green # manifest内のapplication nameをoverride
```

新バージョンにアクセスして、正常に動作していることを確認してください。

```shell
curl -s http://api-tkaburagi-green.apps.pcf.pcflab.jp
```

### アプリケーションをバージョンアップする
`cf map-route api-tkaburagi-green <Domain> -n <Hostname>`で`api-tkaburagi.apps.pcf.pcflab.jp`へのリクエストが`api-tkaburagi-green`にルーティングされるようにします。

```shell
$ cf map-route api-tkaburagi-green apps.pcf.pcflab.jp -n api-tkaburagi
```

2つのアプリケーションに対して`api-tkaburagi.apps.pcf.pcflab.jp`でアクセスできるため、コンソールを見るとcurlの結果はV1とV2の両方が表示されるようになります。

`map-route`とは反対の`unmap-route`コマンドで`api-tkaburagi`へのルーティングを外します。

``` shell
$ cf unmap-route api-tkaburagi apps.pcf.pcflab.jp -n api-tkaburagi
```

これで`V2`のみが出力されるようになりました。


`V2`に問題がなければ、旧バージョンを削除し、`api-tkaburagi-green.apps.pcf.pcflab.jp`のルーティングも削除します。

``` console
$ cf delete api-tkaburagi
$ cf unmap-route api-tkaburagi-green apps.pcf.pcflab.jp -n api-tkaburagi-green
```


`api-tkaburagi.apps.pcf.pcflab.jp`にアクセスし続けていましたが、404エラーなどが発生することなく`V1`から`V2`へ移行することができました。

``` console
$ cf apps
Getting apps in org tmaki / space development as ****@gmail.com...
OK

name                requested state   instances   memory   disk   urls   
api-tkaburagi-green   started           1/1         1G       1G     api-tkaburagi.apps.pcf.pcflab.jp
```

アプリケーション名を`api-tkaburagi`に戻しましょう。

``` console
$ cf rename api-tkaburagi-green api-tkaburagi
```

これで元の通りです。

``` console
$ cf apps
Getting apps in org tmaki / space development as ****@gmail.com...
OK

name          requested state   instances   memory   disk   urls   
api-tkaburagi   started           1/1         1G       1G     api-tkaburagi.apps.pcf.pcflab.jp
```

**ここまで完了したら進捗シートにチェックをしてください。**
