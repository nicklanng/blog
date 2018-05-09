+++
author = "Nick Lanng"
categories = ["code", "architecture", "csharp"]
date = 2015-09-24T16:01:36Z
description = ""
draft = false
slug = "data-access-patterns"
tags = ["code", "architecture", "csharp"]
title = "Data Access Patterns"

+++

I recently got into a conversation with a colleague about system design and testability. One of the fun asides we got into was about data storage and why we use repositories. Below is my best attempt at explaining what I consider to be the main three data access strategies used in modern times.

* [Repositories](#repositories)
* [Query Objects](#queryObjects)
* [Active Records](#activeRecords)

This code is written using [Dapper](https://github.com/StackExchange/dapper-dot-net) and [AutoMapper](https://github.com/AutoMapper/AutoMapper). MySQL connections are created in the service layer, coupling the service with the persistence technology; you don't have to do this but it keeps the examples concise.

What you will probably find from the code examples is that, although the code differences are quite minimal, the architectural considerations vary quite widely between the strategies.

<a name="repositories"></a>
## Repositories

Repositories are great for containing your [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) operations for entities. You can think of an entity as a thing or object in your domain model, some examples could be User, Article or Order. Another rule of thumb is that an entity has to have an Id, not a database auto-increment for linking but a semantically valid Id in the domain. Having all of the database methods for that entity in one class is quite neat.

Sometimes, you need to query the database for something that isn't an entity; perhaps there's some metadata or link table that isn't necessarily part of the entity but doesn't constitute an entity of its own. Sometimes you need to calculate some information that isn't directly modeled in your entities, an example could be that you have two statistics records and you want to find the average time between those two records. A statistics record isn't an entity and the information you want isn't modeled. It's situations like these that can really start to make Repositories muddy and difficult to maintain.

Even when dealing with an entity, there may be a number of variations of the CRUD methods. Often you'll see GetById, GetByName and GetByEmail methods. InsertOrUpdate is another method that can show up. All of these things start to obfuscate the simple CRUD nature of a repository and the class begins to grow and grow. Conversely, sometimes a repository doesn't have all four of the basic CRUD operations because [You Aint Gonna Need](https://en.wikipedia.org/wiki/You_aren%27t_gonna_need_it) one of them. This is fine but when another developer comes along and that method doesn't exist it can cause a cognitive disconnect and start to brew feelings of hindsight bias.

### Public Service Layer

{{< highlight csharp>}}
public class StarshipService : IStarshipService
{
    public IEnumerable<StarshipDTO> GetPagedStarships(long pageNumber)
    {
        var starshipRepository = new StarshipRepository();

        using (var connection = new SqlConnection(DatabaseConfiguration.ConnectionString))
        {
            var starshipsResult = starshipRepository.GetStarships(connection, 10, pageNumber);
            return Mapper.Map<IEnumerable<StarshipEntity>, IEnumerable<StarshipDTO>>(starshipsResult);
        }
    }

    public void CreateStarship(StarshipDTO starship)
    {
        var entity = Mapper.Map<StarshipEntity>(starship);
        var starshipRepository = new StarshipRepository();

        using (var connection = new SqlConnection(DatabaseConfiguration.ConnectionString))
        {
            starshipRepository.Save(connection, entity);
        }
    }

    public void UpdateStarship(StarshipDTO starship)
    {
        var entity = Mapper.Map<StarshipEntity>(starship);
        var starshipRepository = new StarshipRepository();

        using (var connection = new SqlConnection(DatabaseConfiguration.ConnectionString))
        {
            starshipRepository.Update(connection, entity);
        }
    }
}
{{< /highlight >}}

### Data Access

The entity in this example is very lightweight. More often, the entity contains domain logic as well as its basic fields.

{{< highlight csharp>}}
internal class StarshipRepository
{
    public IEnumerable<StarshipEntity> GetStarships(IDbConnection connection, long pageNumber, long pageSize)
    {
        return connection.Query(
                @"SELECT Id, Name, PositionX, PositionY, PositionZ
                  FROM Starship
                  ORDER BY Name
                  LIMIT @pageSize OFFSET @offset;",
                new
                {
                    pageSize,
                    offset = pageNumber * pageSize
                })
            .Select(
                row => new StarshipEntity
                {
                    Id = row.Id,
                    Name = row.Name,
                    Position = new Vector3
                    {
                        X = row.PositionX,
                        Y = row.PositionY,
                        Z = row.PositionZ
                    }
                }
            ).ToList();
    }

    public void Update(IDbConnection connection, StarshipEntity entity)
    {
        connection.Execute(
            @"UPDATE Starship
                SET
                Name = @name,
                PositionX = @positionX,
                PositionY = @positionY,
                PositionZ = @positionZ
                WHERE ID = @id",
            new
            {
                id = entity.Id,
                name = entity.Name,
                positionX = entity.Position.X,
                positionY = entity.Position.Y,
                positionZ = entity.Position.Z
            });
    }

    public void Save(SqlConnection connection, StarshipEntity entity)
    {
        connection.Execute(
            @"INSERT INTO Starship (Name, PositionX, PositionY, PositionZ)
                VALUES (@name, @positionX, @positionY, @positionZ);",
            new
            {
                name = entity.Name,
                positionX = entity.Position.X,
                positionY = entity.Position.Y,
                positionZ = entity.Position.Z
            });
    }
}

internal class StarshipEntity
{
    public long Id { get; set; }
    public string Name { get; set; }
    public Vector3 Position { get; set; }
}
{{< /highlight >}}

<a name="queryObjects"></a>
## Query Objects

Using the query object pattern, every database query and command is modeled as a separate class. This is the ultimate expression of the [Single Responsibility Principle](https://en.wikipedia.org/wiki/Single_responsibility_principle). As with anything there are trade-offs to consider with this approach.

A query object can represent any query, not just your standard entity CRUD operations. Want to get information about your products? Create a query object. Want to get information about customer review of a product? Create a query object. Your classes do not have to directly represent the database structure at all, but they can if you want.

Occasionally you may not need every bit of information that an entity might give you, if a User is logging in to your system why do you need their real name and address? You only need to fetch their username and hashed password. When listing all of your products, do you need the customer ratings, stock levels, price history or just the id, name and current price? Different parts of the system require different views over the same data and with Query Objects you're not struggling with entity representations to get the information you want.

Another example is getting calculated information from your data. Going back to my previous example, if you have a statistics record of something starting and a record of something finishing, a query object could fetch the average of all the times between those two records. This is the sort of query thats much harder to build with repositories.

As the database commands can also be modeled separately, you don't have to update every field every time like you would with repositories or active record. Instead of updating an entire product you can create an UpdateProductPriceCommand object that takes a product id and the new price. This can help reduce database load and also reduces the chance for other fields to be updated by mistake.

An important consideration with this approach is managing the sheer number of classes you might create. If your project is particularly large then you may end up with hundreds of these little objects. If you can't break your project up into smaller ones then there might be considerable cognitive overhead trying to keep your objects neat. Organization and naming conventions will help you here. You'll want to minimize the chances of a developer creating a new query object because they can't find the one that already does what they need.

It is worth noting that these approaches are not mutually exclusive, I have often seen repositories handling domain entities backed up with query objects for the less standard queries and it's worked quite well.

### Public Service Layer

{{< highlight csharp>}}
public class StarshipService : IStarshipService
{
    public IEnumerable<StarshipDTO> GetPagedStarships(long pageNumber)
    {
        var getStarshipsQuery = new GetStarshipsQuery { PageNumber = pageNumber };

        using (var connection = new SqlConnection(DatabaseConfiguration.ConnectionString))
        {
            var starshipsResult = getStarshipsQuery.Execute(connection);
            return Mapper.Map<IEnumerable<StarshipResult>, IEnumerable<StarshipDTO>>(starshipsResult);
        }
    }

    public void CreateStarship(StarshipDTO starship)
    {
        var createRequest = Mapper.Map<CreateStarshipRequest>(starship);
        var createCommand = new CreateStarshipCommand();

        using (var connection = new SqlConnection(DatabaseConfiguration.ConnectionString))
        {
            createCommand.Execute(connection, createRequest);
        }
    }

    public void UpdateStarship(StarshipDTO starship)
    {
        var updateRequest = Mapper.Map<UpdateStarshipRequest>(starship);
        var updateCommand = new UpdateStarshipCommand();

        using (var connection = new SqlConnection(DatabaseConfiguration.ConnectionString))
        {
            updateCommand.Execute(connection, updateRequest);
        }
    }
}
{{< /highlight >}}

### Data Access

Each query object or command has an associated class to act as input or output of the command. You may be tempted to mix this concept with the DTOs coming out of the service layer, but even if they look the same they are semantically different. Also, if your service needs to use multiple query objects then the service needs to compose the results into a single DTO to pass back.

{{< highlight csharp>}}
internal class GetStarshipsQuery
{
    public long PageSize { get; set; } = 10;
    public long PageNumber { get; set; } = 0;

    public IEnumerable<StarshipResult> Execute(IDbConnection connection)
    {
        return connection
            .Query(@"SELECT Id, Name, PositionX, PositionY, PositionZ
                        FROM Starship
                        ORDER BY Name
                        LIMIT @pageSize OFFSET @offset;", new { pageSize = PageSize, offset = PageNumber * PageSize})
            .Select(
                row => new StarshipResult
                {
                    Id = row.Id,
                    Name = row.Name,
                    Position = new Vector3
                    {
                        X = row.PositionX,
                        Y = row.PositionY,
                        Z = row.PositionZ
                    }
                }
            ).ToList();
    }
}

internal class StarshipResult
{
    public long Id { get; set; }
    public string Name { get; set; }
    public Vector3 Position { get; set; }
}
{{< /highlight >}}

{{< highlight csharp>}}
internal class CreateStarshipCommand
{
    public void Execute(IDbConnection connection, CreateStarshipRequest request)
    {
        connection.Execute(
            @"INSERT INTO Starship (Name, PositionX, PositionY, PositionZ)
                VALUES (@name, @positionX, @positionY, @positionZ);",
            new
            {
                name = request.Name,
                positionX = request.Position.X,
                positionY = request.Position.Y,
                positionZ = request.Position.Z
            });
    }
}

internal class CreateStarshipRequest
{
    public string Name { get; set; }
    public Vector3 Position { get; set; }
}
{{< /highlight >}}


{{< highlight csharp>}}
internal class UpdateStarshipCommand
{
    public void Execute(IDbConnection connection, UpdateStarshipRequest request)
    {
        connection.Execute(
            @"UPDATE Starship
                SET
                Name = @name,
                PositionX = @positionX,
                PositionY = @positionY,
                PositionZ = @positionZ
                WHERE Id = @id;",
            new
            {
                id = request.Id,
                name = request.Name,
                positionX = request.Position.X,
                positionY = request.Position.Y,
                positionZ = request.Position.Z
            }
        );
    }
}

internal class UpdateStarshipRequest
{
    public long Id { get; set; }
    public string Name { get; set; }
    public Vector3 Position { get; set; }
}
{{< /highlight >}}

<a name="activeRecords"></a>
## Active Records

Active record has fallen out of favor in recent times but was once the go-to method for object persistence. I think the reasons for this are in the areas of testability and single responsibility principle but let's discuss it in more depth.

What should be glaringly obvious from the example active record below is that it's doing two things, holding the fields for the record and also talking to the database. This often feel like two responsibilities and if you're in the mindset of query objects, even more than that. Further to this, you will notice that the lines of code for database access greatly outnumber the lines for the fields, so much so that you could even miss the fields if you weren't looking for them. Ok sure, you could inherit from a base class that magically hands the database parts, but let's be honest - this only hides that the class is doing too much, which may be fine for you.

There are a couple of static methods in this example. You need to be able to fetch active records from the database without first having an active record. In this case it's common to put a static method on the active record class that will return instances of the record. This can cause issues for some testing strategies, although consider whether you really want to mock out all database access anyway. There are probably other ways to solve this issue without static methods although they would probably stray into the realms of the other strategies I've described.

On a personal note, I can't help but appreciate how neat usages of active records are. Getting a record, updating a field and calling save on it seems very pretty. My example of updating an active record updates every field but you don't have to do this, you can consider the UpdateStarship method messy or clear; this is probably up to you and your team.

### Public Service Layer

{{< highlight csharp>}}
public class StarshipService : IStarshipService
{
    public IEnumerable<StarshipDTO> GetPagedStarships(long pageNumber)
    {
        using (var connection = new SqlConnection(DatabaseConfiguration.ConnectionString))
        {
            var starshipsResult = StarshipActiveRecord.GetStarships(connection, 10, pageNumber);
            return Mapper.Map<IEnumerable<StarshipActiveRecord>, IEnumerable<StarshipDTO>>(starshipsResult);
        }
    }

    public void CreateStarship(StarshipDTO starship)
    {
        var record = new StarshipActiveRecord
        {
            Name = starship.Name,
            Position = new Vector3
            {
                X = starship.PositionX,
                Y = starship.PositionY,
                Z = starship.PositionZ
            }
        };

        using (var connection = new SqlConnection(DatabaseConfiguration.ConnectionString))
        {
            record.Insert(connection);
        }
    }

    public void UpdateStarship(StarshipDTO starship)
    {
        using (var connection = new SqlConnection(DatabaseConfiguration.ConnectionString))
        {
            var record = StarshipActiveRecord.GetById(connection, starship.Id);
            record.Name = starship.Name;
            record.Position.X = starship.PositionX;
            record.Position.Y = starship.PositionY;
            record.Position.Z = starship.PositionZ;
            record.Update(connection);
        }
    }
}
{{< /highlight >}}

### Data Access

{{< highlight csharp>}}
internal class StarshipActiveRecord
{
    public long Id { get; set; }
    public string Name { get; set; }
    public Vector3 Position { get; set; }

    internal static IEnumerable<StarshipActiveRecord> GetStarships(IDbConnection connection, long pageNumber, long pageSize)
    {
        return connection
            .Query(@"SELECT Id, Name, PositionX, PositionY, PositionZ
                    FROM Starship
                    ORDER BY Name
                    LIMIT @pageSize OFFSET @offset;", new { pageSize, offset = pageNumber * pageSize })
            .Select(
                row => new StarshipActiveRecord
                {
                    Id = row.Id,
                    Name = row.Name,
                    Position = new Vector3
                    {
                        X = row.PositionX,
                        Y = row.PositionY,
                        Z = row.PositionZ
                    }
                }
            ).ToList();
    }

    internal static StarshipActiveRecord GetById(IDbConnection connection, long id)
    {
        return connection
            .Query(@"SELECT Id, Name, PositionX, PositionY, PositionZ
                    FROM Starship
                    WHERE Id = @id", new { id })
            .Select(
                row => new StarshipActiveRecord
                {
                    Id = row.Id,
                    Name = row.Name,
                    Position = new Vector3
                    {
                        X = row.PositionX,
                        Y = row.PositionY,
                        Z = row.PositionZ
                    }
                }
            ).Single();
    }

    public void Insert(SqlConnection connection)
    {
        connection.Execute(
            @"INSERT INTO Starship (Name, PositionX, PositionY, PositionZ)
              VALUES (@name, @positionX, @positionY, @positionZ);",
            new
            {
                name = Name,
                positionX = Position.X,
                positionY = Position.Y,
                positionZ = Position.Z
            });
    }

    public void Update(SqlConnection connection)
    {
        connection.Execute(
            @"UPDATE Starship
              SET
                Name = @name,
                PositionX = @positionX,
                PositionY = @positionY,
                PositionZ = @positionZ
              WHERE Id = @id",
            new
            {
                id = Id,
                name = Name,
                positionX = Position.X,
                positionY = Position.Y,
                positionZ = Position.Z
            });
    }
}
{{< /highlight >}}

--------

##### Disclaimer
* All examples are for demonstration purposes only; I haven't compiled it. Don't copy this code into your own projects. Read it, understand it and implement it yourself.
* I haven't included the code for the [DTO](https://en.wikipedia.org/wiki/Data_transfer_object) but just know that this is a [POCO](https://en.wikipedia.org/wiki/Plain_Old_CLR_Object) with basic data type fields.
