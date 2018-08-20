# Ooorm.Data Introduction

Ooorm.Data is a "code first" data access library for new dotnet applications. It generates database schemas and repositories from dotnet types allowing for a large reduction in development overhead and complexity. 

The projects goals are simplicity and enabling rapid, iterative software design. High performance is **not** a design goal. Big data support is **not** a design goal.

A typical table definition may look like this

```cs
namespace MyProject.Models
{
    using Ooorm.Data;

    public class ToDoItem : IDbItem
    {
        public int ID { get; set; }
        public string Text { get; set; }
    }

    public class ToDoComment : IDbItem
    {
        public int ID { get; set; }
        public DbVal<ToDoItem> ItemId { get; set; }
        public string AuthorName { get; set; }
        public string Text { get; set; }
    }
}
```

Note the `DbVal<ToDoItem>` property in `ToDoComment`. Foriegn key relations are defined using strongly typed wrappers on integer IDs to enable the use of rich IDE features like 'Go to definition' and 'Find All References' in addition to enabling more expressive, semantic data access logic. 

An example use of these tables may look something like this. 

```cs
public async Task<IEnumerable<ToDoItem>> ItemsAuthorHasCommentedOn(string author, IDatabase db)
    => (await new ToDoComment{ AuthorName = author }
            .ReadMatchingFrom(db))
            .Select(async comment => comment.ItemId.Get());

```

Connecting to a database gives a generic `IDatabase` instance. 

```cs
// Mostly complete:
// Sql Server
IDatabase sqlDb = new SqlDatabase(SqlConnection.CreateShared("Server=localhost;Database=TestDb;Integrated Security=True;"));

// Work In Progress:
// Sql Lite
IDatabase sqliteDb = new SqliteDatabase(SqliteConnection.CreateFromFile("mydata.sqlite"));
// In memory database
IDatabase volatileDb = new VolatileDatabase();
// Synced database that caches values and is notified of changes (backed by another db server side)
IDatabase signalrDb = new SignalrDb(SignalrConnection.CreateFromUrl("https://localhost:5000"));

// Future:
// Treat 2 different data sources as a single IDatabase (shouldn't have overlapping table definitions)
IDatabase mergedDb = new MergedDb(sqlDb, sqliteDb);
```

Schemas can be fully managed from `IDatabase` instances. Here is how the SqlDatabase unit tests create temporary databases.

```cs
// TestFixture.cs
public static async Task WithTempDb(Func<IDatabase, Task> action, [CallerMemberName] string name = null)
{
    string dbName = "OoormSqlTests" + name + Guid.NewGuid().ToString().Substring(0, 8);
    var master = new SqlDatabase(SqlConnection.CreateShared(ConnectionString("master")));
    await master.DropDatabase(dbName);
    await master.CreateDatabase(dbName);
    try
    {
        var db = new SqlDatabase(SqlConnection.CreateShared(ConnectionString(dbName)));
        await action(db);
    }
    catch (Exception) { throw; }
    finally
    {
        await master.DropDatabase(dbName);
    }
}

// LocalHostTests.cs
[Fact]
public async Task CrudTest()
{
    await TestFixture.WithTempDb(async db =>
    {
        await db.CreateTables(typeof(Widget), typeof(WidgetDoodad), typeof(Doodad))
        // ...
    });
}

[Fact]
public async Task ItemExtensionsTest()
{
    await TestFixture.WithTempDb(async db =>
    {
        await typeof(Widget).CreateTableIn(db);
        await typeof(WidgetDoodad).CreateTableIn(db);
        // ...
    });
}    
```