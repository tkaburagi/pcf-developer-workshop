# Using Pivotal Cloud Cache
ここではMarkeplace上のPivotal Cloud Cacheを利用し、Spring BootのアプリケーションからPivotal Cloud Cacheを扱います。GemFireなどのインメモリデータグリッドにアクセスするには`Look Aside`パターンと`Inline Cache`パターンの二つがありますが、今回は`Look Aside`パターンで実装します。

詳細は[こちら](https://content.pivotal.io/blog/an-introduction-to-look-aside-vs-inline-caching-patterns)を参考にしてください。

## PCCインスタンスを作成
```console 
$ cf marketplace
Getting services from marketplace in org pivot-tkaburagi / space playground as tkaburagi@pivotal.io...
OK

service                       plans                                                                                                                                                                                   description
apigee-edge                   org, microgateway, microgateway-coresident                                                                                                                                              Apigee Edge API Platform
app-autoscaler                standard                                                                                                                                                                                Scales bound applications in response to load
credhub                       default                                                                                                                                                                                 Stores configuration parameters securely in CredHub
metrics-forwarder             unlimited                                                                                                                                                                               Custom metrics service

p-cloudcache                  extra-small, dev-plan, small-footprint                                                                                                                                                  Pivotal Cloud Cache offers the ability to deploy a GemFire cluster as a service in Pivotal Cloud Foundry.

p-rabbitmq                    standard                                                                                                                                                                                RabbitMQ service to provide shared instances of this high-performance multi-protocol messaging broker.
p-redis                       shared-vm                                                                                                                                                                               Redis service to provide pre-provisioned instances configured as a datastore, running on a shared VM.

p.mysql                       db-small                                                                                                                                                                                Dedicated instances of MySQL
p.rabbitmq                    single-node-3.7                                                                                                                                                                         RabbitMQ service to provide dedicated instances of this high-performance multi-protocol messaging broker
p.redis                       cache-small                                                                                                                                                                             Redis service to provide on-demand dedicated instances configured as a cache.
postgresql-10-odb             standalone, general                                                                                                                                                                     Crunchy PostgreSQL 10 On-Demand Managed Service
```

```shell
$ cf create-service p-cloudcache dev-plan pcc
```

作成にはしばらく時間がかかりますので、待っている間次に進んでください。


## アプリケーションの修正
`pom.xml`に以下のエントリを追加します。`spring-boot-starter-web`は`<exclusions> ~ </exclusions>`を追記してください。
```xml
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-web</artifactId>
   <!-- ここから -->
   <exclusions>
       <exclusion>
           <groupId>org.apache.logging.log4j</groupId>
           <artifactId>log4j-to-slf4j</artifactId>
       </exclusion>
   </exclusions>
   <!-- ここから まで追加-->
</dependency>
<!-- 省略 -->
<!-- ここから -->
<dependency>
   <groupId>org.springframework.geode</groupId>
   <artifactId>spring-gemfire-starter</artifactId>
   <version>1.0.0.M3</version>
</dependency>
<!-- ここから まで追加-->
```

`src/main/java/com/example/apidemo/entity`に`BookGemFire.java` を下記のように編集します。

```java
@Region(name = "book")
public class BookGemFire {
    @Id
    private String id;
    private String title;
    private String author_name;
    private String price;

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public String getAuthor_name() {
        return author_name;
    }

    public void setAuthor_name(String author_name) {
        this.author_name = author_name;
    }

    public String getPrice() {
        return price;
    }

    public void setPrice(String price) {
        this.price = price;
    }
}
```

`src/main/java/com/example/apidemo`の`config`パッケージ内に`PccConfig.java`を追加し、下記のように編集します。
```java
@EnableGemfireCaching
@EnableEntityDefinedRegions(basePackages = "com.example.apidemo.entity")
@EnableGemfireRepositories(basePackages = "com.example.apidemo.repository.gem")
@Profile("cloud")
@Configuration
public class PccConfig {
}
```

`src/main/java/com/example/apidemo`に`repository/gem`パッケージを作成し、`BookGemfireRepository`を追加して下記のように編集します。
```java
@Repository
public interface BookGemfireRepository extends GemfireRepository<BookGemFire, String> {

    @Query("SELECT * FROM /book b WHERE b.id = $1")
    BookGemFire findBookById(String id);

}
```

`src/main/java/com/example/apidemo`に`service`パッケージを作成し、`BookService.java`を追加し、下記のように編集します。
```java
@Service
public class BookService {

    private final BookJpaRepository bookJpaRepository;

    public BookService(BookJpaRepository bookJpaRepository) {
        this.bookJpaRepository = bookJpaRepository;
    }

    private volatile boolean cacheMiss = false;

    public boolean isCacheMiss() {
        boolean isCacheMiss = this.cacheMiss;
        this.cacheMiss = false;
        return isCacheMiss;
    }

    protected void setCacheMiss() {
        this.cacheMiss = true;
    }

    @Cacheable(value = "book")
    public Book getBookById(String id) {

        setCacheMiss();

        Book book = this.bookJpaRepository.findBookById(id);

        return book;
    }

}
```

`ApiController.java`を下記のように編集します。
```java
@RestController
public class ApiController {

    private final BookJpaRepository bookJpaRepository;
    private final BookService bookService;
    private final ObjectMapper objectMapper;


    public ApiController(BookJpaRepository bookJpaRepository, BookService bookService, ObjectMapper objectMapper) {
        this.bookJpaRepository = bookJpaRepository;
        this.bookService = bookService;
        this.objectMapper = objectMapper;
    }

    private static Logger log = LoggerFactory.getLogger(ApiController.class);

    @RequestMapping("/")
    public Map helloWolrd() {
        log.info("Handling home");      
        Map<String, Object> body = new LinkedHashMap<>();
        body.put("message","Helloworld V1");
        body.put("index", System.getenv("CF_INSTANCE_INDEX"));
        body.put("host", System.getenv("CF_INSTANCE_IP"));
        body.put("java", System.getProperty("java.vm.version"));
        return body;
    }

    @RequestMapping(method = RequestMethod.GET, value = "/allbooks")
    public Object getAllBook() throws Exception {
        log.info("Handling allbooks");      
        return bookJpaRepository.findAll();
    }

    @RequestMapping(method = RequestMethod.GET, value = "/book")
    public String getBookById(@RequestParam(value = "id") String id) throws Exception {
        log.info("Handling book");      
        String ds="";
        Book book = bookService.getBookById(id);
        Boolean b = bookService.isCacheMiss();
        if (b) {
            ds = "MYSQL";
        } else if (!b) {
            ds = "PCC";
        }

        JSONObject jsonObject = new JSONObject(this.objectMapper.writeValueAsString(book));

        return jsonObject.put("ds",ds).toString();
    }

}
```
アプリをビルドしてpushします。
```shell
$ ./mvnw clean package -DskipTests=true
$ cf push --no-start
```

次に、PCCインスタンスをアプリケーションにbindしていきます。まず先ほどのインスタンスが作成完了しているかを確認します。
```console
$ cf service pcc                               

Service instance: pcc
Service: p-cloudcache
Bound apps: api-tkaburagi
Tags:
Plan: dev-plan
Description: Pivotal Cloud Cache offers the ability to deploy a GemFire cluster as a service in Pivotal Cloud Foundry.
Documentation url: http://docs.pivotal.io/p-cloud-cache/
Dashboard: http://cloudcache-7945e3f8-d8c5-4957-88c4-90307c2e9673.run.pcfone.io/pulse

Last Operation
Status: create succeeded
Message: Instance provisioning completed
Started: 2019-02-09T07:04:10Z
Updated: 2019-02-09T07:09:26Z
```

`Status`が`create succeeeded`になっていることを確認します。`provisioning`になっていたら完了するまで待ってください。完了したら`cf bind-service`でbindします。
```shell
$ cf bind-service api-tkaburagi pcc
$ cf env api-tkaburagi
```

`cf env`で出力されるユーザ名の`cluster_operator_*************`とパスワードの`*************`とLocatorsの`IP[PORT]`の部分をコピペしてメモ帳に残してください。

アプリケーションの`application.properties`を次のように編集します。
```properties
spring.data.gemfire.pool.DEFAULT.locators=${vcap.services.pcc.credentials.locators[0]}
spring.data.gemfire.security.username=${vcap.services.pcc.credentials.users[0].username}
spring.data.gemfire.security.password=${vcap.services.pcc.credentials.users[0].password}
```
//TODO
```shell
$ ./mvnw clean package -DskipTests=true
$ cf push --no-start
```

**ここまで完了したら進捗シートにチェックをしてください。**


## GemFireの準備とアプリケーションの起動
GemFireのcredenatilasから`url.gfsh`を取得し、先ほどと同じユーザ名とパスワードでログインします。

gfshはGemFireの構成変更などを行うCLIクライアントです。
```shell
gfsh
gfsh> connect --use-http=true --use-ssl --url=<GFSH_URL> --user=<CLUSTER_OPERATOR_USERNAME> --password=<CLUSTER_OPERATOR_PASSWD> --skip-ssl-validation
gfsh> create region --name=book --type=REPLICATE
gfsh> list regions
```

次にWebクライアントのPulseでGUI上で見てみます。Webブラウザを開いてGemFireのcredenatilasから取得できる`url.pulse`にアクセスしてください。ユーザ名とパスワードは同じです。ログインすると以下の画面が見えます。
![image](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/img/pulse-1.png)

`Data Browser`をクリックします。
![image](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/img/pulse-2.png)

`QUERY EDITOR`に以下のクエリーを入力し、出力結果を確認します。

`select * from /book` 

データがないためなにも出力されないことがわかります。次にアプリケーションをstartします。
```shell
$ cf start api-tkaburagi
```

```console
$ curl api-tkaburagi.apps.pcf.pcflab.jp/book?id=1 | jq 
{
  "id": "1",
  "title": "What's Pivotal",
  "author_name": "Rob Mee",
  "price": "1500",
  "ds": "MYSQL"
}
```

初回のアクセスのため、CacheMissが発生し、MySQLにデータを取りに行くことがわかります。次にPulseに戻り再度同じクエリーを実行してみましょう。
![image](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/img/pulse-3.png)

PCCにデータがキャッシュされていることがわかるはずです。再度アプリケーションにアクセスしてみます。
```console
$ curl api-tkaburagi.apps.pcf.pcflab.jp//book?id=1 | jq 
{
  "id": "1",
  "title": "What's Pivotal",
  "author_name": "Rob Mee",
  "price": "1500",
  "ds": "PCC"
}
```
キャッシュからデータが取得されていることがわかります。ローカルのメモリから取得していないことを確認するため、アプリケーションを再起動して、再度アクセスしてみます。
```shell
$ cf restart
```
キャッシュからデータが取得されていることを確認します。再起動してもデータは残っており、外部のメモリデータストアからデータを取得されていることがわかります。
```console
$ curl api-tkaburagi.apps.pcf.pcflab.jp/book?id=1 | jq 
{
  "id": "1",
  "title": "What's Pivotal",
  "author_name": "Rob Mee",
  "price": "1500",
  "ds": "PCC"
}
```

**ここまで完了したら進捗シートにチェックをしてください。**

ID: 1-5のデータが入っていますので、他のデータでも試してみましょう。GemFire内のデータを削除するには
```shell
gfsh> remove --region=book --key=<ID>
```
で削除できます。

PCCにはインメモリキャッシュの機能だけではなく`Continuous Query`やWANレプリケーションなど様々な機能が用意されています。
