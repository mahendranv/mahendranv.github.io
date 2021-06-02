---
## Common
title: GraphQL backend — pagination & filters
tags: [graphql, springboot, kotlin]
description: GraphQL pagination and filter setup
published: true

## Github pages
layout: post
image: /assets/img/covers/gql_filters_pagination.jpg
sitemap: false
hide_last_modified: false
no_break_layout: false


## Dev.to
cover_image: https://mahendranv.github.io/assets/img/covers/gql_filters_pagination.jpg
canonical_url: https://mahendranv.github.io/posts/2021-05-25-gql-filters-pagination/
series: GraphQL backend

---

# GraphQL backend — pagination & filters

## Background
Pagination and filters are the much needed components when it comes to listing. They not only narrow down the list for the user, but improves load time and save user from scanning through lots of pages. Overall content engagement will improve when the list is on point and incrementally loaded in small chunks. This post covers setup & implementation of pagination in springboot-graphql environment.

Sample project used in this post is available in [Github](https://github.com/mahendranv/spring-expenses). Project overall architucture is in below diagram.

![](/assets/img/2021-05-26-00-03-41.png)

* toc
{:toc}

## Project overview
Basic project setup involves a database, ORM and graphql to compose the response. *H2* is a lightweight relational database that is a goto option for casual projects. ORM — [Object–relational mapping] is a technique that connects object to the database. With ORM, database can be accessed without the need of writing any query. *JPA* — (Java Persistence API) is used in this project for accessing the database. *Netflix-DGS* framework is used for GraphQL setup — it is a superset of java-graphql implementation.

## Project setup

### Database
H2 database requires way less setup compared to others. H2 is added as a runtime dependency to the application and will be availble to use when the application starts. Following dependency enables the h2-driver to connect with our database.

```groovy
dependencies {
 ...
 runtimeOnly("com.h2database:h2")

}

```

Now that our app knows the H2 driver, it needs the database url and credentials to connect with it. Adding the following properties tells springboot to use h2 driver and use the given credentials.

```properties
//File:"application.properties"

# Driver
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.url=jdbc:h2:file:~/be/db/expenses;AUTO_SERVER=true
spring.datasource.username=sa
spring.datasource.password=password
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect

# Table creation from entity classes
spring.jpa.hibernate.ddl-auto=update

# H2 console
# http://localhost:8080/h2-console/
spring.h2.console.enabled=true

```

The driver section is straightforward where the app reads db url and credentials. H2 database can be used as an in-memory datasource as well as it can persist the data to a `.db` file. For the url `~/be/db/expenses` a local file `expenses.mv.db` created in local storage.

 In the second section, `spring.jpa.hibernate.ddl-auto` property tells springboot to create tables out of the ORM Entity classes. For this example entity class called Expense is considered. Few annotations added to the class / fields to make up table and it's columns.

```kotlin

@Entity(name = "expenses")
data class Expense(

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Int? = 0,

    @Column(nullable = false, name = "amount")
    val amount: Int? = null,

    @Column(name = "remarks", nullable = false)
    val remarks: String? = null,

    @Column(name = "is_income", nullable = false)
    val isIncome: Boolean? = false
)

```

The `H2 Console` section enables the web interface to the h2 database. It will be available at `http://localhost:8080/h2-console/` once the application started. With current setup, it should show the `expenses` table with no records in it.

![](/assets/img/2021-05-25-22-13-58.png)

{:.figcaption}

H2 console web interface

### ORM — JPA repository

JPA can be added to the project with following gradle dependency.
```kotlin
//File: build.gradle.kts

// JPA
implementation("org.springframework.boot:spring-boot-starter-data-jpa:2.5.0")

```

In springboot, repository is an interface makes a skeleton for entity and it's primary key. The repository for the expenses entity is a simple one line interface. Based on the need, complex queries can be declared in the repository.

```kotlin

interface ExpenseRepository : JpaRepository<Expense, Int>

```

On top of common CRUD operations, JpaRepository covers sort/filter/pagination usecases as well. To give an idea on what are all the operations that we get from JPA, pasted the original JpaRepository interface below. 

```java

public interface JpaRepository<T, ID> extends PagingAndSortingRepository<T, ID>, QueryByExampleExecutor<T> {
   List<T> findAll();
   List<T> findAll(Sort sort);
   List<T> findAllById(Iterable<ID> ids);   
   <S extends T> List<S> saveAll(Iterable<S> entities);
   void flush();
   <S extends T> S saveAndFlush(S entity);
   void deleteInBatch(Iterable<T> entities);
   void deleteAllInBatch();
   T getOne(ID id);
   <S extends T> List<S> findAll(Example<S> example);
   <S extends T> List<S> findAll(Example<S> example, Sort sort);
}

```


### GraphQL setup

For GraphQL, Netflix DGS is used in this project, for two main reasons. It generates models out of schema & it registers fetchers and data loaders with few annotations. Following changes required in build file to add dgs to the project.

```kotlin
plugins {

    // Code generation plugin
    id("com.netflix.dgs.codegen") version "4.6.4"

}

tasks.withType<com.netflix.graphql.dgs.codegen.gradle.GenerateJavaTask> {
     packageName = "com.ex2.gql.dgmodels"
     schemaPaths = mutableListOf("${projectDir}/src/main/resources/schema")
}

dependencies {
    implementation("com.netflix.graphql.dgs:graphql-dgs-spring-boot-starter:3.12.1")
}

```

Here, the `com.netflix.dgs.codegen` plugin reads schema file placed in resource directory and generate classes. `com.netflix.graphql.dgs:graphql-dgs-spring-boot-starter` takes care of connecting the data fetchers(REST controller equivalent) & loaders (layers in between fetcher and repository) to the main application.

## Pagination and Filter setup


### What paginates the data source?
The key player in this post is the following method in JPARepository. Although this post drives towards GraphQL, any form of API can make use of the core logic in below method.

```java

//File:"QueryByExampleExecutor.java"

   <S extends T> Page<S> findAll(Example<S> example, Pageable pageable);

```

This method takes in a probe object to filter the data source with and directives for paging and sorting.  This probe object can be the one extended from our entity or entity object itself. Any field that is not required for filtering, can be set to null. 

`findAll` delivers result in a wrapper class called `Page`, which contains the subset (a chunk of items) and blueprint of over all data set (total number of pages). This total pages will be used at the client end to stop fetching further from the list.


### Schema
In GraphQL schema drives the rest of the project, the surface setup and then the underlying machanics are built. DGS framework takes care of code generation when the schema file is present in right resource folder. 

For this post, expected model is a page of expenses which provides meta info regarding the total number of pages and current page number. The schema will look like this.

```graphql

input ExpenseFilter {
    isIncome: Boolean
}

type Expense {
    id: ID
    remarks: String
    amount: Int
    isIncome: Boolean
}

type ExpensePage {
    list: [Expense]
    totalPages: Int
    currentPage: Int
}

type Query {

    fetchExpenses(filter: ExpenseFilter, pageNumber: Int, pageSize: Int) : ExpensePage
}

```

Here, `Expense` denotes single item in expenses list, `ExpensePage` is a wrapper that holds list of expenses and page info. Provided these types, dgs-codegn plugin will generate classes based on type and field names. These classes will be used in later sections to connect with ORM. 

`ExpenseFilter` is a class that represents filter for expense entity, more on this covered in data fetchers section. `fetchExpenses` is a query operation for GraphQL, it takes filter, pageNumber and pageSize as arguments and returns ExpensePage.

```kotlin
// Generated — Expense Page class
public data class ExpensePage(
  @JsonProperty("list")
  public val list: List<Expense?>? = null,
  @JsonProperty("totalPages")
  public val totalPages: Int? = null,
  @JsonProperty("currentPage")
  public val currentPage: Int? = null
) {
  public companion object
}

```

### Data fetchers

A DataFetcher is where GraphQL compose the response from entity. Query mentioned in `Query#fetchExpenses` is materialized here to fetch data from the database. These classes must be marked as `DgsComponent` to be considered as a graphql component. It can contain number of graphQL operations annotated matching the signature in schema. To connect with database, `ExpenseRepository` reference is created to use inside the data fetcher.

```kotlin

@DgsComponent
class ExpenseDataFetcher {

    @Autowired
    private lateinit var expenseRepository: ExpenseRepository

```

`fetchExpenses` operation can be hooked to a method as in below. Arguments are appropriately annotated with field names from the schema. Input and output classes are the DGS generated ones.


```kotlin
    @DgsData(parentType = "Query", field = "expenses")
    fun expenses(
        @InputArgument("filter") filter: ExpenseFilter?,
        @InputArgument("pageNumber") pageNumber: Int,
        @InputArgument("pageSize") pageSize: Int
    ): ExpensePage {
```

**Filter** for JPA can be provided by creating an example object with matching values and passes to the `expenseRepository.findAll` function. If `null` field passed to the example object, it will not be considered for filtering. 

The findAll function also takes in a `PageRequest` object as second argument to provide **pagination**. This page request requires pageNumber, size and an optional Sort parameter.


```kotlin
        val probe = Expense(
            id = null,
            remarks = null,
            isIncome = filter?.isIncome,
            amount = null
        )
        val example = Example.of(probe)
        val pageRequest = PageRequest.of(
            pageNumber, pageSize, Sort.by(
                Sort.Order.desc("id")
            )
        )

        val result: Page<Expense> = expenseRepository.findAll(example, pageRequest)
```

So far, the result is of type `Page<Expense>`, this is close to the `ExpensePage` defined in schema but not exactly the same.  So, the last part is to convert JPAEntity to GraphQL — Object. For this matter, a utility class called `ExpenseGEMapper` is used, it basically reads each field from Jpa model and assigns it to graphql model. Implementation can be found below.


```kotlin
        val list = result.content.map {
            ExpenseGEMapper.toGraph(it)
        }

        return ExpensePage(
            list = list,
            totalPages = result.totalPages,
            currentPage = pageNumber
        )
    }

```


```kotlin

//File:"ExpenseGEMapper.kt"

import com.ex2.gql.expense.jpa.entities.Expense as EExpense
import com.ex2.gql.dgmodels.types.Expense as GExpense

object ExpenseGEMapper : GraphEntityMapper<GExpense, EExpense> {

    override fun toGraph(e: EExpense): GExpense {
        return GExpense(
            id = e.id?.toString(),
            remarks = e.remarks,
            acNumber = e.acNumber,
            isIncome = e.isIncome,
            amount = e.amount,
            account = null
        )
    }

    override fun toEntity(g: GExpense): EExpense {
        return EExpense(
            id = g.id?.toInt(),
            remarks = g.remarks,
            acNumber = g.acNumber,
            isIncome = g.isIncome,
            amount = g.amount
        )
    }
}

```

With above function, the pagination & filter is complete. The expense data fetcher class is available [here](https://github.com/mahendranv/spring-expenses/blob/main/src/main/kotlin/com/ex2/gql/expense/fetchers/ExpenseDataFetcher.kt) for reference.

## Run it

GraphiQL playground is available in `http://localhost:8080/graphiql` to try the queries. Given the filter/page details, it returs the list, and page number to query further in the list. Clients can query until `totalPages`.

```json

query Expense {
  fetchExpenses(filter: {isIncome: false}, pageSize: 10, pageNumber: 0) {
    list {
      id
      remarks
      amount      
    }
    totalPages
    currentPage
  }
}

### response

{
  "data": {
    "expenses": {
      "list": [
        {
          "id": "295",
          "remarks": "Book MK",
          "amount": 90
        }
        ...
        {
          "id": "286",
          "remarks": "Book MK",
          "amount": 90
        }
      ],
      "totalPages": 27,
      "currentPage": 0
    }
  }
}

```



## Summary
A graphQL schema, ORM and data fetcher work together to make an API paginated. As number of filter fields increase, GraphQL will scale gracefully with very little code change where only the ExpenseFilter to Example class conversion is affected.