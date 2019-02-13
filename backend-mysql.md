# Using MySQL for PCF
ここではMarkeplace上のMySQL for Pivotal Cloud Foundryを利用し、Spring BootのアプリケーションからMySQLを扱います。

## MySQLインスタンスを作成

`cf marketplace`コマンドで使えるサービス一覧が確認できます。
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

`cf create-service`コマンドでサービスインスタンスを作成します。`cf create-service <Service Nama> <Plan Nanme> <Service Instance Name`でSevice Brokerを使ってサービスインスタンスをプロビジョニングできます。
```shell
cf create-service p.mysql db-small mysql
```
作成にはしばらく時間がかかりますので、待っている間次に進んでください。

## アプリケーションの修正
`api-tkaburagi`のアプリを修正します。まずは`pom.xml`に以下を追記します。`<dependency> </dependency`>のエントリに以下を追加します。
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-spring-service-connector</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-cloudfoundry-connector</artifactId>
</dependency>
```

`src/main/java/com/example/demo`に`Entity`パッケージを作成し、新しいファイル`Book.java`を追加し下記のように編集します。
```java
package com.example.demo;

import javax.persistence.Entity;
import javax.persistence.Table;

@Entity
@Table(name = "book")
public class Book {
    @javax.persistence.Id
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
`src/main/java/com/example/demo`に`repository/jpa`パッケージを作成し、新しいファイル`BookJpaRepository.java`を追加し下記のように編集します。
```java
import com.example.demo.entity.Book;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface BookJpaRepository extends JpaRepository<Book, String> {
    Book findBookById(final String id);
}
```

`src/main/java/com/example/demo`に`Config`パッケージを作成し、新しいファイル`DbCloudConfig.java`を追加し下記のように編集します。
```java
import javax.sql.DataSource;

import org.springframework.cloud.config.java.AbstractCloudConfig;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;
import org.springframework.core.io.ClassPathResource;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;
import org.springframework.jdbc.datasource.init.DatabasePopulatorUtils;
import org.springframework.jdbc.datasource.init.ResourceDatabasePopulator;

@Configuration
@EnableJpaRepositories(basePackages = "com.example.demo.repo.jpa")
@Profile("cloud")
public class DbCloudConfig extends AbstractCloudConfig {

    @Bean
    public DataSource dataSource() {
        DataSource dataSource = connectionFactory().dataSource();

        ResourceDatabasePopulator databasePopulator = new ResourceDatabasePopulator();
        databasePopulator.addScript(new ClassPathResource("demo.sql"));

        DatabasePopulatorUtils.execute(databasePopulator, dataSource);

        return dataSource;
    }

}
```

`ApiController.java`を下記のように編集します。
```java
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class Controller {

    @Autowired
    BookJpaRepository bookJpaRepository;

    ObjectMapper mapper = new ObjectMapper();

    @RequestMapping("/")
    public String helloWolrd() {
        JSONObject jsonObject = new JSONObject();
        jsonObject.put("message","Helloworld V2");
        jsonObject.put("index", System.getenv("CF_INSTANCE_INDEX"));
        jsonObject.put("host", System.getenv("CF_INSTANCE_IP"));
        return jsonObject.toString();
    }

    @RequestMapping(method = RequestMethod.GET, value = "/allbook")
    public Object getAllBook() throws Exception{
      return bookJpaRepository.findAll();
    }
}
```


`src/main/resources`に`demo.sql`を追加し下記のように編集します。
```sql
DROP TABLE IF EXISTS book;

CREATE TABLE IF NOT EXISTS book
(
id VARCHAR(11),
title VARCHAR(64),
author_name VARCHAR(32),
price VARCHAR(11)
);

INSERT INTO book
(id, title, author_name, price)
values ("1", "What's Pivotal", "Rob Mee", "1500");

INSERT INTO book
(id, title, author_name, price)
values ("2", "eXtream Programming", "Kent Beck", "1200");

INSERT INTO book
(id, title, author_name, price)
values ("3", "Site Reliability Engineering", "Google", "5600");

INSERT INTO book
(id, title, author_name, price)
values ("4", "Introduction of Concourse CI", "Pivotal CI Team", "4900");

INSERT INTO book
(id, title, author_name, price)
values ("5", "Pivotal Cloud Foundry Deep Dive", "Pivotal Japan", "8900");
```

**ここまで完了したら進捗シートにチェックをしてください。**


## アプリケーションのデプロイ
まず先ほどのインスタンスが作成完了しているかを確認します。
```console
$ cf service mysql                               

Service instance: mysql
Service: p.mysql
Bound apps: api-tkaburagi
Tags:
Plan: db-small
Description: Dedicated instances of MySQL
Documentation url: https://docs.pivotal.io/p-mysql/2-5/use.html
Dashboard:

Last Operation
Status: create succeeded
Message: Instance provisioning completed
Started: 2019-02-04T06:39:05Z
Updated: 2019-02-04T06:43:09Z
```

`Status`が`create succeeeded`になっていることを確認します。`provisioning`になっていたら完了するまで待ってください。

アプリケーションをビルドして、アプリケーションをデプロイします。`--no-start`オプションをつけて、アプリケーションを起動せずにpushします。
```shell
./mvnw clean package -DskipTests=true && cf push --no-start
```

次に、`cf bind-service`でMySQLインスタンスの接続情報をアプリケーションの環境変数にバインドします。
```shell
cf bind-service api-tkaburagi mysql
```

`cf env`でセットされている環境変数を確認します。
```console
$ cf env api-tkaburagi
Getting env variables for app api-tkaburagi in org pivot-tkaburagi / space playground as tkaburagi@pivotal.io...
OK

System-Provided:
{
 "VCAP_SERVICES": {
  "p.mysql": [
   {
    "binding_name": null,
    "credentials": {
     "hostname": "q-n4s3y1.q-g35863.bosh",
     "jdbcUrl": "jdbc:mysql://q-n4s3y1.q-g35863.bosh:3306/service_instance_db?user=bee3188c58be4e618e39b321c1ff9431\u0026password=plyfd8s81h923auf\u0026useSSL=false",
     "name": "service_instance_db",
     "password": "plyfd8s81h923auf",
     "port": 3306,
     "uri": "mysql://bee3188c58be4e618e39b321c1ff9431:plyfd8s81h923auf@q-n4s3y1.q-g35863.bosh:3306/service_instance_db?reconnect=true",
     "username": "bee3188c58be4e618e39b321c1ff9431"
    },
    "instance_name": "mysql",
    "label": "p.mysql",
    "name": "mysql",
    "plan": "db-small",
    "provider": null,
    "syslog_drain_url": null,
    "tags": [
     "mysql"
    ],
    "volume_mounts": []
   }
  ]
 }
}

{
 "VCAP_APPLICATION": {
  "application_id": "3c9a5cab-410f-4bba-bfe1-73b591fd0fdb",
  "application_name": "api-tkaburagi",
  "application_uris": [
   "api-tkaburagi.apps.pcfone.io",
  ],
  "application_version": "bd2edf8c-41fd-4161-b251-05202874f76d",
  "cf_api": "https://api.run.pcfone.io",
  "limits": {
   "disk": 1024,
   "fds": 16384,
   "mem": 1024
  },
  "name": "api-tkaburagi",
  "space_id": "974f9813-cf0d-498f-83a5-103b33ceb8be",
  "space_name": "playground",
  "uris": [
   "api-tkaburagi.apps.pcfone.io",
  ],
  "users": null,
  "version": "bd2edf8c-41fd-4161-b251-05202874f76d"
 }
}

User-Provided:
JBP_CONFIG_OPEN_JDK_JRE: { jre: { version: 11.0.+}}

No running env variables have been set

No staging env variables have been set
```

JDBCのURIパスワード、ユーザ名などがセットされていることがわかります。この環境変数はアプリケーションの起動時に読み込まれます。アプリケーションを`cf start`で起動します。
```shell
cf start api-tkaburagi
```

起動したらアプリケーションのエンドポイントにリクエストを送り、デーベースからの値を取得します。
以下のような結果が得られるでしょう。
```console
$ curl api-tkaburagi.apps.pcf.pcflab.jp/allbooks | jq                                               
[
  {
    "id": "1",
    "title": "What's Pivotal",
    "author_name": "Rob Mee",
    "price": "1500"
  },
  {
    "id": "2",
    "title": "eXtream Programming",
    "author_name": "Kent Beck",
    "price": "1200"
  },
  {
    "id": "3",
    "title": "Site Reliability Engineering",
    "author_name": "Google",
    "price": "5600"
  },
  {
    "id": "4",
    "title": "Introduction of Concourse CI",
    "author_name": "Pivotal CI Team",
    "price": "4900"
  },
  {
    "id": "5",
    "title": "Pivotal Cloud Foundry Deep Dive",
    "author_name": "Pivotal Japan",
    "price": "8900"
  }
]
```

**ここまで完了したら進捗シートにチェックをしてください。**