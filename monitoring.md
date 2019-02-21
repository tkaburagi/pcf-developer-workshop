# MicrometerとPrometheusを使ってメトリクスを監視する

[Micrometer](https://micrometer.io/)はベンダーニュートラルなメトリクス収集のためのエンドポイントを提供するツールです。MicrometerはPivotalが開発しています。Prometheus, Datadogなど様々なライブラリがMicrometerに対応しています。今回はPrometheusを利用します。

まず、`api-tkaburagi`のアプリの`pom.xml`に以下の依存ライブラリ情報を追加します。
```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

次に`cf set-env`でアプリに環境変数をセットしエンドポイントを有効にします。
```shell
cf set-env api-tkaburagi management.endpoints.web.exposure.include shutdown,env,prometheus
```

設定はこれだけです。アプリケーションをpushし、エンドポイントにアクセスをしてみます。
```shell
./mvnw clean package -DskipTests=true && cf push
```

```console
$ curl https://api-tkaburagi.apps.pcfone.io/actuator/prometheus

Counting objects: 3, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 882 bytes | 882.00 KiB/s, done.
Total 3 (delta 2), reused 0 (delta 0)
remote: Resolving deltas: 100% (2/2), completed with 2 local objects.
To https://github.com/tkaburagi/pcf-developer-workshop.git
   c495911..eb8bbdb  master -> master
➜  pcf-developer-workshop git:(master) curl https://api-tkaburagi.apps.pcfone.io/actuator/prometheus

logback_events_total{level="warn",} 6.0
logback_events_total{level="debug",} 0.0
logback_events_total{level="error",} 0.0
logback_events_total{level="trace",} 0.0
logback_events_total{level="info",} 19.0
//以下省略
```

## Prometheusにログインする
今回はPrometheus側はすでに設定済みのためすでにメトリクスの収集が始まっています。参考までに、このような設定を行っています。
```yaml
value:
  job_name: api-tkaburagi
  metrics_path: /actuator/prometheus
  scheme: http
  scrape_interval: 20s
  scrape_timeout: 5s
  static_configs:
  - labels:
      cf_app_name: api-tkaburagi
    targets:
    - api-tkaburagi.apps.pcf.pcflab.jp
```

今回は事前に全員分の設定をしていますが、[promregator](https://github.com/promregator/promregator)を利用すると個別で設定する必要がなくなります。
`https://prometheus.sys.pas.ik.am`をブラウザで開きます。

JVM関連のメトリクスを選んでグラフに表示されることを確認してください。
![image](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/img/prometheus.png)

# Circuit BreakerをMicrometerとPrometheusで監視する
`ui-tkaburagi`のアプリの`pom.xml`に以下のエントリを追加します。
```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

次に`application.properties`に以下を追加します。
```properties
management.endpoints.web.exposure.include=*
```

アプリケーションをpushします。
```shell
$ ./mvnw clean package -DskipTests=true
$ cf push
$ curl https://ui-tkaburagi.apps.pcf.pcflab.jp/actuator/hystrix.stream --insecure
```
`curl`コマンドのターミナルはそのままにしておいてください。一度Webブラウザで`http://ui-tkaburagi.apps.pcf.pcflab.jp/?id=1`にアクセスし、上の`curl`の出力を確認しましょう。Actuatorの`hystrix.stream`エンドポイントでCircuit Breakerの状態がストリーミングされます。次に以下のコマンドを実行してください。HystrixのメトリクスをMicrometerにバインドする設定が`AutoConfigured`され、`prometheus`エンドポイントからメトリクスを確認できます。
```console
$ curl ui-tkaburagi.apps.pcf.pcflab.jp/actuator/prometheus
# ...
# HELP tomcat_sessions_created_sessions_total
# TYPE tomcat_sessions_created_sessions_total counter
tomcat_sessions_created_sessions_total 0.0
# HELP jvm_gc_live_data_size_bytes Size of old generation memory pool after a full GC
# TYPE jvm_gc_live_data_size_bytes gauge
jvm_gc_live_data_size_bytes 4.7678488E7
# HELP hystrix_latency_total_seconds_max
# TYPE hystrix_latency_total_seconds_max gauge
hystrix_latency_total_seconds_max{group="UiService",key="dummy",} 0.0
# HELP hystrix_latency_total_seconds
# TYPE hystrix_latency_total_seconds summary
hystrix_latency_total_seconds_count{group="UiService",key="dummy",} 2.0
hystrix_latency_total_seconds_sum{group="UiService",key="dummy",} 0.126
# HELP jvm_threads_states_threads The current number of threads having NEW state
# ...
```

最後にPrometheusのダッシュボードでメトリクスを確認してみます。

`https://prometheus.sys.pas.ik.am`をブラウザで開きます。
先ほど同様にHystrix関連のメトリクスを見つけてグラフに表示させてみましょう。
![image](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/img/hystrix-prome-1.png)


**ここまで完了したら進捗シートにチェックをしてください。**

# ELKでログモニタリングする
`https://kibana.sys.pas.ik.am/`をブラウザで開きます。
![image](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/img/kibana-1.png)

`Add a Filter+`を選択し以下のフィルタの設定をします。アプリ名はご自身の環境に合わせてください。
![image](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/img/kibana-2.png)

自分のアプリケーションだけにフィルタリングされアプリログも表示されることを確認してください。
![image](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/img/kibana-3.png)

時間のある方は`org_name`や`space_name`など色々なフィルタを試してみてください。

**ここまで完了したら進捗シートにチェックをしてください。**

## Advanced
時間のある方`PCF Metrics`を試してみましょう`https://metrics.sys.pcf.pcflab.jp`にアクセスします。
tomcat_sessions_created_sessions_total 0.0て
tomcat_sessions_created_sessions_total 0.0
tomcat_sessions_created_sessions_total 0.0
