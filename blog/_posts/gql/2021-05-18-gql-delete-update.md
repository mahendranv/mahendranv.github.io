---
## Common
title: GraphQL backend — update & delete
tags: [kotlin, graphql]
description: >
    Second installment in GraphQL backend setup. Here we cover delete and update operations.
published: true

## Github pages
layout: post
# image: 
sitemap: false
hide_last_modified: false
no_break_layout: false


## Dev.to
# cover_image: 
# canonical_url: 
series: GraphQL backend

---

# GraphQL backend — update & delete

In the first installment, I covered create and read operations to the Expense entity. Now let's add few more operations ⇨ update and delete queries.

Entire source code is available in [github](https://github.com/mahendranv/spring-expenses). For intial setup read this [post](https://mahendranv.github.io/blog/2021-05-17-gql-simple-backend/). As Netflix — DGS propose, start with schema and then edit the fetcher.

## ⌨️ Code it

Open `schema.json` and add these mutations. They're not treated differently from the `createExpense` mutation. Whatever write operation, it is treated as mutation. Let's update the mutations.

```graphql
//File: "schema.json"

type Mutation {
    createExpense(data: ExpenseInput) : Expense
    deleteExpense(id: ID): Boolean
    updateExpense(expense: UpdatedExpense): Expense
}

input UpdatedExpense {
    id: ID
    remarks: String
    amount: Int
    isIncome: Boolean
}
```

Corresponding query fetchers are pretty straightforward.

```kotlin
  
  // Deletes an expense if any and return true or false based on it.
  @DgsMutation
  fun deleteExpense(id: Int): Boolean {
      return expenses.removeIf { it.id == id }
  }

  @DgsMutation
  fun updateExpense(expense: Expense): Expense {
      // This is an upsert operation. If id already present in list
      // overwrite it / otherwise insert at 0th index
      val index = expenses.indexOfFirst { it.id == expense.id }
      if (index != -1) {
          expenses[index] = expense
      } else {
          expenses.add(0, expense)
      }
      return expense
  }

```

Notice that we have separate input type defined in schema for `updateExpense` mutation while the mutation reuses the same type?

This is because, GrpahQL expects separate entities created for input and data models. Wheras in the dataftcher, only constraint is to match the data model to schema.


## 🚀 Run it

I've listed few queries that you can run in our server.

- Run `./gradlew bootRun` to start the server
- Open `http://localhost:8080/graphiql` in the browser and try these queries

```graphql

# Creates an expense with this input
mutation CreateExpense {
  createExpense(data: {remarks: "From GraphiQL", amount: 100, isIncome: true}) {
    id
    remarks
    amount
    isIncome
  }
}

# Lists all the expenses
query ListExpenses {
  expenses {
    id
    remarks
    amount
    isIncome
  }
}

# Deleting an expense
mutation DeleteItem {
  deleted: deleteExpense(id : 1)
}

# Update / insert an expense based on id
mutation UpdateItem {
  
  updateExpense(expense: {id: 2, remarks: "From GraphiQL", amount: 100, isIncome: true} ) {
    id
    remarks
    amount
    isIncome
  }
}


```

It is rather simple change that we did, and now we have completed CRUD operations on a simple data set. Changes are available in a single commit [here](https://github.com/mahendranv/spring-expenses/commit/206150c32083b1f917048c522a734bfad6273da8?branch=206150c32083b1f917048c522a734bfad6273da8&diff=unified).


## Endnote

This is a short post, it just gives you an idea on how delete / update operations can be written using GraphQL. This whole article has only one take, that we don't have to create `UpdatedExpense` class in data fetcher to run the mutation. Any data model that matches the field type and argument name, will do just fine.

In future articles in the series, I'll explore nested objects and how effective GraphQL is when it comes to resource consumption.