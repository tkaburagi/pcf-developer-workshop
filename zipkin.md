# Zipkinで分散トレーシングをする
ここでは[Zipkin](https://zipkin.io/)を使って分散トレーシングを行う設定を行います。ZipkinはTwitter社によって開発されたOSSの分散トレーシングシステムです。

## アプリケーションの修正
`ui-tkaburagi`と`api-tkaburagi`に以下の設定を追加し再度pushします。`spring.application.name`はアプリごとに変える必要があるので注意してください。

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
`https://zipkin.sys.pas.ik.am/`にブラウザでアクセスしてください。

検索ボックスから以下のように自分のアプリ名でフィルタリングできるように設定してください。
![image](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/img/zipkin-2.png)

以下のようなビューが表示されるはずです。
![image](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/img/zipkin-1.png)

各TraceやSpanを展開すると詳細を見ることができます。
![image](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/img/zipkin-3.png)