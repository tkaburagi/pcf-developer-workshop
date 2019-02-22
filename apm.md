# APMを利用してパフォーマンスモニタリングする
ここでは`Buildpacks`の`Dynatrace`連携機能を利用してパフォーマンスモニタリングの設定を行います。

## Dynatraceの設定
[Dynatraceマニュアル](https://www.dynatrace.com/support/help/cloud-platforms/cloud-foundry/application-only/deploy-oneagent-on-cloud-foundry-for-application-only-monitoring/)の
`Generate PaaS token`を参考にDynatraceのトークンを取得してください。

取得したトークンはメモ帳に残しておきましょう。

**ここまで完了したら進捗シートにチェックをしてください。**

## User Provided Serviceの作成
```console
$ cf cups dynatrace-service -p "environmentid, apitoken, apiurl" 
environmentid> **************
apitoken> **************
apiurl> **************
Creating user provided service dynatrace-service in org pivot-tkaburagi / space playground as tkaburagi@pivotal.io...
OK

$ cf bind-service api-tkaburagi dynatrace-service
$ cf bind-service ui-tkaburagi dynatrace-service
$ cf restage ui-tkaburagi
$ cf restage api-tkaburagi
```

対話式で入力する`environmentid`, `apitoken`, `apiurl`には先ほどメモした内容を入力します。`apiurl`は`https://<envid>.live.dynatrace.com/api`のような形になります。

## Dynatrace Dashboardへアクセス
自分のアカウントでDynatraceにログインしてください。

`Dashboard` > `Home`と進むとアプリケーションが監視対象になっていることがわかります。ドリルダウンしてアプリケーションのパフォーマンス情報を見て見ましょう。

![image](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/img/apm-1.png)

![image](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/img/apm-2.png)

**ここまで完了したら進捗シートにチェックをしてください。**
