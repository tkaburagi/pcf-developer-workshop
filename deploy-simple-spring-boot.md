# Deploying Simple Spring Boot

## プロジェクトの作成

## アプリケーションの編集
`src/main/java/com/example/demo`に新しいファイル`Controller.java`を追加し下記のように編集します。

```java
package com.example.demo;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class Controller {

    @RequestMapping("/hw")
    public String helloWolrd() {
        return "Hello world";
    }

}
```

```shell
./mvnw package -Dmaven.test.skip=true
```

```shell
java -jar target/demo-0.0.1-SNAPSHOT.jar
```

```shell
curl localhost:8080/hw
```

## PCFにアプリケーションをpushする
```shell
cf push api-<STUDENT_ID> -p target/demo-0.0.1-SNAPSHOT.jar
```

`manifest.yml`を追加して以下のように編集します。
```yaml
applications:
- name: api-tkaburagi
  buildpack: https://github.com/cloudfoundry/java-buildpack.git#v4.16
  path: target/demo-0.0.1-SNAPSHOT.jar
  memory: 1g
  env:
    JBP_CONFIG_OPEN_JDK_JRE: '{ jre: { version: 11.0.+}}'
```

## Exploring cf cli and Apps Manager
