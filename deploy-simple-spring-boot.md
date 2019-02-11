# Deploying Simple Spring Boot

## �ץ������Ȥ�����
���ץꥱ�`������`git clone`���ޤ���
����Υǥ��쥯�ȥ�����¤Υ��ޥ�ɤ�g�Ф��Ƥ���������

```console
$ mkdir pcf-workshop
$ cd pcf-workshop
$ git clone https://github.com/tkaburagi/hello-cf
$ cd hello-cf
```

`pom.xml`��ΤΤ褦�˾������ޤ���
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

## ���ץꥱ�`�����ξ���
`src/main/java/com/example/demo`��`Controller`�ѥå��`�������ꤽ���¤��¤����ե�����`ApiController.java`��׷�Ӥ���ӛ�Τ褦�˾������ޤ���
import�Ĥ��狼��ʤ����Ϥ���ɰ����դ��Ƥ���������

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

���ä����ץ��ӥ�ɤ�����`����ǼڃP�����Ƥߤޤ���
```shell
./mvnw package -Dmaven.test.skip=true
java -jar target/demo-0.0.1-SNAPSHOT.jar
```

```shell
curl localhost:8080/ | jq
```

���¤Τ褦�ʥ쥹�ݥ󥹤����ä�����гɹ��Ǥ���
```json
{
  "message": "Helloworld V1",
  "index": "0",
  "host": "192.168.16.42"
}
```

## PCF�˥��ץꥱ�`������push����
�Τ�`cf push`��ʹ�äƥ��ץꥱ�`������PCF�Ϥ˥ǥץ����ޤ���`cf target`�ǥ����󤬳����Ƥ��뤳�Ȥ�_�J���Ƥ���������
```shell
cf push api-<STUDENT_ID> -p target/demo-0.0.1-SNAPSHOT.jar
```

`tkaburagi`�β��֤������STUDENT_ID���ä��Q���Ƥ���������ͬ���쥹�ݥ󥹤����`�Ф��鷵�äƤ���Ǥ��礦��
```shell
curl https://api-tkaburagi.apps.pcfone.io | jq
```

���Ϥ�������ץꥱ�`�����Υǥץ��ϽK�ˤǤ���

**�����ޤ����ˤ������M�å��`�Ȥ˥����å��򤷤Ƥ���������**


## Exploring cf cli and Apps Manager
�����ǤϤ����Ĥ��λ����Ĥ�cf cli��ʹ�äƥ��ץꥱ�`������������Ƥߤޤ���

### �����Ĥ�cf cli
���ץꥱ�`�����򥹥��`�륢���Ȥ������Ȥ��ϡ�`cf scale`���ޥ�ɤ����ä��ޤ���
```shell
cf scale -i 2
curl https://api-tkaburagi.apps.pcf.pcflab.jp | jq
```
`curl`���ޥ�ɤ�ζȤ�ߵ���Ƥ������������ץꥱ�`����󤬥����`�륢���Ȥ���`index`�ˌ�����ؓ�ɷ�ɢ���Ƥ��뤳�Ȥ��狼��ޤ����ޤ������󥹥��󥹤��ۥ��Ȥ���Ƥ���`host`�΂���index���Ȥˉ仯���Ƥ��뤳�Ȥ�_�J���Ƥߤޤ��礦��

�Τˡ�`cf logs` ��ʹ�äƥ��ץ�Υ���ȡ�ä��ޤ���
```shell
cf logs api-tkaburagi --recent
```
`--recent`���ץ�����Ĥ����Loggregator�˥Хåե�����Ƥ��롢����Υ������פǤ��ޤ���
```shell
cf logs api-tkaburagi
```
���ץ����ʤ����ȡ��������ȥ�`�ߥ󥰤���ޤ�������״�B�Ǆe���`�ߥʥ��`curl`���ޥ�ɤ�g�Ф����ץ�Υ���ɥݥ���Ȥ˥����������ƤߤƤ���������

log�����ȥ�`�ߥ󥰤���Ƥ��뤳�Ȥ��狼��ޤ���PCF�Ǥϥ��ץ�������ǤϤʤ������`�Х��䥢���������ʤɤ򅧼����ޤ���

�������ˤ�`cf apps`��`cf app api-tkaburagi`�ʤɤ�g�Ф�����Է֤Υƥʥ�ȤΥ��ץ�һ�E��1���ץ��Ԕ�������Ҋ��ޤ���
```shell
cf apps
cf app api-tkaburagi
```

`cf -h`��g�Ф���ȥإ�פ��ʾ�Ǥ��ޤ��Τǡ��r�g�Τ��뷽��ɫ���ʥ��ޥ�ɤ�ԇ���ƤߤƤ���������

### �ޥ˥ե����Ȥ�ʹ�ä��ǥץ�
���¤�g�Ф���һ�ȥ��ץ���������ޤ���
```shell
cf delete api-tkaburagi
cf apps
```
`cf apps`��g�Ф���ȥ��ץ꤬��������Ƥ��뤳�Ȥ��狼��Ǥ��礦��

�Τˡ��ץ������ȥǥ��쥯�ȥ�ֱ�¤�`manifest.yml`��׷�Ӥ������¤Τ褦�˾������ޤ���
`- name:`��api-<STUDENT_ID>�ˤʤ�褦�����Q���Ƥ���������
```yaml
applications:
- name: api-tkaburagi
  buildpack: https://github.com/cloudfoundry/java-buildpack.git#v4.16
  path: target/demo-0.0.1-SNAPSHOT.jar
  memory: 1g
  env:
    JBP_CONFIG_OPEN_JDK_JRE: '{ jre: { version: 11.0.+}}'
```

�Ȥۤɤ�`cf push`�Υ��ץ�����ʹ�äƥ��ץ��ǥץ����ޤ�������manifest��ʹ�äƥǥץ����ޤ��������Ǥϥ��ץꥱ�`����������ӥ�ɥѥå������ץ�Υѥ������ä���JRE�ΥЩ`�����ʤɤ�ָ�����Ƥ��ޤ���

```shell
cf push
```
���Υޥ˥ե����Ȥ򥫥��ȥǥ��쥯�ȥ���ä�`cf push`����ȥǥ��쥯�ȥ�ֱ�¤�manifest���i���z�ߡ����ץ꤬�ǥץ�����ޤ����ޤ���`cf push -f path/tomanifest.yml`�Τ褦���Τǥޥ˥ե����Ȥ�ָ�����뤳�Ȥ�Ǥ��ޤ���

**�����ޤ����ˤ������M�å��`�Ȥ˥����å��򤷤Ƥ���������**

### PCF Apps Manager������
Web�֥饦����`https://apps.sys.pcf.pcflab.jp`�˥�����������cf cli��ͬ���˥����󤷤Ƥ���������������혤��v����һ�w�˥֥饦����������ޤ������äƤ��뷽�����ɤ�ʹ�ä�Ҋ�Ƥ���������
