+++
date = '2024-05-13T19:45:00+01:00'
draft = false
title = 'MongoDB Transaction Read Locks in Go'
tags = ["Blog", "English", "IT", "Go", "Learn"]
[params]
comments = true
+++

Sometimes, things do not go well on the coding level. In that case, we know what to do - fix the issue, get to another task. Sometimes, things don't work out on the architectural level. For such stuff we have to apply additional resources to work around it. Fixing it won't make the tech debt disappear though.

One of such issues was our MongoDB schema design. Most of us acknowledge the fact that a noSQL DB should be denormalized to be used efficiently. This convention was overlooked in one of the projects, which led us to a normalized, relational-like schema. And it actually worked fine, until it didn't. Remember ACID?

<!--more-->

## How transactions work and why we might need them

With MongoDB 4.0, [multi-document ACID transactions were introduced](https://www.mongodb.com/products/capabilities/transactions), raising the question: how did we manage without them before? Note the term "multi-document" - Mongo already guaranteed a single document atomicity before 4.0, and if we think about it, we don't require something else to ensure that our denormalized nested documents changes are queued up (especially in the distributed environment).

Thus, we would refer to multi-document transactions only in case of relational ideas sneaking into our MongoDB architecture, just as in our project's case. Consider two ~~tables~~ collections: "movies" and "directors":

```
,-------------.
|movies       |
|-------------|
|id: int      |
|name: string |
|director: int|
`-------------'
       |       
       |       
,------------. 
|directors   | 
|------------| 
|id: int     | 
|name: string| 
`------------'
```

If app A deletes a director record while app B tries to reference it during a new movie record creation, then we would obviously end up with the inconsistent state of DB. We shall use [multi-document transactions](https://www.mongodb.com/docs/manual/core/transactions/) in this very relational case. No wonder they promote transactions feature for banking and healthcare apps, which usually tend to be designed with relational approach in mind.

OK, problem solved? Not exactly. Although we are covered for any `write` and `update` operations, what about reads?

## Problem of read locks

Let's take a look at the [test code](https://github.com/gren236/mongo-transaction-test) prepared. It comes with a `docker-compose.yml` file to spin up a test MongoDB container, and also with `test-init.js` file that provides initial seed records for testing. It creates two collections:

* `foo` with the entry `{"_id":"6475eb087660882fa85dff59","hello":"world"}`
* `bar` with the entry `{"_id":"6475ebec7c8c0d02309b0a46","answer":42}`

Imagine we have a business logic in our app that requires value `world` to be present in the field `hello` (`foo` collection) in order to update field `answer` with value `43` (`bar` collection)

The example code, found at [snapshot-concern/main.go](https://github.com/gren236/mongo-transaction-test/blob/master/snapshot-concern/main.go), spawns two goroutines: session A and session B.

```go
var wg sync.WaitGroup

// Do session A
wg.Add(1)
go func() {
	...
}()

// Do session B
wg.Add(1)
go func() {
	...
}()
```

Let's pretend those are two different apps, or even two replicas of the same app in a cluster.

There are also two channels created - `aChan` and `bChan`.

```go
aChan := make(chan bool)
bChan := make(chan bool)
```

Their job is to indicate when we would like to lock one goroutine's execution and switch to another.

The overall flow of the test is:

* Start the multi-document transaction in session A
  ```go
  res, err := session.WithTransaction(ctx, func(sessCtx mongo.SessionContext) (interface{}, error) {...}
  ```
* Session A reads the value of `hello` field from `foo`'s collection document
  ```go
  fooRec := fooColl.FindOne(sessCtx, bson.D{{"_id", "6475eb087660882fa85dff59"}})
  fooRec.Decode(&fooRes)
  ```
* Session A stops the execution and switches to the session B
  ```go
  aChan <- true
  ```
* Session B updates the only document of collection `foo` and changes `hello` field value from `world` to `bar`
  ```go
  _, dErr := fooColl.UpdateOne(  
      sessCtx,  
      bson.D{{"_id", "6475eb087660882fa85dff59"}},  
      bson.D{{"$set", bson.D{{"hello", "bar"}}}},  
  )
  ```
* Session A continues the execution and checks if the value of the field `hello` is `world` successfully (!)
  ```go
  if t := <-bChan; t {  
      if fooRes.Hello != "world" {  
        return nil, fmt.Errorf("failed to compare foo record")  
      }
      
      ...
  }
  ```
* Session A updates field `answer` with `43` in collection `bar`, which contradicts our business logic
  ```go
  _, dErr := barColl.UpdateOne(  
      sessCtx,  
      bson.D{{"_id", "6475ebec7c8c0d02309b0a46"}},  
      bson.D{{"$set", bson.D{{"answer", 43}}}},  
  )
  ```

As we can see, nothing prevents us from reading the value in one transaction and committing the update in another, which leads to an inconsistency. This would not happen with `write` operations, as Mongo would just acquire the `writeLock` for documents we create or change in the context of a single transaction, preventing others from changing it as well. So, how do we deal with that?

## Solution

The only reasonable solution I found (besides finally making an effort to denormalize documents) is described in [the MongoDB blog post by Renato Riccio](https://www.mongodb.com/blog/post/how-to-select--for-update-inside-mongodb-transactions). The general idea here is to "abuse" the `writeLock` functionality and acquire it even for `read` operations. In my opinion this method sounds more like a workaround than an actual solution, but I guess we have to pay the price for being too relational 😉

In the scope of Go application, we could wrap our calls to Mongo driver's `FindOne` function like that:

```go
func FindOneLock(  
    coll *mongo.Collection,  
    ctx context.Context,  
    filter interface{},  
    opts ...*options.FindOneOptions,  
) *mongo.SingleResult {  
    // TODO map options.FindOneOptions to options.FindOneAndUpdateOptions  
    return coll.FindOneAndUpdate(ctx, filter, bson.D{{  
       "$set",  
       bson.D{{"lockRandom", primitive.NewObjectID()}},  
    }})  
}
```

Thus, every time we read any document in the transaction, a field is created in this document with some random ID hash. We have to use a random value here, because the `update` function is lazy and checks if the value is actually **changed** before making any updates to the document.

Another good idea would be to add a check to make sure that we do this only in the context of the transaction:

```go
switch ctx.(type) {  
case mongo.SessionContext:  
    ...  
default:  
    ...  
}
```

Generally, that sounds lot like `SELECT ... FOR UPDATE` from the relational databases world, which is rather our goal here.

## Conclusion

I can't call this approach "optimal" or even "valid", but if we are stuck with the relational model - there are not many options. Feel free to express other ideas how to deal with that and, please, make sure to use noSQL conventions when designing your DB structure.
