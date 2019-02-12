# Micrometer とPrometheusを使ってメトリクスを監視する

[Micrometer](https://micrometer.io/)はベンダーニュートラルなメトリクス収集のためのエンドポイントを提供するツールです。MicrometerはPivotalが開発しています。Prometheus, Datadogなど様々なライブラリがMicrometerに対応しています。今回はPrometheusを利用します。

まず、`pom.xml`に以下の依存ライブラリ情報を追加します。
```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

次に`application.properties`に以下を追記します。
```properties
management.endpoints.web.exposure.include=prometheus
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
`https://prometheus.sys.pas.ik.am`をブラウザで開きます。


# Micrometer とPCF Metricsを使ってメトリクスを監視する

# ELKでログモニタリングする
//TODO