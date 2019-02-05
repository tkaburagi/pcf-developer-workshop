# Using Pivotal Cloud Cache

```shell 
cf marketplace
```

```shell
cf create-service p-cloudcache extra-small pcc
cf create-service-key pcc pcc-svc-key
```

`cf create-servie`は少し時間がかかります。

```shell
cf service-key pcc pcc-svc-key
```

```shell
$ gfsh
$ gfsh> connect --use-http=true --use-ssl --url=<GFSH_URL> --user=***** --password=*****
$ gfsh> create region --name=book --type=REPLICATE
$ gfsh> list regions
```
`pom.xml`に以下を追記します。
```xml
<dependency>
    <groupId>org.springframework.geode</groupId>
    <artifactId>spring-geode-starter</artifactId>
</dependency>
```
`Book.java` を下記のように編集します。

```java
@Entity
@Table(name = "book")
@Region(name = "book")
public class Book {
//省略
}
```

`CloudCacheConfig.java`を追加し、下記のように編集します。

