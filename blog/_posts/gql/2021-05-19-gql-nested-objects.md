---
## Common
title: GraphQL backend — nested objects
tags: [kotlin, graphql]
description: >
    How GraphQL handles nested objects?
published: true

## Github pages
layout: post
image: /assets/img/nested_objects.jpg
sitemap: false
hide_last_modified: false
no_break_layout: false


## Dev.to
cover_image: https://mahendranv.github.io/assets/img/nested_objects.jpg
canonical_url: https://mahendranv.github.io/blog/2021-05-19-gql-nested-objects/
series: GraphQL backend

---

# GraphQL backend — nested objects


A nested object is like a [Matryoshka doll](https://en.wikipedia.org/wiki/Matryoshka_doll) where one object placed inside another and you can choose to not open the next level — a perfect usecase to demonstrate the power of GraphQL.

In this post we'll see how nested objects can be read from GraphQL server and how the mapping works here. For this purpose, I'm bringing in a new entity — called `Account`. For any expense or income, an account will be mapped to it. It is a Many (Expense)s to One (Account) mapping. This opens possibilities to few queries that involve both entities.

1. Get expense details — Include account info
2. For a given account, list expenses
3. Delete account and related expenses

We'll focus on the first one. Enough with the introduction, let's jump to the coding part.

* toc
{:toc}

## Account entity setup

Our account entity is a simple model which has three fields as in below block.

```kotlin
  data class Account(
      val acNumber: Int,
      val nickName: String,
      val balance: Int
  )
```

Complete Account Entity, Schema and Fetcher changes are present in this [single commit](https://github.com/mahendranv/spring-expenses/commit/158e9577b19304f7dc2e4ac5eb803dc6d2f6610f). Since we already covered CRUD on single entity, I'm fast-forwarding here. I left some sample queries on Account entity to play in GraphiQL (http://localhost:8080/graphiql).
 

```graphql

mutation CreateAccount {
  updateAccount(account: {acNumber: 5, nickName: "Retirement", balance: 10000}) {
    acNumber
    nickName
    balance
  }
}

query AllAccounts {
  accounts {
    acNumber
    nickName
    balance
  }
}

mutation DeleteAccount {
  deleteAccount(acNumber: 1)
}

```


So far our account and expense entities don't know each other. Let's introduce them. We have three stages in it, I made separate commits for each stage.

1. [Foreign key mapping & migration?!](https://github.com/mahendranv/spring-expenses/commit/e8d1795fdb4d90585f90d11a6060f55f2e5ca962)
2. [Nested object lookup](https://github.com/mahendranv/spring-expenses/commit/cf53cfb485b14a3518e2d33c9e231c06dc21f8e8)
3. [On demand data loading](https://github.com/mahendranv/spring-expenses/commit/500d9a5ddf25a3788a87366be7c58d7385275dad)

### Foreign key mapping
Before linking the entities let's understand how both of them are connected. In this case, given an expense it is connected to an account. For any account, there could be multiple expenses. That means a many-to-one relationship. By convension we represent this as a foreign key in Expense table.

![](/assets/img/2021-05-20-12-10-43.png)

***

Same can be represented in our expense entity as follows. 

```diff

 data class Expense(
@@ -11,4 +12,5 @@ data class Expense(
     val amount: Int,
     val remarks: String,
     val isIncome: Boolean,
+    val acNumber: Int
 )

```
If you've noticed, the `acNumber` is a non-nullable int field in expense. That means we have to ensure the same on Expense creation part. So our expense input and schema gets few changes as well.

```diff

## src/main/resources/schema/schema.graphqls
 input ExpenseInput {
     remarks: String
     amount: Int
+    acNumber: Int
 }

## src/main/kotlin/com/ex2/gql/expense/data/models/ExpenseInput.kt

 data class ExpenseInput(
     val amount: Int,
     val remarks: String,
+    val acNumber: Int
 )

```

We can reflect the same in create query, and include the `acNumber` to create an expense with mapping. That's about the creation part — now all the new expenses will be linked to an account. Note that, we haven't added any validation whether the account is present in the accounts list. Our focus is on read part, above creation flow is un-avoidable.

```graphql

mutation CreateExpense {
  createExpense(data: { remarks:"new expnse", amount: 122, isIncome: false, acNumber: 1}) {
    id
    remarks
  } 
}

```

Try the above mutation in GraphiQL to create expense. Next part is fetching it.

### Nested object lookup

Though, the object is represented as FK in our entity, what client really need is a fat-object that includes account in it. So, let's start edit the schema and bubble up the changes to the fetcher.

```diff

+++ b/src/main/resources/schema/schema.graphqls
type Expense {
     remarks: String
     amount: Int
     isIncome: Boolean
+    account: Account
 }

```

We've added the account field in expense, but the fetcher doesn't know it yet. It still returns the expense entity which has acNumber as forign key. To expand the same, create an intermediatery object that can directly map to the schema — lack of names calling it a `FatExpense`. This replaces the FK with actual object.

```kotlin

data class FatExpense(
    val id: Int,
    val amount: Int,
    val remarks: String,
    val isIncome: Boolean,
    val account: Account
)

```

Next, in the fetcher make changes to intialize the `account` field. This involves querying the accounts list per expense. So, the DAO — and fetcher changed as follows.

```kotlin

        // DAO
        fun getAccount(acNumber: Int): Account {
            println("Accounts.getAccount > $acNumber")
            return accounts.find { it.acNumber == acNumber }!!
        }

        // Fetcher
        fun expenses(): List<FatExpense> {
        return DataSource.expenses.map {
            FatExpense(
                id = it.id,
                amount = it.amount,
                remarks = it.remarks,
                isIncome = it.isIncome,
               account = DataSource.DAO.getAccount(it.acNumber)
           )
       }
     }

```

This should do it, head over to the GraphiQL and run a nested query. A simple query & response will look like this.

```json

query NestedExpense {
  expenses {
    remarks
    amount
    account {
    	acNumber
    }
  }
}

### Response
{
  "data": {
    "expenses": [
      {
        "remarks": "new expense",
        "amount": 122,
        "account": {
          "acNumber": "1"
        }
      },
  }
}

```

Nice! Is it over yet? **No**
Let's say I need only the amount and remark in my list. Following query should get it to me.

```json

query SimpleExpense {
  expenses {
    remarks
    amount
  }
}

## Response
{
  "data": {
    "expenses": [
      {
        "remarks": "new expense",
        "amount": 122
      }
  }
}

```

It still works as expected. What's wrong with it?
Checking the network logs, we can find the `SimpleExpense` query still probes the accounts list even though we havn't mentioned it in the selection [fields that we request from client]. This is bad for resource consumption. How do we fix it? Is GraphQL/DGS equipped with anything to help with it?

### On demand data loading

DGS can tell a fetcher what are all the fields that is present in the request. Same is available in the form of special parameter called `DgsDataFetchingEnvironment`. After adding it to our query and debug the paramter for selection, I found the selection three levels down.

```kotlin

    @DgsQuery
    fun expenses(dfe: DgsDataFetchingEnvironment): List<FatExpense> {
        // Should load account??
        val loadAccount = dfe.field
            .selectionSet
            .selections
            .any { field -> (field as? graphql.language.Field)?.name == "account" }
    }

```

`selections` is a tree like structure that will expand as query's depth. For this use-case, we're checking the field name `account`. Refer below screenshot for how selection is structured.

![](/assets/img/2021-05-20-13-57-09.png)

Now we know when to query for account, to implement it in fetcher `FatExpense#account` is made nullable — var, and on-demand assigned inside fetcher.

```diff

+++ b/src/main/kotlin/com/ex2/gql/expense/data/models/ExpenseInput.kt
@@ -22,5 +22,5 @@ data class FatExpense(
-    val account: Account
+    var account: Account?

+++ b/src/main/kotlin/com/ex2/gql/expense/fetchers/ExpenseDataFetcher.kt
        val expense = FatExpense(
                 ... ...
-                account = DataSource.DAO.getAccount(it.acNumber)
+                account = null
             )
+            val loadAccount = dfe.field...             
+            if (loadAccount) {
+                expense.account = DataSource.DAO.getAccount(it.acNumber)
+            }

```

Same query from above section won't invoke account query now. This is a first level of optimization over the nested graph — Nodes are fetched on-demand. However, there is still room for improvement. Look at the `NestedExpense` usecase, we're fetching the `Account`... that's expected. Problem is we're fetching the same account for multiple accounts. Spoiler alert — modern dbs caching this kind of query results to mitigate the performance impact. That doesn't mean we should code it this way.  

Let's catch up later.

🚀 Happy coding 🚀

