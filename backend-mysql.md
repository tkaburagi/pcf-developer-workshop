# Using MySQL for PCF

```shell 
cf marketplace
```

```shell
cf create-service p.mysql small mysql
```

`pom.xml`に以下を追記します。
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
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.geode</groupId>
            <artifactId>spring-gemfire-starter</artifactId>
            <version>1.0.0.M3</version>
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

`src/main/java/com/example/demo`に新しいファイル`Book.java`を追加し下記のように編集します。
```java
package com.example.demo;

import org.springframework.data.annotation.Id;

import javax.persistence.Entity;
import javax.persistence.Table;

@Entity
@Table(name = "book")
public class Book {
    @Id
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
`src/main/java/com/example/demo`に新しいファイル`BookJpaRepository.java`を追加し下記のように編集します。
```java
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface BookJpaRepository extends JpaRepository<Book, String> {
    Book findBookById(final String id);
}
```

`src/main/java/com/example/demo`に新しいファイル`DbCloudConfig.java`を追加し下記のように編集します。
```java
package com.example.demo;

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

`Controller.java`を下記のように編集します。
```java
package com.example.demo;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class Controller {

    @Autowired
    BookJpaRepository bookJpaRepository;

    @RequestMapping("/hw")
    public String helloWolrd() {
        return "Hello world";
    }
    @RequestMapping(method = RequestMethod.GET, value = "/allbook")
    public String getAllBook() throws Exception{

        ObjectMapper mapper = new ObjectMapper();
        return mapper.writeValueAsString(bookJpaRepository.findAll());
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

```shell
./mvnw clean package -DskipTests=true && cf push -p target/demo-0.0.1-SNAPSHOT.jar --no-start
```

```shell
cf bind-service api-tkaburagi mysql
```

```shell
cf start api-tkaburagi
```