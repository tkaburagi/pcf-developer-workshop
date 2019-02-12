# Circuit Breakerを導入する
マイクロサービスアーキテクチャのアプリケーションでは障害の伝搬を防ぐため、Circuit Breakerパターンの導入が必要です。ここではSpring Cloudの`netflix-hystrix`を使って導入してみます。

## 障害が伝搬するアプリを作る
まずは障害が伝搬してしまうようなアプリに少し変更を加えます。

`pom.xml`に以下の依存ライブラリ情報を追加してください。
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
    <version>2.1.0.RELEASE</version>
</dependency>
```

次に`application.properties`を下記のように編集します。
```properties
api.url.dummy=http://api-tkaburagi.apps.internal:8080/dummy
```

`UiService.java`のクラス内に以下の変数とメソッドを追加します。
```java
 @Value( "${api.url.dummy}" )
 private String dummyUrl;
 
 public Model dummy(Model model) {
        String message = restTemplate.getForObject(dummyUrl, String.class);
        model.addAttribute("message", message);
        return model;
    }
```

`com.example.demo`の直下に`UiController.java`を下記のように編集します。
```java
@Component
@Controller
@Service
public class UiController {

    @Autowired
    UiService uiService;


    @RequestMapping(method = RequestMethod.GET, value = "/")
    public String home(String id, Model model) throws Exception {

        uiService.getAllBooks(model);
        uiService.getBookById(id, model);
        uiService.dummy(model);
        return "ui/index";
    }
}
```

`dummyUrl`はプロパティの`http://api-tkaburagi.apps.pcf.pcflab.jp/dummy`へのリクエストでAPI側のアプリにはこのようなエンドポイントはないため、`404`が返るはずです。

最後に`index.html`を以下のように編集します。
```html
//以上省略
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

    <p th:text="${message}"> </p> <!-- この行を追加 -->

</body>
</html>
```

一度この状態でアプリをデプロイします。

```shell
./mvnw package -Dmaven.test.skip=true && cf push
```

Webブラウザで`http://ui-tkaburagi.apps.pcf.pcflab.jp/?id=1`にアクセスしてみましょう。エラーが返ってくるはずです。APIアプリからエラーが返り、それがUIのアプリに伝搬し、そのか全ての機能に影響を与えていることがわかると思います。
//TODO

**ここまで完了したら進捗シートにチェックをしてください。**

## Circuit Breakerの導入
`UiService.java`の先ほどの`dummyメソッド`を以下のように変更します。
```java
 @HystrixCommand(fallbackMethod = "executeFallback")
 public Model dummy(Model model) {
     String message = restTemplate.getForObject(dummyUrl, String.class);
     model.addAttribute("message", message);
     return model;
 }

 public Model executeFallback(Model model, Throwable e) {
     return model.addAttribute("message", "No available");
  }
```

`@HystrixCommand`を`dummyメソッド`に付与し、`fallbackMethod`(このメソッドで例外が発生したときに呼ばれるメソッド)を`executeFallback`として定義しています。最後に`DemoUiApplication.java`を以下のように変更します。
```java
@EnableCircuitBreaker
@SpringBootApplication
public class DemoUiApplication{

    public static void main(String[] args) {
        SpringApplication.run(DemoUiApplication.class, args);
    }
}
```


`@EnableCircuitBreaker`を付与することでCiucuit Breakerのコンフィグレーションが完了します。再度アプリをデプロイします。
```shell
./mvnw package -Dmaven.test.skip=true && cf push
```

Webブラウザで`http://ui-tkaburagi.apps.pcf.pcflab.jp/?id=1`にアクセスしてみましょう。

例外が発生しているメソッドの処理は`fallbackMethod`が実行され、`No available`というメッセージが表示され、そのほかの機能は正常に動作していることがわかります。
//TODO

Circuit Breakerにはこの他にも様々な設定を行うことがきます。

[https://github.com/Netflix/Hystrix/wiki/Configuration](https://github.com/Netflix/Hystrix/wiki/Configuration
)

**ここまで完了したら進捗シートにチェックをしてください。**