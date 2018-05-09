+++
author = "Nick Lanng"
categories = ["code", "fsharp", "mongodb"]
date = 2015-07-17T12:56:56Z
description = ""
draft = false
slug = "mongodb-driver-with-f"
tags = ["code", "fsharp", "mongodb"]
title = "MongoDB Driver with F#"

+++

I had a bit of difficulty today using MongoDB in F#.
This is just simple explanation of my 'Get all' function for posterity.

{{< highlight fsharp>}}
let getAllShips (database:IMongoDatabase) =
  let collection = database.GetCollection<BsonDocument> "ships"
  let wildcard = FilterDefinition<BsonDocument>.op_Implicit("{}")

  collection.Find(wildcard).ToListAsync()
  |> Async.AwaitTask
  |> Async.RunSynchronously
  |> Seq.map mapDocumentToShip
  |> List.ofSeq
{{< /highlight >}}

Let's go through this code line by line.

{{< highlight fsharp>}}
let getAllShips (database:IMongoDatabase) =
{{< /highlight >}}
In the new MongoDB Driver 2.x, MongoDatabase is exposed as IMongoDatabase. As everything is explicit in F# we have to use the interface here.

{{< highlight fsharp>}}
let collection = database.GetCollection<BsonDocument> "ships"
{{< /highlight >}}
I originally used the MongoDB.FSharp package, but this isn't yet updated to use the new driver. There are a couple of pull requests waiting to be merged to update the driver, but until that happens we have to do some things a little differently. The problem I was having is trying to Map directly to an F# record, it found the MongoDB record but all the values were null.

My MongoDB schema is designed for primarily for the REST API, which is a Node app using Mongoose ODM. The REST API will be reading the data a lot more often than this server app, so I prefer to map from the Mongoose schema to the F# records.

{{< highlight fsharp>}}
let wildcard = FilterDefinition<BsonDocument>.op_Implicit("{}")
{{< /highlight >}}
In MongoDB, you search by supplying an object with the fields you want to match in the returned results. To get all the records you need to pass an empty object. So this line is creating a filter for BsonDocument type objects (the base of all MongoDB objects) with an empty object.

{{< highlight fsharp>}}
collection.Find(wildcard).ToListAsync()
{{< /highlight >}}
Here we're submitting the request to MongoDB and the only method to return a list is asynchronous. This means we have to do a few extra things before we can map the BsonDocument to a Ship.

{{< highlight fsharp>}}
|> Async.AwaitTask
{{< /highlight >}}
This is a function that takes a System.Threading.Tasks.Task<T> and turns it into an Async<T>.

{{< highlight fsharp>}}
|> Async.RunSynchronously
{{< /highlight >}}
This is a function that runs the Async and returns the sequence of BsonDocuments.

{{< highlight fsharp>}}
|> Seq.map mapDocumentToShip
{{< /highlight >}}
Now we pass each BsonDocument through a function to return a Ship record.

{{< highlight fsharp>}}
|> List.ofSeq
{{< /highlight >}}
And then turn the Ships into a list, so the function returns a Ship List.
