# APMを利用してパフォーマンスモニタリングする
ここでは`Buildpacks`の`Dynatrace`連携機能を利用してパフォーマンスモニタリングの設定を行います。

## Dynatraceの設定
[Dynatraceマニュアル](https://www.dynatrace.com/support/help/cloud-platforms/cloud-foundry/application-only/deploy-oneagent-on-cloud-foundry-for-application-only-monitoring/)の
`Generate PaaS token`を参考にDynatraceのトークンを取得してください。

取得したトークンはメモ帳に残しておきましょう。

**ここまで完了したら進捗シートにチェックをしてください。**

## User Provided Serviceの作成
```shell
cf cups dynatrace-service -p "environmentid, apitoken, apiurl"
cf bind-service api-tkaburagi dynatrace-service
cf bind-service ui-tkaburagi dynatrace-service
```

`environmentid`, `apitoken`, `apiurl`には先ほどメモした内容を入力します。

## Dynatrace Dashboardへアクセス
自分のアカウントでDynatraceにログインしてください。


