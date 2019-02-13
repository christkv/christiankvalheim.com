---
disqus_url: "http://christiankvalheim.com/post/the_new_bulk_api/"
disqus_title: "The New Bulk API"
author: "Christian Kvalheim"
date: 2014-09-25
title: The New Bulk API
weight: 10
authorAvatar: hugo-logo.png
keywords: ["Development","MongoDB","Node.js","github","mongodb","js","Javascript"]
tags: ["mongodb", "drivers", "node.js"]
slug: "the_new_bulk_api"
section: "post"
---

# The New Bulk API
One of the core new features in MongoDB 2.6 is the new bulk write operations. All the drivers include a new bulk api that allows applications to leverage these new operations using a fluid style API. Let's explore the API and how it's implemented in the Node.js driver.

## The API
The API have two core concepts. The **ordered** and the **unordered** bulk operation. The main difference is in the way the operations in a bulk are executed. In the case of an **ordered** bulk operation every operation will be executed in the order they are added to the bulk operation. In the case of an **unordered** bulk operation however there is no guarantee what order the operations are executed. Later we will look at how they both are implemented.

## Operations
You can initialize an **ordered** or **unordered** bulk operation in the following way.

    var ordered = db.collection('documents').initializeOrderedBulkOp();
    var unordered = db.collection('documents').initializeUnorderedBulkOp();

Once you have a bulk operation you can start adding operations to the bulk. The following operations are valid.

### updateOne (update first matching document)

    ordered.find({ a : 1 }).updateOne({$inc : {x : 1}});

### update (update all matching documents)

    ordered.find({ a : 1 }).update({$inc : {x : 2}});

### replaceOne (replace entire document)

    ordered.find({ a : 1 }).replaceOne({ x : 2});

### updateOne or upsert (update first existing document or upsert)

    ordered.find({ a : 2 }).upsert().updateOne({ $inc : { x : 1}});

### update or upsert (update all or upsert)

    ordered.find({ a : 2 }).upsert().update({ $inc : { x : 2}});

### replace or upsert (replace first document or upsert)

    ordered.find({ a : 2 }).upsert().replaceOne({ x : 3 });

### removeOne (remove the first document matching)

    ordered.find({ a : 2 }).removeOne();

### remove (remove all documents matching)

    ordered.find({ a : 1 }).remove();

### insert

    ordered.insert({ a : 5});

So what happens under the covers when you do start adding operations to a bulk operation. Let's take a look at the new write operations.

## The New Write Operations
MongoDB 2.6 introduces a completely new set of write operations. Before 2.6 all write operations where done using wire protocol messages at the socket level. From 2.6 this changes to using commands. So what does these commands look like.

### Insert Write Command
The insert write commands allow an application insert batches of documents. Let's take a look at the command and it's options.

    {
        insert: 'collection name'
      , documents: [{ a : 1}, ...]
      , writeConcern: {
        w: 1, j: true, wtimeout: 1000
      }
      , ordered: true/false
    }

A couple of things to note. The **documents** field contains an array of all the documents that are to be inserted. The **writeConcern** field specifies what would have previously been a **getLastError** command that would follow the pre 2.6 write operations. In other words there is always a response from a write operation in 2.6. This means that **w:0** has different semantics than what one is used to in pre 2.6. In the context **w:0** basically means only return an **ack** without any information about the **success** or **failure** of insert operations.

Now let's quickly look at the update and remove write commands before we take a look at the results that are returned when executing these operations against 2.6.

### Update Write Command
There are some slight differences in the update write command in comparison to the insert write command. Let's take a look.

    {
        update: 'collection name'
      , updates: [{ 
            q: { a : 1 }
          , u: { $inc : { x : 1}}
          , multi: true/false
          , upsert: true/false
        }, ...]
      , writeConcern: {
        w: 1, j: true, wtimeout: 1000
      }
      , ordered: true/false
    }

We notice that the main difference here is that the updates array is an array of update operations where each entry in the array contains the **q** field that specifies the selector for the update. The **u** contains the update operation. **multi** specifies if we will updateOne or updateAll documents that matches the selection. Finally **upsert** tells the server if it will perform an upsert if the document is not found. Finally let's look at the remove write command.

### Remove Write Command
The remove write command is very similar to the update write command. Let's take a look at it.

    {
        delete: 'collection name'
      , deletes: [{ 
            q: { a : 1 }
          , limit: 0/1
        }, ...]
      , writeConcern: {
        w: 1, j: true, wtimeout: 1000
      }
      , ordered: true/false
    }

Just an for updates we can see that the entries in the **deletes** array contain documents with specific fields. The **q** field is the selector that will match which documents will be removed. The **limit** field sets the number of elements to be remove. Currently **limit** only supports two values, 0 and 1. The value 0 for **limit** removes all documents that match the selector. A value of 1 for **limit** removes the first matching document only.

So how does the response look when executing the commands against the server.

### Write Command Results
One of the best new aspects of the new write commands is that they can return information about each individual operation error in the batch. To avoid not having to transmit more information than necessary only information about errors are returned as well as the aggregated counts of successful operations. Let's look at what a **comprehensive*** result could look like.

    {
      "ok" : 1,
      "n" : 0,
      "nModified": 1, (Applies only to update)
      "nRemoved": 1, (Applies only to removes)
      "writeErrors" : [
        {
          "index" : 0,
          "code" : 11000,
          "errmsg" : "insertDocument :: caused by :: 11000 E11000 duplicate key error index: t1.t.$a_1  dup key: { : 1.0 }"
        }
      ],
      writeConcernError: {
        code : 22,
        errInfo: { wtimeout : true },
        errmsg: "Could not replicate operation within requested timeout"
      }      
    }

The two most interesting fields here are **writeErrors** and **writeConcernError**. If we take a look at **writeErrors** we can see how it's an array of objects that include an **index** field as well as a **code** and **errmsg**. The **field** references the position of the failing document in the original **documents**, **updates** or **deletes** array allowing the application to identify the original batch document that failed.

### The Effect of Ordered (true/false)
The effect of setting **ordered** to true or false have a direct implication on how a write command is processed. Most importantly if **ordered** is set to **true** the write operation will fail on the first write error (meaning the first error that fails to apply the operation to memory). If one sets **ordered** to false however the operation will continue until all operations have been executed (potentially in parallel) and finally return all the results. **writeConcernError** on the other hand does not stop the processing of a bulk operation as the document did not fail to be written to MongoDB, the writeConcern could not be applied.

It helps to think of **writeErrors** as **hard** errors and **writeConcernError** as a soft error.

### The Special Case of w:0
The semantics for **w:0** changed for the write commands over the old style pre 2.6 write operations that are a combination of a write wire message and a **getLastError** command afterwards. In the old style **w:0** just meant that the driver would not send a **getLastError** command after the write operation. However all write commands respond and thus the old semantics for **w:0** are not possible to retain. The compromise is to make **w:0** mean I don't care about the results of the command just send me an **Ack**.

So if you execute.

    {
        insert: 'collection name'
      , documents: [{ a : 1}, ...]
      , writeConcern: {
        w: 0
      }
      , ordered: true/false
    }

All you receive from the server is the result

    {ok : 1}

## The Implication For The Bulk API
There are some implications to the fact that write commands are not mixed operations but either insert/update or removes. The Bulk API lets you mix operations and then merges the results back into a single result that simulates a mixed operations command in MongoDB. What does that mean in practice. Well let's look at how node.js implements **ordered** and **unordered** bulk operations. Let's use examples to show what happens.

### Ordered Operations
Let's take the following set off operations performed on a bulk

    var ordered = db.collection('documents').initializeOrderedBulkOp();
    ordered.insert({ a : 1 });
    ordered.find({ a : 1 }).update({ $inc: { x : 1 }});
    ordered.insert({ a: 2 });
    ordered.find({ a : 2 }).remove();
    ordered.insert({ a: 3 });

When running in ordered mode the bulk API guarantees the ordering of the operations and thus will execute this as 5 operations one after the other.

    insert bulk operation
    update bulk operation
    insert bulk operation
    remove bulk operation
    insert bulk operation

We have now reduced the bulk API to performing single operations and your throughput suffers accordingly.

If we re-order our bulk operations in the following way.

    var ordered = db.collection('documents').initializeOrderedBulkOp();
    ordered.insert({ a : 1 });
    ordered.insert({ a: 2 });
    ordered.insert({ a: 3 });
    ordered.find({ a : 1 }).update({ $inc: { x : 1 }});
    ordered.find({ a : 2 }).remove();

The execution is reduced to the following operations one after the other.

    insert bulk operation
    update bulk operation
    remove bulk operation

Thus for ordered bulk operations the ordered of operations will impact the number of write commands that need to be executed and thus the throughput possible.

### Unordered Operations
Unordered operations have not guarantee about the execution order of operations. Let's take the operations example from above.

    var ordered = db.collection('documents').initializeOrderedBulkOp();
    ordered.insert({ a : 1 });
    ordered.find({ a : 1 }).update({ $inc: { x : 1 }});
    ordered.insert({ a: 2 });
    ordered.find({ a : 2 }).remove();
    ordered.insert({ a: 3 });

The Node.js driver will collect the operations into separate type specific operations. So we get.

    insert bulk operation
    update bulk operation
    remove bulk operation

In difference to the **ordered** operation these bulks all get executed in parallel in Node.js and the results then merged when they have all finished.

## Takeaway
Due to MongoDb not currently implementing a mixed write operations command it's important to keep this in mind when performing **ordered** bulk operations to avoid the scenario above and take a hit on the write throughput. In the case of **unordered** operations the missing mixed operations command does not impact the throughput.

One thing to note. Although the Bulk API actually supports downconversion to 2.4 the performance impact is considerable as all operations are reduced to single write operations with a **getLastError**. It's recommended to leverage this API primarily with 2.6 or higher.