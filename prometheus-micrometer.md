# Micrometer とPrometheusを使ってメトリクスを監視する

`pom.xml`に以下の依存ライブラリ情報を追加します。
```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

`application.properties`に以下を追記します。
```properties
management.endpoints.web.exposure.include=prometheus
```

```shell
./mvnw clean package -DskipTests=true && cf push
curl https://api-tkaburagi.apps.pcfone.io/actuator/prometheus
```

## Prometheusにログインする
`https://prometheus.sys.pas.ik.am`をブラウザで開きます。
