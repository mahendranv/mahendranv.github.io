---
## Common
title: GraphQL backend — data loaders
tags: [graphql, springboot, kotlin]
categories: [GraphQL]
description: >
    How to make use of graphql data loaders to write declarative data fetchers?
published: true

## Github pages
layout: post
image: /assets/img/data_loaders.jpg
sitemap: false
hide_last_modified: false
no_break_layout: false


## Dev.to
# cover_image: 
# canonical_url: 
# series: GraphQL backend

---

In the [previous post](https://mahendranv.github.io/posts/2021-05-20-gql-nested-objects/), I created a FatExpense object and added manual checks to avoid fetching Account entity. If we're scanning the selection manually and write our own check, what's the role of GraphQL here?

Actually GraphQL handles all this, I just wanted show the problem of doing it manually. Let's see what is what and what does each component do.

Here we're looking at two problems
1. We have to create Entity to match our schema model [FatExpense]
2. For each Expense — a query made to Account data source [i.e N+1 problem]

Let me explain what is N+1 problem in breif and we can jump to the implementaion.

* toc
{:toc}

## The N+1 problem

N+1 is when you make N queries from the outer object to get inner object. So the number of total queries made is 1 (outer object) + N (one query per row). In our example, we make 1 query to fetch Expense list and N queries to fill in account inside each expense. As nesting increase, it affects performance of the system.

Our *mock data set* is designed in a way that, all the expenses shared between three accounts. We consider our solution is a win when we achieve max 3 calls to the accounts source instead of 100 (one per expense).
```kotlin

    val accounts = mutableListOf(
        Account(acNumber = 1, nickName = "Wallet", balance = 100000),
        Account(acNumber = 2, nickName = "Axis", balance = 240000),
        Account(acNumber = 3, nickName = "ICICI", balance = 28000),
    )

    val expenses = mutableListOf<Expense>()

    private val random = Random(10000)
    private fun randomExpense() = random.nextInt(from = 100, until = 800)

    init {
        for (i in 1..100) {
            expenses.add(
                Expense(
                    id = i,
                    amount = randomExpense().absoluteValue,
                    remarks = "Transaction $i",
                    isIncome = (i % 2 == 0),
                    acNumber = (i % 3 + 1)
                )
            )
        }
    }

```

## 👨‍💻 Code it

We need three components to decouple our entities and fix N+1 problem.
1. DataSource that supports batch fetch
2. A DataLoader
3. DataFetcher that carries the context

![](/assets/img/2021-05-21-23-13-33.png)

### Preparing your datasource
To alleviate the N fetches, what we can do is to find unique set of account numbers and then fetch them in one go. To achieve it, our data source has to support batch fetch of accounts as unlike our `get acccount by id` which deliver a single row of account. Let's add it.

``` kotlin

    fun getAccounts(acNumbers: List<Int>): List<Account> {
        println("DAO.getAccounts - $acNumbers")
        return accounts.filter {
            acNumbers.contains(it.acNumber)
        }
    }

```

### Create a batch loader
A batch loader is a component in GraphQL that extracts unique set of keys from series of requests and make a batch request to the data source. This way the resulting query is optimized to contain unique key/ids and only one fetch made to the data source.

```kotlin

@DgsDataLoader(name = "AccountNumberToAccount")
class AccountsDataLoader : BatchLoader<Int, Account> {

    override fun load(keys: MutableList<Int>?)
            : CompletionStage<MutableList<Account>> {
        return CompletableFuture.supplyAsync {
            return@supplyAsync DataSource.DAO
                .getAccounts(keys ?: emptyList())
                .toMutableList()
        }
    }
}

```

In the above code block, we register `AccountsDataLoader` to DGS registry under the alias of `AccountNumberToAccount`. This name acts like an identifier in `DgsDataFetchingEnvironment` to obtain the DataLoader.

It is a parameterized class, that maps Int (Account number) to Account entity and we have a `load` function that takes in set of unique account numbers and convert them to `Future`, that will be exucuted in batches. This `Future` nature of content fetch encourages GraphQLs async nature of resolving a query.

> **Why it is a CompletableFuture instead of upfront fetch?**
> <br>Assume each request from server to database takes 10ms. 100 sequntial calls to the database will endup in 1000 ms latency for a single field. Making the loader asynchronous enables us to fill in other fields for the response dynamically and deliver the result faster.
{:.lead}

### Throw away the fat-object
If we create a wrapper entity for each nested query, we'll endup creating a bunch of entities (or an obese one). DGS got it covered for us. Just delete the FatExpense and update the ExpenseDataFetcher with the original entity.

```diff

+++ b/src/main/kotlin/com/ex2/gql/expense/fetchers/ExpenseDataFetcher.kt
 
     @DgsData(parentType = "Query", field = "expenses")
-    fun expenses(dfe: DgsDataFetchingEnvironment): List<FatExpense> {
-        return DataSource.expenses.map {
-            val result = FatExpense(
-                id = it.id,
-                amount = it.amount,
-                remarks = it.remarks,
-                isIncome = it.isIncome,
-                account = null
-            )
-
-            val loadAccount = dfe.field
-                .selectionSet
-                .selections
-                .any { field -> (field as? graphql.language.Field)?.name == "account" }
-
-            if (loadAccount) {
-                result.account = DataSource.DAO.getAccount(it.acNumber)
-            }
-            result
-        }
+    fun expenses(dfe: DgsDataFetchingEnvironment): List<Expense> {
+        return DataSource.expenses
     }

```

### Tell DGS how to fetch Account

Now that we deleted the `FatExpense` class, we have to wire up the connection between Expense and Account. We do it in Account Data fetcher. First revisit the schema before do the linking.

```graphql
type Expense {
    ...
    account: Account
}
```

We have a type called Expense and it holds a field `account` of type Account. Inside DGSData annotation, mention the parentType and which field in the schema it resolves. And pass down the dfe (DgsDataFetchingEnvironment) context for fetching Account.

```kotlin

    @DgsData(parentType = "Expense", field = "account")
    fun getAccount(dfe: DgsDataFetchingEnvironment): CompletableFuture<Account> {
        val dataLoader: DataLoader<Int, Account> = dfe.getDataLoader(AccountsDataLoader::class.java)
        val source = dfe.getSource<Expense>()
        val acNumber = source.acNumber
        return dataLoader.load(acNumber)
    }

```

> `dfe` passes context information such as source from which this request originated and selection (fields required in account entity) to the called function.

Inside the function body, from `DgsDataFetchingEnvironment` we get a loader for AccountsDataLoader. It is important to use dfe to create the loader, as it sets scope for the data loader. This helps to reuse the same loader for each Expense that tries to load account information. `dfe` carries the callers scope that we can get using `getSource`. From there, we can retrieve the account number and queue account fetch request. Just leaving a note here, the parentType


That's it, we don't have to make any check on `selection` or create an entity that matches the schema or manually club the N+1 queries. A data loader and a `DgsData` annotation will handle scoping — de-duplicating & delivering result in async manner.



## 🚀 Launch it

Launch `http://localhost:8080/graphiql` in browser and feed the below query. And check the `DataSource.DAO#getAccounts` logs.

```graphql

query Expense {
  expenses {
    id
    remarks
    amount
    account {
      acNumber
      nickName
    }
  }
}

### Logs

DAO.getAccounts - [2, 3, 1]

```

We have a single fetch request made to the account data source with a batch of account numbers. 


### How does it scale?
Let's say we have another data type called `Transfer` with `from` and `to` — both fields of type Account. All we need to do is to declare two functions for `from` and `to` fields. Rest of the DataLoader and DataSource changes can be reused.

```kotlin
    @DgsData(parentType = "Transfer", field = "from")
    ...
    @DgsData(parentType = "Transfer", field = "to")
    ...

```


## Endnote
With three components added to the codebase (two of them reusable), we have a decoupled yet context aware data loading in our system. Here we explored `BatchLoader` — one of the commonly used loader. There are other types such as `BatchLoaderWithContext`, `MappedBatchLoader` and `MappedBatchLoaderWithContext` each tailored to address specific usecase. Make use of them and enjoy GraphQL*ing* your backend. 

DataLoader is part of [java-graphql](https://github.com/graphql-java) project and not specific to DGS. Role of DGS here is to provide a solid registry for data fetchers & loaders.

## Resources
* [Commit: Migrating to DataLoader](https://github.com/mahendranv/spring-expenses/commit/ffa51a9cf081772c72002dfca61541f252cbe925)
* [Spring-expenses Repository](https://github.com/mahendranv/spring-expenses)
* [java-dataloader](https://github.com/graphql-java/java-dataloader)