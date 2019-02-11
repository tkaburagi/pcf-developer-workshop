# Using Pivotal Cloud Cache
ここではMarkeplace上のPivotal Cloud Cacheを利用し、Spring BootのアプリケーションからPivotal Cloud Cacheを扱います。

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
`Book.java` を下記のように編集します。

```java
import org.springframework.data.annotation.Id;
import org.springframework.data.gemfire.mapping.annotation.Region;

import javax.persistence.Entity;
import javax.persistence.Table;

@Entity
@Table(name = "book")
@Region(name = "book")
public class Book {
//省略
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

**ここまで完了したら進捗シートにチェックをしてください。**

## CredHubの利用
まず先ほどのインスタンスが作成完了しているかを確認します。
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

`Status`が`create succeeeded`になっていることを確認します。`provisioning`になっていたら完了するまで待ってください。
先ほどは`bind-service`で直接アプリケーションの環境変数にセットしましたが、設定内容がターミナル上で表示されてしまいます。そこでCredHub Service Brokerを利用し、セキュアに設定情報をセットします。まずは`cf create-service-key`でサービスキーを作成します。

```console
$ cf create-service-key pcc pcc-svc-key
$ cf service-key pcc pcc-svc-key
Getting key pcc-svc-key for service instance pcc as tkaburagi@pivotal.io...

{
 "distributed_system_id": "0",
 "locators": [
  "192.168.12.34[55221]"
 ],
 "urls": {
  "gfsh": "https://cloudcache-7945e3f8-d8c5-4957-88c4-90307c2e9673.run.pcfone.io/gemfire/v1",
  "pulse": "https://cloudcache-7945e3f8-d8c5-4957-88c4-90307c2e9673.run.pcfone.io/pulse"
 },
 "users": [
  {
   "password": "jsZX6TJjhkwewK848hhLA",
   "roles": [
    "cluster_operator"
   ],
   "username": "cluster_operator_L9szKr9Ss777W4eak158w"
  },
  {
   "password": "xbcZbexkOuI9mZrGwH3VSg",
   "roles": [
    "developer"
   ],
   "username": "developer_8sbVAyVvBbiUWtNYXkLcg"
  }
 ],
 "wan": {
  "sender_credentials": {
   "active": {
    "password": "FC8djvalsc73ZWRX4hQig",
    "username": "gateway_sender_01T6b35pyJZBEXTEFsoh7g"
   }
  }
 }
}
```

ユーザ名の`cluster_operator_L9szKr9Ss777W4eak158w`とパスワードの`jsZX6TJjhkwewK848hhLA`の部分をコピペしてメモ帳に残してください。Credhubのサービスインスタンスを作成します。
```shell
cf create-service credhub default pcc-cred-api  -c '{"pccusername":"cluster_operator_L9szKr9Ss777W4eak158w", "pccpassword":"jsZX6TJjhkwewK848hhLA"}'
```

```console
$ cf service pcc-cred-api
Service instance: pcc-cred-api
Service: credhub
Bound apps:
Tags:
Plan: default
Description: Stores configuration parameters securely in CredHub
Documentation url:
Dashboard:

Last Operation
Status: create succeeded
Message:
Started: 2019-02-11T04:41:14Z
Updated: 2019-02-11T04:41:14Z
```

PCCの認証情報を持ったCredHubインスタンスが作成され、これをアプリケーションにbindする。
```shell
cf bind-service api-tkaburagi pcc-cred-api
```

```console
$ cf env api-tkaburagi
Getting env variables for app api-tkaburagi in org pivot-tkaburagi / space playground as tkaburagi@pivotal.io...
OK

System-Provided:
{
 "VCAP_SERVICES": {
  "credhub": [
   {
    "binding_name": null,
    "credentials": {
     "credhub-ref": "/credhub-service-broker/credhub/f95602af-33fd-422a-815e-fec3dc975ec8/credentials"
    },
    "instance_name": "pcc-cred-api",
    "label": "credhub",
    "name": "pcc-cred-api",
    "plan": "default",
    "provider": null,
    "syslog_drain_url": null,
    "tags": [
     "credhub"
    ],
    "volume_mounts": []
   }
  ],
  //以下省略
```

CredHubへの参照のみが表示され、認証情報は表示されていないことがわかります。

アプリケーションの`application.properties`を次のように編集します。
```properties
spring.data.gemfire.security.username=${vcap.services.pcc-cred-api.credentials.pccusername}
spring.data.gemfire.security.password=${vcap.services.pcc-cred-api.credentials.pccpassword}
```

アプリケーションからは直接CredHubを参照する必要はなく、CredHubの参照キーを環境変数から取得し、設定ファイルから取得します。

**ここまで完了したら進捗シートにチェックをしてください。**


## GemFireの準備とアプリケーションの起動
GemFireのサービスキーから`url.gfsh`を取得し、先ほどと同じユーザ名とパスワードでログインします。
```console
$ cf service-key pcc pcc-svc-key
Getting key pcc-svc-key for service instance pcc as tkaburagi@pivotal.io...

{
 "distributed_system_id": "0",
 "locators": [
  "192.168.12.34[55221]"
 ],
 "urls": {
  "gfsh": "https://cloudcache-7945e3f8-d8c5-4957-88c4-90307c2e9673.run.pcfone.io/gemfire/v1",
  "pulse": "https://cloudcache-7945e3f8-d8c5-4957-88c4-90307c2e9673.run.pcfone.io/pulse"
 },
 "users": [
  {
   "password": "jsZX6TJjhkwewK848hhLA",
   "roles": [
    "cluster_operator"
   ],
   "username": "cluster_operator_L9szKr9Ss777W4eak158w"
  },
  {
   "password": "xbcZbexkOuI9mZrGwH3VSg",
   "roles": [
    "developer"
   ],
   "username": "developer_8sbVAyVvBbiUWtNYXkLcg"
  }
 ],
 "wan": {
  "sender_credentials": {
   "active": {
    "password": "FC8djvalsc73ZWRX4hQig",
    "username": "gateway_sender_01T6b35pyJZBEXTEFsoh7g"
   }
  }
 }
}
```

gfshはGemFireの構成変更などを行うCLIクライアントです。
```shell
gfsh
gfsh> connect --use-http=true --use-ssl --url=<GFSH_URL> --user=<CLUSTER_OPERATOR_USERNAME> --password=<CLUSTER_OPERATOR_PASSWD> --skip-ssl-validation
gfsh> create region --name=book --type=REPLICATE
gfsh> list regions
```

次にWebクライアントのPulseでGUI上で見てみます。Webブラウザを開いてサービスキーから取得できる`url.pulse`にアクセスしてください。ユーザ名とパスワードは同じです。




```shell
./mvnw clean package -DskipTests=true && cf push
```


