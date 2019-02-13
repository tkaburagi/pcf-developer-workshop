# Using Pivotal Cloud Cache
ここではMarkeplace上のPivotal Cloud Cacheを利用し、Spring BootのアプリケーションからPivotal Cloud Cacheを扱います。GemFireなどのインメモリデータグリッドにアクセスするには`Look Asideパターン`と`Inline Cache`パターンの二つがありますが、今回は`Look Aside`パターンで実装します。

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
cf create-service p-cloudcache dev-plan pcc
```

作成にはしばらく時間がかかりますので、待っている間次に進んでください。


## アプリケーションの修正
`src/main/java/com/example/demo/entity`に`BookGemfire.java` を下記のように編集します。

```java
import org.springframework.data.annotation.Id;
import org.springframework.data.gemfire.mapping.annotation.Region;

import javax.persistence.Entity;
import javax.persistence.Table;

@Entity
@Region(name = "book")
public class BookGemfire {
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

`src/main/java/com/example/demo`の`Config`パッケージ内に`Config.java`を追加し、下記のように編集します。
```java
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;
import org.springframework.data.gemfire.cache.config.EnableGemfireCaching;
import org.springframework.data.gemfire.config.annotation.EnableEntityDefinedRegions;
import org.springframework.data.gemfire.repository.config.EnableGemfireRepositories;

@EnableGemfireCaching
@EnableEntityDefinedRegions(basePackages = "com.example.demo.entity")
@EnableGemfireRepositories(basePackages = "com.example.demo.repo.gem")
@Profile("cloud")
@Configuration
public class PccConfig {
}
```

`src/main/java/com/example/demo`に`repository/gem`パッケージを作成し、`BookGemfireRepository`を追加して下記のように編集します。
```java
import com.example.demo.entity.Book;
import org.springframework.data.gemfire.repository.GemfireRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface BookGemFireRepository extends GemfireRepository<Book, String> {

    @Query("SELECT * FROM /book b WHERE b.id = $1")
    Book findBookById(String id);

}
```

`src/main/java/com/example/demo`に`service`パッケージを作成し、`BookService.java`を追加し、下記のように編集します。
```java
import com.example.demo.entity.Book;
import com.example.demo.repo.jpa.BookJpaRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

@Service
public class BookService {
    @Autowired
    BookJpaRepository bookJpaRepository;

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

        Book book = bookJpaRepository.findBookById(id);

        return book;
    }

}
```

`ApiController.java`を下記のように編集します。
```java
import com.example.demo.repo.jpa.BookJpaRepository;
import com.example.demo.service.BookService;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class Controller {

    @Autowired
    BookJpaRepository bookJpaRepository;

    @Autowired
    BookService bookService;

    ObjectMapper mapper = new ObjectMapper();

    @RequestMapping("/")
    public String helloWolrd() {
        JSONObject jsonObject = new JSONObject();
        jsonObject.put("message","Helloworld V1");
        jsonObject.put("index", System.getenv("CF_INSTANCE_INDEX"));
        jsonObject.put("host", System.getenv("CF_INSTANCE_IP"));
        return jsonObject.toString();
    }

    @RequestMapping(method = RequestMethod.GET, value = "/allbooks")
    public String getAllBook() throws Exception {

        return mapper.writeValueAsString(bookJpaRepository.findAll());

    }

    @RequestMapping(method = RequestMethod.GET, value = "/book")
    public String getBookById(@RequestParam(value = "id") String id) throws Exception {

        String ds="";
        Book book = bookService.getBookById(id);
        Boolean b = bookService.isCacheMiss();
        if (b) {
            ds = "MYSQL";
        } else if (!b) {
            ds = "PCC";
        }

        JSONObject jsonObject = new JSONObject(mapper.writeValueAsString(book));
        jsonObject.put("ds",ds);

        return jsonObject.toString();
    }

}
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
cf bind-service api-tkaburagi pcc
cf env api-tkaburagi
```

`cf env`で出力されるユーザ名の`cluster_operator_*************`とパスワードの`*************`とLocatorsの`IP[PORT]`の部分をコピペしてメモ帳に残してください。

**ここまで完了したら進捗シートにチェックをしてください。**


アプリケーションの`application.properties`を次のように編集します。
```properties
spring.data.gemfire.pool.DEFAULT.locators=10.0.8.4[55221]
spring.data.gemfire.security.username=${vcap.services.pcc.users.password}
spring.data.gemfire.security.password=${vcap.services.pcc.users.username}
```

**ここまで完了したら進捗シートにチェックをしてください。**


## GemFireの準備とアプリケーションの起動
GemFireのサービスキーから`url.gfsh`を取得し、先ほどと同じユーザ名とパスワードでログインします。

gfshはGemFireの構成変更などを行うCLIクライアントです。
```shell
gfsh
gfsh> connect --use-http=true --use-ssl --url=<GFSH_URL> --user=<CLUSTER_OPERATOR_USERNAME> --password=<CLUSTER_OPERATOR_PASSWD> --skip-ssl-validation
gfsh> create region --name=book --type=REPLICATE
gfsh> list regions
```

次にWebクライアントのPulseでGUI上で見てみます。Webブラウザを開いてサービスキーから取得できる`url.pulse`にアクセスしてください。ユーザ名とパスワードは同じです。ログインすると以下の画面が見えます。
![image](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/img/pulse-1.png)

`Data Browser`をクリックします。
![image](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/img/pulse-2.png)

`QUERY EDITOR`に以下のクエリーを入力し、出力結果を確認します。

`select * from /book` 

データがないためなにも出力されないことがわかります。次にアプリケーションをpushします。
```shell
./mvnw clean package -DskipTests=true && cf push
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
cf restart
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