# UIアプリケーションをデプロイする
ここではAPIを提供するアプリケーションを利用するUI側のアプリを作りデプロイします。
まずその前にAPI側のアプリのマニフェストを修正し、pushします。下記のように修正してください。
```yaml
applications:
- name: api-tkaburagi
  buildpack: java_buildpack_offline
  path: target/apidemo-0.0.1-SNAPSHOT.jar
  memory: 1g 
  routes:
    - route: api-tkaburagi.apps.internal
    - route: api-tkaburagi.apps.pcf.pcflab.jp
  env:
    JBP_CONFIG_OPEN_JDK_JRE: '{ jre: { version: 11.0.+}}'
```
PCFでは`apps.internal`というコンテナ間で通信するための内部向けのドメインが付与されています。このドメインは外部からのリクエストは受け付けません。上のマニフェストでは`api-tkaburagi.apps.internal`というルート情報をマップしています。この内部ドメインは[BOSH DNS](https://bosh.io/docs/dns/)により[Service Discovery](https://www.cloudfoundry.org/blog/polyglot-service-discovery-container-networking-cloud-foundry/)されています。外からはアクセスできません。`cf push`します。

```console
$ cf push
# curl api-tkaburagi.apps.internal
curl: (6) Could not resolve host: api-tkaburagi.apps.internal
```

リクエストすると、エラーが返るはずです。これからデプロイするUIのアプリケーションからはこの内部ドメインを使ってAPIに対してリクエストします。

## UIアプリの開発
[Spring Initializr](https://start.spring.io/)にアクセスして以下のようにSpring Bootプロジェクトを作成します。
* Group: そのまま
* Artifact: uidemo
* Dependencies: web, Actuator

![image](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/img/boot-2.png)

IDEへのインポートの方法は[簡単なアプリをデプロイする](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/deploy-simple-spring-boot.md)の章を参照して下さい。


manifest.ymlを追加します。`name`の`tkaburagi`の部分は自分のIDに合わせて書き換えてください。
```yaml
applications:
- name: ui-tkaburagi
  buildpack: java_buildpack_offline
  path: target/uidemo-0.0.1-SNAPSHOT.jar
  memory: 1g
  env:
    JBP_CONFIG_OPEN_JDK_JRE: '{  jre: { version: 11.0.+}}'
```

`appications.properties`を下記のように編集します。
```properties
management.endpoints.web.exposure.include=*
management.cloudfoundry.skip-ssl-validation=true
```

`pom.xml`を編集します。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.2.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.example</groupId>
    <artifactId>uidemo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>uidemo</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>11</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-rest</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-logging</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-log4j2</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <repositories>
        <repository>
            <id>spring-milestones</id>
            <name>Spring Milestones</name>
            <url>https://repo.spring.io/libs-milestone</url>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
        <repository>
            <id>spring-releases</id>
            <name>Spring Releases</name>
            <url>http://repo.spring.io/release</url>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
    </repositories>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```
今回はSpring Bootで多く使われる[Thymeleaf](https://www.thymeleaf.org/doc/tutorials/3.0/usingthymeleaf.html)というテンプレートエンジンを利用します。

次に`application.properties`を下記のように編集します。
```properties
api.url=http://api-tkaburagi.apps.internal:8080
```

`com.example.uidemo`の直下に`Book.java`を追加し、下記のように編集します。
```java
public class Book implements Serializable {
    private String id;
    private String title;
    private String author_name;
    private String price;
    private String ds;

    public String getDs() {
        return ds;
    }

    public void setDs(String ds) {
        this.ds = ds;
    }

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

`com.example.uidemo`の直下に`AppInfo.java`を追加し、下記のように編集します。
```java
public class AppInfo implements Serializable {
    private String message;
    private String index;
    private String host;
    private String java;

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }

    public String getIndex() {
        return index;
    }

    public void setIndex(String index) {
        this.index = index;
    }

    public String getHost() {
        return host;
    }

    public void setHost(String host) {
        this.host = host;
    }

    public String getJava() {
        return java;
    }

    public void setJava(String java) {
        this.java = java;
    }
}

```

`com.example.uidemo`の直下に`UiService.java`を追加し、下記のように編集します。
```java
@Service
public class UiService {

    private final RestTemplate restTemplate;
    private final ObjectMapper objectMapper;

    public UiService(RestTemplateBuilder builder, ObjectMapper objectMapper) {
        this.restTemplate = builder.build();
        this.objectMapper = objectMapper;
    }

    @Value( "${api.url}" )
    private String apiUrl;

    public AppInfo getAppInfo() throws Exception {
        String result = this.restTemplate.getForObject(apiUrl, String.class);
        AppInfo a = this.objectMapper.readValue(result, AppInfo.class);
        return  a;
    }

    public Book[] getAllBooks() throws Exception {
        String result = this.restTemplate.getForObject(apiUrl + "/allbooks", String.class);
        Book[] bList = this.objectMapper.readValue(result, Book[].class);
        return  bList;
    }

    public Book getBookById(@RequestParam("id") String id) throws Exception {
        String targetUrl = UriComponentsBuilder.fromUriString(apiUrl + "/book").queryParam("id", id).build().toString();
        String result = this.restTemplate.getForObject(targetUrl, String.class);
        Book b = this.objectMapper.readValue(result, Book.class);
        return b;
    }
}

```

`com.example.uidemo`の直下に`UiController.java`を追加し、下記のように編集します。
```java
@Controller
public class UiController {

    private final UiService uiService;
    private static Logger log = LoggerFactory.getLogger(UiController.class);
    
    public UiController(UiService uiService) {
        this.uiService = uiService;
    }

    @RequestMapping(method = RequestMethod.GET, value = "/")
    public String home(String id, Model model) throws Exception {
        log.info("Handling home");
        model.addAttribute("appinfo", this.uiService.getAppInfo());
        model.addAttribute("allbooks", this.uiService.getAllBooks());
        model.addAttribute("searchedBook", this.uiService.getBookById(id));
        return "ui/index";
    }
}
```

最後に`src/main/resources/templates/`に`ui`フォルダを作り、その中に`index.html`を追加し、以下のように編集します。
```html
<!DOCTYPE html>
<html xmlns:th="http:///www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
    <link rel="stylesheet"
          href="http://yui.yahooapis.com/pure/0.6.0/pure-min.css" />
</head>
<body>
<table class="pure-table pure-table-striped table logo">
    <thead>
    <tr>
        <th>Message</th>
        <th>Index</th>
        <th>Host</th>
        <th>Java Version</th>
    </tr>
    </thead>
    <th:block th:each="appinfo : ${appinfo}">
        <tr>
            <td width="30%" th:text="${appinfo.message}" />
            <td width="30%" th:text="${appinfo.index}" />
            <td width="30%" th:text="${appinfo.host}" />
            <td width="30%" th:text="${appinfo.java}" />
        </tr>
    </th:block>
</table>
<table class="pure-table pure-table-striped table logo">
    <thead>
    <tr>
        <th>Title</th>
        <th>Author</th>
        <th>Price</th>
    </tr>
    </thead>
    <th:block th:each="allbooks : ${allbooks}">
        <tr>
            <td width="30%" th:text="${allbooks.title}" />
            <td width="30%" th:text="${allbooks.author_name}" />
            <td width="30%" th:text="${allbooks.price}" />
        </tr>
    </th:block>
</table>

<form class="pure-form" th:action="@{/}">
    <input type="text" class="pure-input-rounded" name="id" />
    <button type="submit">Search</button>
</form>
<table class="pure-table pure-table-striped table logo">
    <thead>
    <tr>
        <th>Title</th>
        <th>Author</th>
        <th>Price</th>
        <th>Data Source</th>
    </tr>
    </thead>
    <th:block th:each="searchedBook : ${searchedBook}">
        <tr>
            <td width="30%" th:text="${searchedBook.title}" />
            <td width="30%" th:text="${searchedBook.author_name}" />
            <td width="30%" th:text="${searchedBook.price}" />
            <td width="30%" th:text="${searchedBook.ds}" />
        </tr>
    </th:block>
</table>
</body>
</html>
```

アプリケーションをビルドし、PCFにpushしてみましょう。
```shell
$ ./mvnw package -Dmaven.test.skip=true
$ cf push
```

Webブラウザで`http://ui-tkaburagi.apps.pcf.pcflab.jp/?id=1`にアクセスしてみましょう。エラーが返ってくるはずです。なぜでしょうか？

APIがリスンする`apps.internal`のドメインのエンドポイントはデフォルトでは全てのポートへのアクセスが遮断されています。`cf add-network-policy`でネットワークポリシーを定義し、アクセスルートを開きます。

使い方は` cf add-network-policy SOURCE_APP --destination-app DESTINATION_APP [(--protocol (tcp | udp) --port RANGE)]`です。

今回は`SOURCE_APP`は`ui-tkaburagi`、`DESTINATION_APP`は`api-tkaburagi`、`--protocol`はtcp、`--port`は`8080`です。

```shell
$ cf add-network-policy ui-tkaburagi --destination-app api-tkaburagi  --protocol tcp --port 8080
```

再度`http://ui-tkaburagi.apps.pcf.pcflab.jp/?id=1`にアクセスしてみましょう。
以下のような画面表示されるはずです。
![image](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/img/ui-1.png)

これでUI - API間の疎通ができました。

**ここまで完了したら進捗シートにチェックをしてください。**

