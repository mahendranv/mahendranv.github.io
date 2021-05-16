---

layout: post
title: Making the most out of GraphQL - Type Adapters
image: /assets/covers/gql_adapter.jpg
description: >
  GraphQL can take care of code-gen. But, how do I tell my date format to GraphQL?
sitemap: false
hide_last_modified: false
tags: [android, kotlin, graphql]
---

# Making the most out of GraphQL - Type Adapters

GraphQL is smart enough to generate classes & parsers for your primitive / nested objects. However there are cases where GQL cannot determine what to do with a field. In such cases, we can lend a hand and get type safe fields in return. Our GraphQL client is Apollo.



We'll peek though the schema & query files that drives the codegen and get it to bend for our need. This post covers a special data type `timestampz` which needs some care from developer to translate into concrete data type.


<!-- ![graphql-type-adapter]() -->

![image](https://user-images.githubusercontent.com/6584143/118398586-e8057500-b676-11eb-9d9d-5265513422f1.png)



For the implementation, skip to [ the how to section](#how-do-i-translate-timestamp-to-date-object-).

* toc
{:toc}


---



## Pre-requisite

Basic understanding of what is GraphQL. Having a working GraphQL android codebase is even better. Read [this](https://dev.to/mahendranv/android-api-calls-to-graphql-server-49a1) for setting up the *ApolloClient* and how to write query for a given schema. Also, Apollo has a [quick guide](https://www.apollographql.com/docs/android/essentials/get-started-kotlin/) to get started with GraphQL.



---

## Data types in GraphQL

A GraphQL query is made of nodes and leaves (referred as `scalars`) too often. There are default scalars defined in system as such, `Int, Float, String & Boolean`. However like in our case, backend can mark fields as custom scalars to expect client side processing.



```
expenses {         ## node
    id
    amount
    remarks
    is_income
    created_at     ## Custom scalar
}
```



Here in the above query we have `created_at` maked with `timestamptz`. Let's scan through schema.json for how it looks.

---

## Peek to the schema



```json
// Example : default scalar
{
    "args": [],
    "isDeprecated": false,
    "deprecationReason": null,
    "name": "remarks",
    "type": {
        "kind": "NON_NULL",
        "name": null,
        "ofType": {
            "kind": "SCALAR",
            "name": "String",   // Known date type
            "ofType": null
        }
    },
    "description": "Where the money went"
},

// Example : custom scalar
{
    "name": "created_at",
    "type": {
        "kind": "NON_NULL",
        "name": null,
        "ofType": {
            "kind": "SCALAR",
            "name": "timestamptz",
            "ofType": null
        }
    },
    "description": "Created timestamp"
},
```




As we can see, the timestamp cannot fall into a known scalar platform specific implementation for below reasons

1. The plugin just don't know what's the date format is. There is no single format universally agreed for DateTime communication. Moreover, it can be just date or time. So it's up to the dev to decide on this.
2. Plenty of options out there when it comes to Date and time. Based on personal preference, one can go for Jave Date, Calendar, DateTime, Joda(don't use this), three-ten-bp or kotlinx datetime Api.

---

## How codegen handles custom scalars?

Apollo generates an enum to book-keep the custom scalars. It maps the scalar name to corresponding  (fully qualified) type name known in the platform. Without any type adapters, it look like this.



```kotlin
// CustomTypes.kt

import com.apollographql.apollo.api.ScalarType
import kotlin.String

enum class CustomType : ScalarType {

  TIMESTAMPTZ {
    override fun typeName(): String = "timestamptz"

    override fun className(): String = "kotlin.Any"
  }
}
```



Generated class for Expense node look like this. Currently the `created_at` is of type `Any`, waiting for us to make it concrete. Scaning through the the file, you'll find where it is serialized and deserialized.

```kotlin
// Expenses.kt

data class Expense(
    val __typename: String = "expenses",
    val id: Int,
    val amount: Int,
    val remarks: String,
    val is_income: Boolean,
    val created_at: Any
  ) {
    fun marshaller(): ResponseFieldMarshaller = ResponseFieldMarshaller.invoke { writer ->
			...
      writer.writeString(RESPONSE_FIELDS[3], this@Expense.remarks)
			...
      // Serializing cusom fields here                                                                          
      writer.writeCustom(RESPONSE_FIELDS[5] as ResponseField.CustomTypeField,
          this@Expense.created_at)
    }

    companion object {
      private val RESPONSE_FIELDS: Array<ResponseField> = arrayOf(
          ...
          ResponseField.forString("remarks", "remarks", null, false, null),
					...
 
          // Definition of custom response field
          ResponseField.forCustomType("created_at", "created_at", null, false,
              CustomType.TIMESTAMPTZ, null)
          )

      operator fun invoke(reader: ResponseReader): Expense = reader.run {
        ...
        val remarks = readString(RESPONSE_FIELDS[3])!!
        val is_income = readBoolean(RESPONSE_FIELDS[4])!!
        // De-Serializing the field here
        val created_at = readCustomType<Any>(RESPONSE_FIELDS[5] as ResponseField.CustomTypeField)!!
        Expense(
					...
        )
      }
      ...
    }
  }
```



Though, on surface `created_on` marked as `Any` field, under the hood it leaves markers indicating this type can be changed to some concrete type.

Let's convert the `timestampz`.

---

## How do I translate `timestamp` to Date object? <a name="implementation">
To convert the `timestampz` to Date, we need both compiler and apollo client to work together. Solution consist of three parts.

1. Choosing concrete type for custom scalar

2. Mapping it in code-gen

3. Attaching adapter to the apollo client

   

###  1. Choosing concrete type for custom scalar - [threetenbp lib]
My choice of DateTime library here is `threetenbp`. This ports Java SE 8 LocalDateTime classes to java 6 and 7. My initial feasibility check with `kotlinx-datetime` results didn't look so good as custom formatting of date time is not supported yet.



Add this gradle dependency to your app module.

```groovy

// https://mvnrepository.com/artifact/org.threeten/threetenbp
implementation "org.threeten:threetenbp:1.5.1"

```






### 2. Code-gen changes - Apollo compile time setup to map timestamp
Compile time setup is rather simple. We have to map the fully qualified target class name against our custom scalar. This is enough to convert our scalar to LocalDateTime. 



```groovy
// fie: app/build.gradle

apollo {
    generateKotlinModels.set(true)
    customTypeMapping = [
            "timestamptz" : "org.threeten.bp.LocalDateTime"
    ]
}

dependencies {
...
```



A peek through updated `CustomType` enum changes.


```diff
   TIMESTAMPTZ {
     override fun typeName(): String = "timestamptz"
 
-    override fun className(): String = "kotlin.Any"
+    override fun className(): String = "org.threeten.bp.LocalDateTime"
```



And in `Expense` class, the field type and deserializer changed.

```diff
--- 
+++ 
import org.threeten.bp.LocalDateTime

@@ -1,13 +1,13 @@
 data class Expense(
     val __typename: String = "expenses",
     val id: Int,
     val amount: Int,
     val remarks: String,
     val is_income: Boolean,
-    val created_at: Any
+    val created_at: LocalDateTime // Now it's concrete
   ) {

         // Field deserialization - now takes LocalDateTime
         // instead of Any
-        val created_at = readCustomType<Any>(RESPONSE_FIELDS[5] as
+        val created_at = readCustomType<LocalDateTime>(RESPONSE_FIELDS[5] as
             ResponseField.CustomTypeField)!!
```



Now we have the type changed in generated classes. Yet, at runtime apollo doesn't know how to create a `LocalDateTime` object from a `timestamptz`. Let's provide the adapter in next step.




### 3. Apollo runtime setup for parsing date
This step is to convert string to Date and vice-verse. Threetenbp has `DateTimeFormatter` formatter class for this purpose. Let's have a look at our sample  date-time from our backend

```

2021-05-13T04:00:49.815194+00:00

```



This is of pattern `yyyy-MM-dd'T'HH:mm:ss.SSSSSSz`. Let's create a formatter for this.



```kotlin

val formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ss.SSSSSSz")

```



Now we create `CustomTypeAdapter` object to attach with apollo client. It has `decode` - `encode` functions to carry out the conversions. Using the above formatter, we can create an adapter like this,



```kotlin
/**
 * Timestamp delivered as string from API. This adapter takes care of encode / decode the same.
 */
private val timeStampAdapter = object : CustomTypeAdapter<LocalDateTime> {

    private val formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ss.SSSSSSz")

    override fun decode(value: CustomTypeValue<*>): LocalDateTime {
        return try {
            LocalDateTime.parse(value.value.toString(), formatter)
        } catch (e: Exception) {
            throw RuntimeException("Cannot parse date: ${value.value}")
        }
    }

    override fun encode(value: LocalDateTime): CustomTypeValue<*> {
        return try {
            CustomTypeValue.GraphQLString(formatter.format(value))
        } catch (e: Exception) {
            throw RuntimeException("Cannot serialize  the date date: ${value}")
        }
    }
}
```



Now map the adapter to the enum through `ApolloClient.Builder`.



```kotlin

val apolloClient = ApolloClient
        .builder()
        .serverUrl("// url //")
        .addCustomTypeAdapter(CustomType.TIMESTAMPTZ, timeStampAdapter)
        .build()

```



That's it!!  Now ApolloClient knows timestamp is LocalDateTime. 



---
## Endnote
Now we've seen how to write adapters for a scalar, you might be in urge to create adpaters for Enums. Don't do it. Push it back to the backend and ask them to define it in schema. Apollo can generate enums if it is defined properly.