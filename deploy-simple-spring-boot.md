# Deploying Simple Spring Boot

## プロジェクトの作成
アプリケーションを`git clone`します。
任意のディレクトリで以下のコマンドを実行してください。

```console
$ mkdir pcf-workshop
$ cd pcf-workshop
$ git clone https://github.com/tkaburagi/hello-cf
$ cd hello-cf
```

`pom.xml`を次のように編集します。
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
    <artifactId>demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>demo</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>11</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>org.apache.logging.log4j</groupId>
                    <artifactId>log4j-to-slf4j</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
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

## アプリケーションの編集
`src/main/java/com/example/demo`に`Controller`パッケージを作りその下に新しいファイル`ApiController.java`を追加し下記のように編集します。
import文がわからない場合は完成版を参照してください。

```java
@RestController
public class Controller {

    ObjectMapper mapper = new ObjectMapper();

    @RequestMapping("/")
    public String helloWolrd() {
        JSONObject jsonObject = new JSONObject();
        jsonObject.put("message","Helloworld V1");
        jsonObject.put("index", System.getenv("CF_INSTANCE_INDEX"));
        jsonObject.put("host", System.getenv("CF_INSTANCE_IP"));
        return jsonObject.toString();
    }

}
```

作ったアプリをビルドし、ローカルで稼働させてみます。
```shell
./mvnw package -Dmaven.test.skip=true
java -jar target/demo-0.0.1-SNAPSHOT.jar
```

```shell
curl localhost:8080/ | jq
```

以下のようなレスポンスが返って来れば成功です。
```json
{
  "message": "Helloworld V1",
  "index": "0",
  "host": "192.168.16.42"
}
```

## PCFにアプリケーションをpushする
次に`cf push`を使ってアプリケーションをPCF上にデプロイします。`cf target`でログインが出来ていることを確認してください。
```shell
cf push api-<STUDENT_ID> -p target/demo-0.0.1-SNAPSHOT.jar
```

`tkaburagi`の部分は自身のSTUDENT_IDに置き換えてください。同じレスポンスがサーバから返ってくるでしょう。
```shell
curl https://api-tkaburagi.apps.pcfone.io | jq
```

以上で最初アプリケーションのデプロイは終了です。

**ここまで完了したら進捗シートにチェックをしてください。**


## Exploring cf cli and Apps Manager
ここではいくつかの基本的なcf cliを使ってアプリケーションを操作してみます。

### 基本的なcf cli
アプリケーションをスケールアウトしたいときは、`cf scale`コマンドを利用します。
```shell
cf scale -i 2
curl https://api-tkaburagi.apps.pcf.pcflab.jp | jq
```
`curl`コマンドを何度か叩いてください。アプリケーションがスケールアウトし、`index`に対して負荷分散していることがわかります。また各インスタンスがホストされている`host`の値もindexごとに変化していることも確認してみましょう。

次に、`cf logs` を使ってアプリのログを取得します。
```shell
cf logs api-tkaburagi --recent
```
`--recent`オプションをつけるとLoggregatorにバッファされている、最近のログをダンプできます。
```shell
cf logs api-tkaburagi
```
オプションなしだと、ログがストリーミングされます。この状態で別ターミナルで`curl`コマンドを実行しアプリのエンドポイントにアクセスしてみてください。

logがストリーミングされていることがわかります。PCFではアプリログだけではなく、サーバログやアクセスログなどを収集します。

その他にも`cf apps`や`cf app api-tkaburagi`などを実行すると自分のテナントのアプリ一覧や1アプリの詳細な情報が見れます。
```shell
cf apps
cf app api-tkaburagi
```

`cf -h`を実行するとヘルプを表示できますので、時間のある方は色々なコマンドを試してみてください。

### マニフェストを使ったデプロイ
以下を実行し、一度アプリを削除します、
```shell
cf delete api-tkaburagi
cf apps
```
`cf apps`を実行するとアプリが削除されていることがわかるでしょう。

次に、プロジェクトディレクトリ直下に`manifest.yml`を追加して以下のように編集します。
`- name:`をapi-<STUDENT_ID>になるよう書き換えてください。
```yaml
applications:
- name: api-tkaburagi
  buildpack: https://github.com/cloudfoundry/java-buildpack.git#v4.16
  path: target/demo-0.0.1-SNAPSHOT.jar
  memory: 1g
  env:
    JBP_CONFIG_OPEN_JDK_JRE: '{ jre: { version: 11.0.+}}'
```

先ほどは`cf push`のオプションを使ってアプリをデプロイしましたが、manifestを使ってデプロイします。ここではアプリケーション名、ビルドパック、アプリのパスや利用するJREのバージョンなどを指定しています。

```shell
cf push
```
このマニフェストをカレントディレクトリに置き`cf push`するとディレクトリ直下のmanifestを読み込み、アプリがデプロイされます。また、`cf push -f path/tomanifest.yml`のような形でマニフェストを指定することもできます。

**ここまで完了したら進捗シートにチェックをしてください。**

### PCF Apps Managerの利用
Webブラウザで`https://apps.sys.pcf.pcflab.jp`にアクセスし、cf cliと同様にログインしてください。この手順は講師と一緒にブラウザを操作します。待っている方は自由に使って見てください。
