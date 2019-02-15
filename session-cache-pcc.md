# セッションを外部ストアに保存する
セッション情報を外部のデータストアに保存することは信頼性が高くスケーラブルなアプリケーションを開発する上でとても重要です。ここでは先に利用したPivotal Cloud CacheとBuildpacksの機能を使って簡単にSession State Cachingを実現します。

## PCCインスタンスのアップデート
```shell
cf update-service pcc -t session-replication
cf bind-service ui-tkaburagi pcc
```
RedisもしくはPCCのインスタンスで`tag`に`session-replication`を持つインスタンスもしくはインスタンス名に`session-replication`が入っているインスタンスをbindするとHTTP SessionがRedisもしくはPCCに格納されます。上記のコマンドは前に作ったPCCのインスタんしに`session-replication`のタグを付与しています。

詳細は[ドキュメント](https://github.com/cloudfoundry/java-buildpack/blob/a0cf9ece33ddcc763079668bd40fbd99b66fbd7a/docs/container-tomcat.md)で確認してください。


```console
$ cf service pcc

Service instance: pcc
Service: p-cloudcache
Bound apps: api-tkaburagi
Tags: session-replication
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

セッションを格納するためのGemFireのRegionを新しく作ります。Region名はアプリ側から指定ができますが、デフォルトだと`ClusteredSpringSessions`というRegionになるため今回はそれを利用します。
```shell
gfsh
gfsh> connect --use-http=true --use-ssl --url=<GFSH_URL> --user=<CLUSTER_OPERATOR_USERNAME> --password=<CLUSTER_OPERATOR_PASSWD> --skip-ssl-validation
create region --name=ClusteredSpringSessions --type=PARTITION_HEAP_LRU
```

`connect`のユーザ名などは[こちら](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/backend-pcc.md)を参考にしてください。

これでPCC側の準備は完了です。

## アプリの修正
`pom.xml`に以下の依存ライブラリ情報を追加してください。
```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-gemfire</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-gemfire</artifactId>
</dependency>
 ```

 `PccConfig.java`を追加し、以下のように編集します。
```java
import org.springframework.data.gemfire.config.annotation.ClientCacheApplication;
import org.springframework.session.data.gemfire.config.annotation.web.http.EnableGemFireHttpSession;

@EnableGemFireHttpSession(poolName = "DEFAULT")
@ClientCacheApplication
@EnableSecurity
public class PccConfig {

}
 ```

`UiController.java`を以下のように編集します。
```java     
@RequestMapping(method = RequestMethod.GET, value = "/")
public String home(String id, Model model, HttpSession session) throws Exception {
        log.info("Handling home");
        model.addAttribute("appinfo", uiService.getAppInfo());
        model.addAttribute("allbooks", uiService.getAllBooks());
        model.addAttribute("searchedBook", uiService.getBookById(id));
        model.addAttribute("message", uiService.dummy());
        session.setAttribute("iam", "I'm a Session");
        return "ui/index";
}
```

最後に`index.html`を以下のように編集します。
```html
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

    <p th:text="${message}"> </p>
    <p th:text="${session.iam}"> </p>   <!-- この行を追加 -->

</body>
</html>
```

[PCCを利用する](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/backend-pcc.md)の手順同様、PCCの認証情報はCredHubから取得します。
```shell
cf bind-service ui-tkaburagi pcc-cred-api
```

アプリケーションの`application.properties`を次のように編集します。
```properties
spring.data.gemfire.pool.DEFAULT.locators=${vcap.services.pcc.credentials.locators[0]}
spring.data.gemfire.security.username=${vcap.services.pcc.credentials.users[0].username}
spring.data.gemfire.security.password=${vcap.services.pcc.credentials.users[0].password}
```

アプリケーションをデプロイしましょう。
```shell
./mvnw clean package -DskipTests=true && cf push
```

**ここまで完了したら進捗シートにチェックをしてください。**


## PCCからセッション情報を取得する。
Webブラウザで`http://ui-tkaburagi.apps.pcf.pcflab.jp/?id=1`にアクセスしてみましょう。

セッション情報が表示されているでしょう。次にPulseからGemFireのインスタンスの中身を確認してみます。WebブラウザでPulseにアクセスし、`Data Browser`を開いてください。`QUERY EDITOR`に以下のクエリーを入力し、出力結果を確認します。

`select * from /ClusteredSpringSessions` 
![image](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/img/session-1.png)


データが格納されていることを確認します。`last access time`の値をメモし、再度ブラウザからアプリにアクセスしてみましょう。`last access time`の値が更新され、GemFireからセッションデータが取得されていることがわかります。
![image](https://github.com/tkaburagi/pcf-developer-workshop/blob/master/img/session-2.png)

**ここまで完了したら進捗シートにチェックをしてください。**