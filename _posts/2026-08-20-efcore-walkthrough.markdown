---
layout: post
title: "Entity Framework Core: A Step-by-Step Walkthrough"
date: 2026-08-20 00:00:00 +0000
categories: [.NET]
tags:
  - dotnet
  - efcore
  - entity-framework
  - csharp
  - database
---


# Entity Framework Core: A Step-by-Step Walkthrough

Entity Framework Core (EF Core) is the default ORM for .NET applications, and for good reason -- it handles the tedious plumbing of database access while letting you work with C# objects instead of raw SQL. But getting started with EF Core involves several moving parts: project setup, NuGet packages, DbContext configuration, migrations, and the application layer that ties it all together.

This article walks through building a complete ASP.NET Core MVC application with EF Core, step by step. We will create a project from scratch, add EF Core with SQL Server, define a data model, run migrations, and implement full CRUD operations. By the end, you will have a clear mental model of how all the pieces fit together.

## Step 1: Create the Project

Start by creating a solution and an MVC project:

```bash
mkdir InstantScratchIts
cd InstantScratchIts

dotnet new sln --name InstantScratchIts
dotnet new mvc --name InstantScratchIts.Web --output InstantScratchIts.Web
dotnet sln add ./InstantScratchIts.Web/InstantScratchIts.Web.csproj
```

Verify everything compiles:

```bash
dotnet build
dotnet run -p ./InstantScratchIts.Web/InstantScratchIts.Web.csproj
```

Why a solution file? Even for a single project, solutions make it easier to add class libraries, test projects, and other components later. It is a good habit from day one.

## Step 2: Add Entity Framework Core

EF Core is modular -- you install only the database provider you need. We will use SQL Server, so we need:

```bash
dotnet add ./InstantScratchIts.Web/InstantScratchIts.Web.csproj package Microsoft.EntityFrameworkCore.SqlServer
dotnet add ./InstantScratchIts.Web/InstantScratchIts.Web.csproj package Microsoft.EntityFrameworkCore.Tools
```

The `SqlServer` package is the database provider. The `Tools` package enables the `dotnet ef` CLI commands for migrations and database updates. You can find the full [list of EF Core providers](https://docs.microsoft.com/en-us/ef/core/providers/) in the Microsoft documentation -- PostgreSQL, SQLite, MySQL, Cosmos DB, and more are all supported.

### Configure the Connection String

Add the connection string to `appsettings.json`:

```json
"ConnectionStrings": {
  "Database": "Server=(localdb)\\mssqllocaldb;Database=instant;Trusted_Connection=True;MultipleActiveResultSets=true"
}
```

### Register the DbContext

Create your `AppDbContext` class and register it in `Startup.ConfigureServices` (or `Program.cs` in minimal hosting):

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Database")));
```

The `AddDbContext<T>` extension method registers the context as a scoped service and configures it with the connection string. The `DbContextOptionsBuilder` gives you full control over EF Core's internals if you need it.

## Step 3: Define the Data Model

EF Core uses a code-first approach by default. You define your domain as plain C# classes (POCOs), and EF Core maps them to database tables.

For our example application -- a scratch ticket game manager -- we have games that are valid in one or more jurisdictions. Define your entity classes and register them as `DbSet<T>` properties on your `AppDbContext`.

The key principle here: your entities should be simple classes with properties. EF Core handles the rest through conventions and, when needed, explicit configuration via the Fluent API or data annotations.

## Step 4: Create and Apply Migrations

Migrations are EF Core's mechanism for keeping your database schema in sync with your code. Generate a migration:

```bash
dotnet ef migrations add InitialSchema --project ./InstantScratchIts.Web/InstantScratchIts.Web.csproj
```

Apply it to create (or update) the database:

```bash
dotnet ef database update --project ./InstantScratchIts.Web/InstantScratchIts.Web.csproj
```

This command does four things:

1. Builds your application
2. Loads the services configured in your app's startup, including the DbContext
3. Checks whether the database exists; creates it if not
4. Applies any unapplied migrations

After running this, you can verify the database and tables exist using SQL Server Management Studio or Azure Data Studio.

## Step 5: Create the Domain Service

The domain service encapsulates business logic and sits between your controllers and the DbContext. This separation keeps controllers thin and business rules testable:

```csharp
services.AddScoped<InstantGameService>();
```

For a simple domain, a single service class may suffice. In larger applications, you would have multiple services, each responsible for a bounded context.

## Step 6: Reading Data

With the infrastructure in place, let's build the read operations. Start with the controller:

```csharp
public class InstantGameController : Controller
{
    public InstantGameService _service;
    public InstantGameController(InstantGameService service)
    {
        _service = service;
    }

    public IActionResult Index()
    {
        var models = _service.GetInstantGames();
        return View(models);
    }
}
```

The service queries the database and projects results into view models. Note the use of `.Select()` to project directly in the query -- this generates efficient SQL that only retrieves the columns you need:

```csharp
public ICollection<InstantGamesSummaryViewModel> GetInstantGames()
{
    return _context.InstantGames
        .Where(r => !r.IsDeleted)
        .Select(x => new InstantGamesSummaryViewModel
        {
            Id = x.InstantGameId,
            GameNo = x.GameNo,
            Name = x.Name,
            TicketAmount = x.TicketAmount,
        })
        .ToList();
}
```

### Detail View with Related Data

For the detail view, include related entities (jurisdictions) using projection:

```csharp
public InstantGameDetailViewModel GetInstantGameDetail(int id)
{
    return _context.InstantGames
        .Where(x => x.InstantGameId == id)
        .Where(x => !x.IsDeleted)
        .Select(x => new InstantGameDetailViewModel
        {
            Id = x.InstantGameId,
            Name = x.Name,
            TicketAmount = x.TicketAmount,
            Jurisdictions = x.Jurisdiction
                .Select(region => new InstantGameDetailViewModel.Region
                {
                    Name = region.Name,
                    Allocation = region.Allocation
                })
        })
        .SingleOrDefault();
}
```

Using `.Select()` instead of `.Include()` for read-only scenarios is a best practice -- it avoids loading unnecessary data and gives you full control over the shape of the result.

## Step 7: Editing Data

The edit flow follows a two-action pattern common in MVC: a GET action to load the form and a POST action to save changes.

First, load the current values into an update command object:

```csharp
public IActionResult Edit(int id)
{
    var model = _service.GetInstantGameForUpdate(id);
    if (model == null)
        return NotFound();
    return View(model);
}
```

The command pattern is important here. The `UpdateInstantGameCommand` class encapsulates both the data and the update logic:

```csharp
public class UpdateInstantGameCommand : EditInstantGameBase
{
    public int Id { get; set; }

    public void UpdateInstantGame(InstantGame game)
    {
        game.Name = Name;
        game.GameNo = GameNo;
        game.TicketAmount = TicketAmount;
    }
}
```

The base class includes validation attributes:

```csharp
public class EditInstantGameBase
{
    [Required, StringLength(100)]
    public string Name { get; set; }
    [Range(0, 24), DisplayName("Name of the Game")]
    public int GameNo { get; set; }
    [Range(1000, 9999), DisplayName("Game Number")]
    public decimal TicketAmount { get; set; }
}
```

## Step 8: Saving Updates

The POST action validates the model state, calls the service, and either redirects on success or redisplays the form with errors:

```csharp
[HttpPost]
public IActionResult Edit(UpdateInstantGameCommand command)
{
    try
    {
        if (ModelState.IsValid)
        {
            _service.UpdateInstantGame(command);
            return RedirectToAction(nameof(View), new { id = command.Id });
        }
    }
    catch (Exception)
    {
        ModelState.AddModelError(
            string.Empty,
            "An error occured saving the instant game"
        );
    }
    return View(command);
}
```

The service method loads the entity, applies changes via the command, and saves:

```csharp
public void UpdateInstantGame(UpdateInstantGameCommand cmd)
{
    var game = _context.InstantGames.Find(cmd.Id);
    if (game == null) { throw new Exception("Unable to find the instant game"); }
    if (game.IsDeleted) { throw new Exception("Unable to update a deleted instant game"); }

    cmd.UpdateInstantGame(game);
    _context.SaveChanges();
}
```

This is EF Core's change tracking in action. When you modify properties on a tracked entity and call `SaveChanges()`, EF Core generates the appropriate `UPDATE` SQL statement automatically.

## Step 9: Deleting Data

The application uses soft deletes -- setting an `IsDeleted` flag rather than removing the row from the database. This preserves data for auditing and recovery:

```csharp
public void DeleteInstantGame(int id)
{
    var game = _context.InstantGames.Find(id);
    if (game.IsDeleted) { throw new Exception("Unable to delete a deleted game"); }

    game.IsDeleted = true;
    _context.SaveChanges();
}
```

Soft deletes pair well with the `.Where(x => !x.IsDeleted)` filter used in all read queries. For a more robust approach, consider using EF Core's global query filters to apply this automatically.

## Step 10: Creating New Records

The create flow is similar to edit but starts with an empty form and inserts a new entity:

```csharp
public int CreateInstantGame(CreateInstantGameCommand cmd)
{
    var game = cmd.ToInstantGame();
    _context.Add(game);
    _context.SaveChanges();
    return game.InstantGameId;
}
```

The `CreateInstantGameCommand` includes a factory method that converts the command into a domain entity, including related jurisdiction entities:

```csharp
public class CreateInstantGameCommand : EditInstantGameBase
{
    public IList<CreateJurisdictionCommand> Regions { get; set; }
        = new List<CreateJurisdictionCommand>();

    public InstantGame ToInstantGame()
    {
        return new InstantGame
        {
            Name = Name,
            GameNo = GameNo,
            TicketAmount = TicketAmount,
            Jurisdictions = Regions?.Select(x => x.ToJurisdiction()).ToList()
        };
    }
}
```

When you call `_context.Add(game)`, EF Core tracks both the parent entity and all related child entities. A single `SaveChanges()` call inserts everything in the correct order, respecting foreign key relationships.

## Key Takeaways

- **Start with a solution file** even for single-project applications
- **Use code-first migrations** to keep your database schema version-controlled alongside your code
- **Project with `.Select()`** instead of loading full entities for read-only scenarios
- **Separate concerns** with a service layer between controllers and the DbContext
- **Use the command pattern** to encapsulate validation and update logic
- **Prefer soft deletes** for data you might need to audit or recover
- **EF Core's change tracking** handles INSERT and UPDATE SQL generation automatically

## References

- [Entity Framework Core Documentation](https://learn.microsoft.com/en-us/ef/core/)
- [EF Core Database Providers](https://docs.microsoft.com/en-us/ef/core/providers/)
- [DbContext Lifetime and Configuration](https://learn.microsoft.com/en-us/ef/core/dbcontext-configuration/)
- [EF Core Migrations Overview](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/)
- [Global Query Filters](https://learn.microsoft.com/en-us/ef/core/querying/filters)
