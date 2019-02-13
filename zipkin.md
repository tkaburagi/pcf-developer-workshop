# Zipkinで分散トレーシングをする
ここでは[Zipkin](https://zipkin.io/)を使って分散トレーシングを行う設定を行います。ZipkinはTwitter社によって開発されたOSSの分散トレーシングシステムです。

## アプリケーションの修正
`ui-tkaburagi`と`api-tkaburagi`に以下の設定を追加し再度pushします。

`application.properties`
```properties
spring.zipkin.base-url=https://zipkin.sys.pas.ik.am
spring.application.name=ui-tkaburagi
spring.sleuth.sampler.probability=1.0
```

`pom.xml`
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
    <version>2.1.0.RELEASE</version>
</dependency>
```

```shell
./mvnw clean package -DskipTests=true && cf push
```

設定は以上です。`ui-tkaburagi`と`api-tkaburagi`両方に実施してください。

## Zipkinダッシュボードにアクセス
