# Using MySQL for PCF

```shell 
cf marketplace
```

```shell
cf create-service
```

`pom.xml`に以下を追記します。
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
```

`src/main/java/com/example/demo`に新しいファイル`Config.java`を追加し下記のように編集します。
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

    public Book() {

    }

    public Book(String id, String title, String author_name, String price) {
        this.id = id;
        this.title = title;
        this.author_name = author_name;
        this.price = price;
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
`src/main/java/com/example/demo`に新しいファイル`BookJpaRepository.java`を追加し下記のように編集します。
```java
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository("BookJpaRepository")
public interface BookJpaRepository extends JpaRepository<Book, String> {

    Book findBookById(final String id);

}
```

`src/main/resources`に新しいファイル`demo.sql`を追加し下記のように編集します。
```sql
CREATE TABLE book
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
cf push --no-start
```

```shell
cf bind-service
```

```shell
cf start
```