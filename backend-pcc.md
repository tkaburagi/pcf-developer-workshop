# Using Pivotal Cloud Cache

```shell 
cf marketplace
```

```shell
cf create-service p-cloudcache extra-small pcc
cf create-service-key pcc pcc-svc-key
```

`cf create-servie`は少し時間がかかります。

```shell
cf service-key pcc pcc-svc-key
cf bind-service api-tkaburagi pcc
```

```shell
$ gfsh
$ gfsh> connect --use-http=true --use-ssl --url=<GFSH_URL> --user=***** --password=*****
$ gfsh> create region --name=book --type=REPLICATE
$ gfsh> list regions
```

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

`CloudCacheConfig.java`を追加し、下記のように編集します。
```java
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;
import org.springframework.data.gemfire.cache.config.EnableGemfireCaching;
import org.springframework.data.gemfire.config.annotation.EnableEntityDefinedRegions;
import org.springframework.data.gemfire.repository.config.EnableGemfireRepositories;

@EnableGemfireCaching
@EnableEntityDefinedRegions(basePackages = "com.example.demo.model")
@EnableGemfireRepositories(basePackages = "com.example.demo.repo.gem")
@Profile("cloud")
@Configuration
public class PccConfig {
}
```

`BookGemfireRepository`を追加し、下記のように編集します。
```java
import com.example.demo.model.Book;
import org.springframework.data.gemfire.repository.GemfireRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface BookGemFireRepository extends GemfireRepository<Book, String> {

}
```

`BookService.java`を追加し、下記のように編集します。
```java
import com.example.demo.model.Book;
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

`Controller.java`を下記のように編集します。
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

    @RequestMapping("/hw")
    public String helloWolrd() {
        return "Hello world";
    }

    @RequestMapping(method = RequestMethod.GET, value = "/allbook")
    public String getAllBook() throws Exception {

        ObjectMapper mapper = new ObjectMapper();
        return mapper.writeValueAsString(bookJpaRepository.findAll());

    }

    @RequestMapping(method = RequestMethod.GET, value = "/book")
    public String getBookById(@RequestParam(value = "id") String id) {

        return bookService.getBookById(id).getTitle() + " from " + bookService.isCacheMiss();
    }

}
```

```shell
./mvnw clean package -DskipTests=true && cf push
```


