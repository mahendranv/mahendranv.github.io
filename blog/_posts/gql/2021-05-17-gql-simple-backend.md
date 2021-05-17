---
## Common
title: GraphQL - simple backend server - SpringBoot application
tags: [springboot, kotlin, graphql, beginner]
description: A simple SpringBoot application using in memory data source
published: true

## Github pages
layout: post
image: https://i.imgur.com/C1Xt6P8.png
sitemap: false
hide_last_modified: false
no_break_layout: false


## Dev.to
cover_image: https://i.imgur.com/C1Xt6P8.png
canonical_url: https://mahendranv.github.io/assets/covers/gql_adapter.jpg
series: GraphQL backend

---


# GraphQL - simple backend server - SpringBoot application

In this post, I'll cover how to create a simple GraphQL server with Springboot and Netflix DGS. DGS is chosen because you start with GraphQL schema and then build server side logic around it.



* toc
{:toc}



## Prerequisite

- Intellij IDEA

- GraphQL JS plugin

- [Kotlin fill class plugin](https://plugins.jetbrains.com/plugin/10942-kotlin-fill-class) - optional

  ---

## Initial setup

To start with SpringBoot https://start.spring.io/  has set of templates 

For springboot project, headover to https://start.spring.io/ and add *Spring Web* dependency. Select — Kotlin,  jar, Java 11 as project base in lefet pane. Hit Generate button and extract the downloaded zip to a project folder.



<img src="https://i.imgur.com/oZRPpFm.png" alt="image-20210517200554913" style="zoom:50%;" />

---

## Know the project structure

Our application is a single module project in the below tree format. Now it has bare minimum classes to just start the server.

```
├─build.gradle.kts
├─gradle
│  └─ wrapper
│      ├─ gradle-wrapper.jar
│      └─ gradle-wrapper.properties
├─gradlew
├─gradlew.bat
├─settings.gradle.kts
└─src
   ├─ main
   │   ├─ kotlin
   │   │   └─ com.ex2.gql.expense   //package
   │   │        └─ ExpenseApplication.kt
   │   └─ resources
   │       ├─ application.properties
   │       ├─ static
   │       └─ templates
   └─ test
       └─ kotlin
           └─ com.ex2.gql.expense
                └─ ExpenseApplicationTests.kt
```

*fig. Springboot - initial project structure*



Know about few files from the project structure.

1. `build.gralde.kts` - is where we add our project dependencies [database / logger / graphql]
2. `ExpenseApplication.kt` is the starting point of our application. Java `main` method is placed here
3. Files and folders of name `gradle*` are present here to build our app
4. `resources` directory is where we place out graphql schemas
5. `src/main/kotlin` is we actually code our business logic

---

## How to run the project?

Start terminal at the project root and run.

```shell

./gradlew bootRun

```



— or — 



For development, we'll use Intellij. So, select the `bootRun` gradle task in the gradle tool window and start.

<img src="https://i.imgur.com/KFHlW0l.png" alt="image-20210517213150720" style="zoom:56%;" />

On both cases, gradle wrapper will download all the dependencies (including the gradle distribution itself), and then start the application. 



## Netflix DGS framework setup

Netflix has released the dgs framework as open source on 2020. It powers the netflix API layer for over an year now. DGS plugin is available in mavencentral for us to use - Add the below dependency to your project.

```groovy

// https://mvnrepository.com/artifact/com.netflix.graphql.dgs/graphql-dgs-spring-boot-starter
	implementation("com.netflix.graphql.dgs:graphql-dgs-spring-boot-starter:3.12.1")

```



This plugin provides set of annotations that will connect our data fetchers / queries to our main application `ExpenseApplication` and generates `graphiql` playground for us to play around with our data.



## GraphQL schema setup

`schema` is the contract between client and server. DGS is a schema first framework. i.e we define a schema and DGS generate model classes for us. Let's start by creating a schema file in resource directory.



```

└─resources
    ├─ application.properties
    ├─ schema
    │   └─ schema.graphqls
    ├─ static
    └─ templates

```



Create file `schema.graphqls` in the `resources/schema` directory. And define an operation and a data model as below.



```graphql

type Query {
    expenses : [Expense]
}

type Expense {
    id: ID
    remarks: String
    amount: Int
    isIncome: Boolean
}

```



## Mapping java model to the schema

To get basic understanding on how model/entity is connected to the schema, Let's create a data class for `Expense` and corresponding fetcher. Fetchers are the helper classes that acts as an adapter between schema and data model.



```kotlin
//File:"ExpenseDataFetcher.kt"

@DgsComponent
class ExpenseDataFetcher {

    private val expenses = mutableListOf(
        Expense(id = 1, amount = 10, remarks = "Expense 1", isIncome = false),
        Expense(id = 2, amount = 120, remarks = "Expense 2", isIncome = false),
        Expense(id = 3, amount = 110, remarks = "Income 3", isIncome = false),
    )

    @DgsQuery
    fun expenses(): List<Expense> {
        return expenses
    }


    data class Expense(
        val id: Int,
        val amount: Int,
        val remarks: String,
        val isIncome: Boolean
    )
}

```



**Tips:** To fill in the data class, you can use the Fill data class plugin. It can generate default values for all the parameters. Inside the Expenses block invoke suggestions by `Alt(Option) + Return(Enter)` key.

<img src="https://i.imgur.com/rPEWs02.png" alt="image-20210517221425546" style="zoom:70%;" />



That's it!!

Now we have a working setup of a GraphQL query to play with. Annotations `DgsQuery` and `DgsComponent` will connect our model to the schema and register our component with main application.



## GraphiQL playground

Now run the application and visit http://localhost:8080/graphiql. You should see the GraphiQL page — A visual tool where you construct your queries and verify the fetcher.



```graphql

query MyQuery { 
  expenses {
    amount
    remarks
  }
}
```



<img src="https://i.imgur.com/LWg6gLp.png" alt="image-20210517223413283" style="zoom:50%;" />



Query call to the server
{:.figcaption}

---

## Adding a mutation

To add a mutation, head over to the schema file and add input type and a mutation for expense.

```graphql

type Mutation {
    createExpense(data: ExpenseInput) : Expense
}

input ExpenseInput {
    remarks: String
    amount: Int
    isIncome: Boolean
}

```



Add a bit of code to insert an element into the list. Give matching names and a method to create field [auto generated id]. And an annotation to connect function to the mutation query.

```kotlin
//File:"ExpenseDataFetcher.kt"

  private val random = Random(10000)
  private fun randomInt() = random.nextInt()

  @DgsMutation
  fun createExpense(data: ExpenseInput): Expense {
      val expense = Expense(
          id = randomInt(),
          amount = data.amount,
          remarks = data.remarks,
          isIncome = data.isIncome,
      )
      expenses.add(0, expense)
      return expense
  }

  data class ExpenseInput(
      val amount: Int,
      val remarks: String,
      val isIncome: Boolean
  )

```



Now try the mutation in GraphiQL playground. You should see the new Expense object is returned. Verify the entry added to the list by again running  `MyQuery  ` in the playground. 

```graphql

mutation MyMutation {
  createExpense(data: {remarks: "From GraphiQL", amount: 100, isIncome: true}) {
    id
    remarks
    amount
    isIncome
  }
}

```



![image-20210517231344271](https://i.imgur.com/BcADY41.png){:.lead loading="lazy"}



Mutation call to the server
{:.figcaption}

...

---

## Accessing the GraphQL endpoint in client apps and Insomnia / Postman



If you're fond of a REST/GraphQL client or to use the API in your client already, use the following endpoint. 

```

// Endpoint for Client
http://localhost:8080/graphql

// GraphiQL playground 
http://localhost:8080/graphiql

```

> Note there is no `i` in endpoint.



![image-20210517235204232](https://i.imgur.com/sNUqmuh.png)

GraphQL query in Insomnia
{:.figcaption}

---

## Source code

Entire source code is available in [Github](https://github.com/mahendranv/spring-expenses).





---

## Endnote

This the the bare minimum setup to make Create / Read requests to the backend using GraphQL. It is a steep learning curve to write queries for nested objects and connecting to a persistent database. Also there are other components available in DGS framework for codegen and custom scalars. I'll explore and add each usecase to this series.



