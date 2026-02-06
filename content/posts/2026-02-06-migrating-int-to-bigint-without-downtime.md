---
title: "Migrating INT to BIGINT in SQL Server without (significant) downtime"
date: 2026-02-06T12:00:00+02:00
draft: true
categories: ["C#"]
tags: ["SQL Server", "Entity Framework", "EF Core", "Database Migration", "C#"]
---

Hello again! Long time no see. And by "long time" I mean over a decade. My last post was in August 2013. I'd love to say I was on some kind of epic sabbatical, but really I just got busy and forgot I had a blog. You know how it is.

Anyway, I'm back, and I figured if I was going to break a twelve-year silence, I'd better make it worth your while. So here's a story about database migrations, integer overflow, expression tree rewriting, and the kind of engineering that makes you question your life choices. If that doesn't get you excited, I don't know what will.

What do you do when your biggest database table is about to hit the 2.1 billion row limit of a SQL Server `INT` identity column? You migrate it to `BIGINT`, of course. Easy, right?

Well, no. Not when that table has hundreds of millions of rows and counting, not when you have dozens of separate databases with the same schema, and *definitely* not when you need to migrate them one by one without restarting your application servers. Oh, and the system needs to stay online during most of it.

This is the story of how we're pulling it off.

## The ticking time bomb

Our application stores image metadata in a SQL Server table called `Images`. Every single version of every single file that enters the system gets a row in this table. The primary key column `ID`, as is tradition, is an auto-incrementing `INT`. A dozen or so related tables reference this column as a foreign key (think `ImageData.ImageId`, `ImageTags.ImageId`, etc.).

For those who aren't intimately familiar with SQL Server data types (lucky you), `INT` maxes out at 2,147,483,647. That's 2.1 billion. Sounds like a lot, until you do the math on how fast your table is growing and realize you'll hit the ceiling in a few months.

We discovered this in December 2025. Our biggest customer's database would hit the limit by April 2026. Four months. Merry Christmas to us.

When that happens, SQL Server will simply refuse to insert new rows. No new images can be processed. The system freezes. Not great.

The fix is straightforward in theory: change the column from `INT` to `BIGINT`, which supports up to 9.2 quintillion values. That should last a while. (Famous last words, but seriously, 9.2 * 10^18 is a lot of images.)

## Why this is harder than it sounds

If you've ever tried to `ALTER TABLE ... ALTER COLUMN` on a table with hundreds of millions of rows, multiple foreign keys, indexed views, and a bunch of related tables... you know this isn't a five-minute operation. SQL Server essentially needs to rewrite the entire table.

But the real complexity comes from our architecture. We don't have one database, we have *many*. Each customer (or group of small customers) gets their own database with the same schema. We'd be migrating these databases one by one over a period of weeks or months.

This means our application needs to simultaneously handle:
- Pre-migration databases where `Images.ID` is still an `INT`
- Post-migration databases where `Images.ID` is now a `BIGINT`

And it needs to figure out which is which *at runtime*, without a restart. No deployment in between. The same running application, talking to databases with different schemas.

Let that sink in for a moment.

## Part 1: Teaching the application layer to be flexible

We have two backend codebases: a legacy one on .NET Framework using Entity Framework 6, and a modern one on .NET 9 using EF Core. As you might expect, the modern codebase was easier to adapt. Let's start there.

### The modern codebase: EF Core

Here's the pleasant surprise: EF Core is perfectly happy mapping a C# `long` property to either an `INT` or `BIGINT` database column. It just needs to know which one it's dealing with. So the trick is to have two DbContext subclasses:

```csharp
public sealed class IntRepoDbContext(DbContextOptions<IntRepoDbContext> options)
    : RepoDbContext(options)
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        modelBuilder.Entity<Image>().Property(i => i.Id).HasColumnType("int");
        modelBuilder.Entity<ImageData>().Property(i => i.ImageId).HasColumnType("int");
        // ... same for all foreign key columns referencing Images.ID
    }
}

public sealed class BigIntRepoDbContext(DbContextOptions<BigIntRepoDbContext> options)
    : RepoDbContext(options)
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        modelBuilder.Entity<Image>().Property(i => i.Id).HasColumnType("bigint");
        modelBuilder.Entity<ImageData>().Property(i => i.ImageId).HasColumnType("bigint");
        // ... same for all foreign key columns referencing Images.ID
    }
}
```

Both inherit from a shared `RepoDbContext` base class that contains all the common configuration. The only difference is the column type declarations. Clean, simple, no drama.

Then, a provider decides which one to use:

```csharp
internal sealed class RepoDbContextProvider : IRepoDbContextProvider
{
    private readonly IBigIntSchemaChecker _bigIntSchemaChecker;
    private readonly IDbContextFactory<BigIntRepoDbContext> _bigIntRepoDbContextFactory;
    private readonly IDbContextFactory<IntRepoDbContext> _intRepoDbContextFactory;

    public IRepoDbContext Provide()
    {
        return _bigIntSchemaChecker.RepoDbAlreadyUsesBigInt()
            ? _bigIntRepoDbContextFactory.CreateDbContext()
            : _intRepoDbContextFactory.CreateDbContext();
    }
}
```

That's it for EF Core. I told you it was the easy part.

### The legacy codebase: EF6

Now for the legacy codebase. Entity Framework 6 is... less accommodating. It flat out refuses to map a `long` property to an `INT` column (and vice versa). The CLR type and database type must match exactly.

So we need to get creative. The entity class now carries *two* backing properties, and a public-facing property that delegates to the correct one:

```csharp
public class Image
{
    // The property that all application code uses
    public long ID
    {
        get => UsesBigInt ? BigIntId : IntId;
        set
        {
            if (UsesBigInt)
            {
                BigIntId = value;
            }
            else
            {
                if (value > int.MaxValue)
                    throw new OverflowException(
                        $"Value {value} exceeds the maximum for Int32");
                IntId = (int)value;
            }
        }
    }

    public int IntId { get; set; }
    public long BigIntId { get; set; }
    internal bool UsesBigInt { get; set; } = true;
}
```

Application code still uses `Image.ID` as before. Under the hood, it routes to `IntId` or `BigIntId` depending on which database we're talking to. The `UsesBigInt` flag is set by the DbContext, not the entity itself.

Speaking of the DbContext, it has its own fun:

```csharp
protected RepoContext(DbConnection connection, /* ... */, bool usesBigInt)
    : base(connection, true)
{
    _usesBigInt = usesBigInt;

    // When entities are loaded from the database, set the UsesBigInt flag
    ((IObjectContextAdapter)this).ObjectContext.ObjectMaterialized +=
        (_, e) =>
        {
            if (e.Entity is Image image)
                image.UsesBigInt = _usesBigInt;
            // ... same for related entity types
        };
}
```

The `ObjectMaterialized` event fires whenever EF6 loads an entity from the database. We hook into it to stamp each entity with the correct flag. Similarly, `SaveChanges` is overridden to propagate the flag before saving.

But wait, there's more. (There's always more with EF6.)

The fluent mapping API also needs two configurations:

```csharp
public sealed class ImageIntMap : ImageMap
{
    public ImageIntMap()
    {
        HasKey(t => t.IntId);
        Property(t => t.IntId)
            .HasColumnName("ID")
            .HasColumnType("int")
            .HasDatabaseGeneratedOption(DatabaseGeneratedOption.Identity);
        Ignore(t => t.BigIntId);
        Ignore(t => t.ID);
    }
}

public sealed class ImageBigIntMap : ImageMap
{
    public ImageBigIntMap()
    {
        HasKey(t => t.BigIntId);
        Property(t => t.BigIntId)
            .HasColumnName("ID")
            .HasColumnType("bigint")
            .HasDatabaseGeneratedOption(DatabaseGeneratedOption.Identity);
        Ignore(t => t.IntId);
        Ignore(t => t.ID);
    }
}
```

Notice how the `ID` property (the public one) is `Ignore`d in both cases. EF6 only knows about `IntId` or `BigIntId`, never the abstraction on top.

And then there's the real brain-twister: LINQ queries. Application code writes things like:

```csharp
var image = context.Images.FirstOrDefault(i => i.ID == someId);
```

But EF6 doesn't know about `ID`. It only knows `IntId` or `BigIntId`. If we let EF6 try to translate `i.ID` to SQL, it will throw the classic *"LINQ to Entities does not recognize the method"* error.

The solution? Expression tree rewriting. We intercept LINQ queries at runtime and rewrite property accesses. The `BigIntRewritingExpressionVisitor` does the actual property swapping:

```csharp
public sealed class BigIntRewritingExpressionVisitor : ExpressionVisitor
{
    protected override Expression VisitMember(MemberExpression node)
    {
        // Image.ID → Image.BigIntId
        if (node.Member.DeclaringType == typeof(Image)
            && node.Member.Name == nameof(Image.ID))
        {
            return Expression.Property(
                node.Expression,
                PropertiesToVisit.ImageProperties.BigIntId);
        }

        // ... same for other entities with foreign keys to Images
        return base.VisitMember(node);
    }
}
```

And its counterpart for databases that still use `INT` adds explicit casts:

```csharp
public sealed class IntRewritingExpressionVisitor : ExpressionVisitor
{
    protected override Expression VisitMember(MemberExpression node)
    {
        // Image.ID → (long)Image.IntId
        if (node.Member.DeclaringType == typeof(Image)
            && node.Member.Name == nameof(Image.ID))
        {
            var intIdProperty = Expression.Property(
                node.Expression,
                PropertiesToVisit.ImageProperties.IntId);
            return Expression.Convert(intIdProperty, typeof(long));
        }

        // ... same for other entities
        return base.VisitMember(node);
    }
}
```

These visitors are chained together with a `CompositeExpressionVisitor` and injected into the query pipeline. Every `DbSet` property on the context is wrapped in a `BigIntRewritingDbSet<T>` that intercepts queries through a custom `IQueryProvider`:

```csharp
public BigIntRewritingDbSet<Image> Images
    => new BigIntRewritingDbSet<Image>(base.Set<Image>(), _visitor);
```

If you've ever written a custom `ExpressionVisitor` you know this is deep in "dark magic" territory. (I may have [written about dark magic before](/posts/2013-03-09-replacing-method-calls-linqkit/).) But it works, and application code remains blissfully unaware.

#### Fighting EF6's reflection habits

Here's something that'll make your eye twitch: EF6 uses *reflection* to discover methods on your queryable types. It doesn't use an interface. It just looks for a public method called `Include` with the right signature. If you're wrapping `DbSet<T>` in a custom type (like we are), you need to expose these methods yourself, or EF6 will silently fall back to behavior you don't expect.

Our `BigIntRewritingDbSet<T>` had to grow a bunch of methods that look suspiciously like `DbSet<T>` methods:

```csharp
public sealed class BigIntRewritingDbSet<T> : IDbSet<T>, IOrderedQueryable<T>
    where T : class
{
    private readonly DbSet<T> _innerSet;
    private readonly ExpressionVisitor _visitor;

    // EF6 discovers this via reflection. No interface. Just vibes.
    public BigIntRewritingQueryable<T> Include(string path)
    {
        return new BigIntRewritingQueryable<T>(
            _innerSet.Include(path), _visitor);
    }

    // Same story - discovered via reflection by EF Migrations
    [SuppressMessage("ReSharper", "UnusedMember.Global",
        Justification = "Used by EF Migrations internally via reflection")]
    public void AddOrUpdate(params T[] entities)
    {
        _innerSet.AddOrUpdate(entities);
    }

    // The escape hatch: expose the inner DbSet for nested queries
    public DbSet<T> DbSet => _innerSet;

    // ... all the standard CRUD methods delegating to _innerSet ...
}
```

Note the `DbSet` property at the end. That's our escape hatch, and it deserves its own section.

#### The nested query problem

Not everything could be rewritten automatically. The expression rewriting works great for top-level queries:

```csharp
// This works: the query provider intercepts and rewrites
context.Images.Where(i => i.ID == someId);
```

But when you use a `DbSet` property *inside* a nested expression -- say, a subquery -- EF6 encounters our `BigIntRewritingDbSet<T>` wrapper as a constant in the expression tree and throws a "cannot compute constant" exception. It doesn't know what to do with it.

To handle this automatically where possible, we wrote *another* expression visitor -- `DbSetAccessRewritingExpressionVisitor` -- that runs before the property rewriting. It detects accesses to `BigIntRewritingDbSet<T>` properties inside expressions and unwraps them to the underlying `DbSet`:

```csharp
public sealed class DbSetAccessRewritingExpressionVisitor : ExpressionVisitor
{
    protected override Expression VisitMember(MemberExpression node)
    {
        if (node.Member is PropertyInfo propertyInfo
            && propertyInfo.PropertyType.IsGenericType
            && propertyInfo.PropertyType.GetGenericTypeDefinition()
                == typeof(BigIntRewritingDbSet<>)
            && typeof(IRepoContext).IsAssignableFrom(propertyInfo.DeclaringType))
        {
            // repoContext.SomeDbSet → repoContext.SomeDbSet.DbSet
            var originalExpression = Visit(node.Expression);
            var repoContextPropertyAccess =
                Expression.Property(originalExpression, propertyInfo);
            var dbSetProperty = propertyInfo.PropertyType
                .GetProperty(nameof(BigIntRewritingDbSet<object>.DbSet));
            if (dbSetProperty != null)
            {
                return Expression.Property(
                    repoContextPropertyAccess, dbSetProperty);
            }
        }
        return base.VisitMember(node);
    }
}
```

The two visitors are chained: first unwrap, then rewrite properties. Order matters.

But even this isn't enough for every case. Some nested queries still trip up EF6, and for those we had to go through the codebase and manually update the queries to use the `.DbSet` property directly:

```csharp
// Before (crashes with "cannot compute constant"):
var results = repo.UserRoles
    .Where(r => r.RoleName == roleName)
    .Where(r => !repo.UserRoles.Any(
        other => other.RoleName != otherRole && other.UserId == r.UserId
    ));

// After (explicit .DbSet to bypass the wrapper):
var results = repo.UserRoles
    .Where(r => r.RoleName == roleName)
    .Where(r => !repo.UserRoles.DbSet.Any(
        other => other.RoleName != otherRole && other.UserId == r.UserId
    ));
```

And some queries were so deeply nested that even `.DbSet` wasn't enough. A `SelectMany` that references another DbSet inside a lambda? EF6 simply can't handle that with our wrapper. Those had to be split into two separate queries entirely:

```csharp
// Before (no amount of expression rewriting can save this):
return repo.ImageTags
    .Where(x => x.ImageId == imageId)
    .SelectMany(tag =>
        repo.Tags.Where(t => tag.TagId == t.ID))
    .ToList();

// After (two queries, materialized in between):
var tagIds = repo.ImageTags
    .Where(x => x.ImageId == imageId)
    .Select(x => x.TagId)
    .ToList();

return repo.Tags
    .Where(t => tagIds.Contains(t.ID))
    .ToList();
```

Finding all these cases was... not fun. Our first attempt at the expression rewriting was actually reverted entirely after we realized the scope of the problem. The second attempt -- with the two-layer visitor chain -- stuck, but still required a codebase-wide hunt for queries that needed manual attention. (92 commits across the legacy codebase, if you're counting.)

## Part 2: Runtime schema detection

So we have two code paths. How does the application know which one to use for a given database?

It asks the database:

```csharp
public bool AlreadyUsesBigIntIds()
{
    using var connection = CreateSqlConnection();
    connection.Open();
    var command = connection.CreateCommand();
    command.CommandText = @"
        SELECT DATA_TYPE
        FROM INFORMATION_SCHEMA.COLUMNS
        WHERE TABLE_NAME = 'Images' AND COLUMN_NAME = 'ID'";
    var result = command.ExecuteScalar()?.ToString();
    return string.Equals(result, "bigint", StringComparison.OrdinalIgnoreCase);
}
```

Simple enough. But you're probably thinking: "are we going to run this query *every time* we create a DbContext?" And the answer is: no, of course not. We cache it.

The schema check result is cached in memory for 20 minutes per database. But here's where it gets interesting: during an active migration, we need the cache to be disabled for the database being migrated. Otherwise the application might keep using the `IntRepoDbContext` for up to 20 minutes after the migration completes.

We solved this with a system setting called `DisableBigIntMigrationSchemaCachingForRepoDbNames`. (Yes, that name is deliberately verbose. It's designed to make its temporary nature obvious.) This setting contains a comma-separated list of database names where caching should be bypassed.

```csharp
public bool RepoDbAlreadyUsesBigInt()
{
    var repoDbNamesWhereSchemaCachingIsDisabled =
        _mainDbApi.SystemSettings
            .DisableBigIntMigrationSchemaCachingForRepoDbNames ?? [];

    if (repoDbNamesWhereSchemaCachingIsDisabled.Any(_repoDb.MatchesRepoDbName))
    {
        return _repoDb.AlreadyUsesBigIntIds();  // No caching, query every time
    }

    return _memoryCache.GetOrCreate(
        $"RepoDbUsesBigInt_{_repoDb.Name.ToLowerInvariant()}",
        _ => _repoDb.AlreadyUsesBigIntIds(),
        new MemoryCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(20)
        }
    );
}
```

The system setting itself is also cached (for 10 minutes), because, well, it's also in a database and we don't want to overwhelm it with queries either.

## Part 3: The SQL migration itself

Now for the fun part: actually migrating hundreds of millions of rows from `INT` to `BIGINT` in a production database. We split this into six phases.

### Phase 1: Setup

We can't just `ALTER COLUMN` in place. Instead, we create a parallel set of tables with the new schema:

```sql
CREATE TABLE dbo.Images_New (
    ID BIGINT IDENTITY(1,1) NOT NULL,
    -- ... all the other columns ...
    CONSTRAINT PK_Images_New PRIMARY KEY CLUSTERED (ID)
);
```

Then we create synchronization triggers on the *old* tables that mirror every INSERT, UPDATE, and DELETE to the new tables:

```sql
CREATE TRIGGER trg_Images_Migrate_Insert ON dbo.Images
AFTER INSERT
AS
BEGIN
    SET NOCOUNT ON;
    SET IDENTITY_INSERT dbo.Images_New ON;
    INSERT INTO dbo.Images_New (ID, /* ... */)
    SELECT ID, /* ... */ FROM inserted;
    SET IDENTITY_INSERT dbo.Images_New OFF;
END
```

From this point on, any new data flowing into the old table is automatically copied to the new one. The system is still fully online.

### Phase 2: Backfill

Now we need to copy all the *existing* data. We do this in batches, looping from the highest ID down to the lowest:

```sql
DECLARE @BatchSize INT = 20000;
DECLARE @CurrentID INT = @MaxID;

WHILE @CurrentID >= @MinID
BEGIN
    INSERT INTO dbo.Images_New (ID, /* ... */)
    SELECT ID, /* ... */
    FROM dbo.Images
    WHERE ID > @CurrentID - @BatchSize
      AND ID <= @CurrentID;

    SET @CurrentID = @CurrentID - @BatchSize;
    WAITFOR DELAY '00:00:00.050';  -- Throttle to protect I/O
END
```

Why loop from high to low? Because new rows are being inserted at the *top* (highest IDs) via the triggers. If we went low to high, we'd be chasing a moving target. Going high to low means we process a fixed range while the triggers handle everything above.

To give you a sense of scale, here are the timings from one of our staging environment migrations (486 million image records):

| Step | Duration |
|---|---|
| Backfill (data copy) | 5h 30m |
| Index creation | 3h |
| **Total backfill phase** | **8h 30m** |

All of this while the system remained fully operational. Eight and a half hours of background work with zero user impact.

And our biggest database? 5.1 times larger than this one. The timings scale roughly linearly, putting us at an estimated 44 hours of backfill. That's nearly two full days of copying data in the background. Good thing the system stays online during all of it.

We haven't run the production migrations yet at the time of writing -- those are coming soon. But I'll sleep a little easier knowing the approach has been validated end-to-end on realistic data.

### Phase 3: Validation

Before we proceed, we validate that the new tables contain the same data as the old ones. Trust but verify.

### Phase 4: The cutover

This is the only part that requires downtime. Fortunately, it's brief.

```sql
BEGIN TRANSACTION;
    -- Lock everything
    DECLARE @lock INT = (
        SELECT TOP 1 ID FROM dbo.Images WITH (TABLOCKX, HOLDLOCK)
    );

    -- Drop the sync triggers
    DROP TRIGGER trg_Images_Migrate_Insert;
    DROP TRIGGER trg_Images_Migrate_Update;
    DROP TRIGGER trg_Images_Migrate_Delete;

    -- Drop views and foreign keys
    DROP VIEW [dbo].[vwImageCountIndexed];
    DROP VIEW [dbo].[vwImageSummary];

    -- The swap
    EXEC sp_rename 'dbo.Images', 'Images_Old';
    EXEC sp_rename 'dbo.Images_New', 'Images';

    -- Reseed the identity
    DECLARE @NewMaxID BIGINT = (SELECT MAX(ID) FROM dbo.Images);
    DBCC CHECKIDENT ('dbo.Images', RESEED, @NewMaxID);

    -- Recreate foreign keys and views
    -- ...
COMMIT TRANSACTION;
```

Here's the cutover breakdown from the same 486 million row staging database:

| Step | Duration | Impact |
|---|---|---|
| Table swap (the actual rename) | ~1s | Image processing paused |
| Indexed view recreation | ~22s | Some queries unavailable |
| Update related tables | ~27s | Other queries unavailable |
| **Total cutover** | **~50s** | |

Under a minute. For our biggest database (5.1x larger), we estimate roughly 4.5 minutes of cutover downtime. Not bad for a table with billions of potential rows.

### Phase 5: Cleanup

Drop the old tables (`Images_Old`, etc.). The migration is complete.

### Phase 6: Validation

A comprehensive check that verifies everything is in order: correct data types, primary keys, foreign keys, indexes, no leftover temporary tables, no orphaned triggers.

## The operational playbook

From the perspective of someone actually running the migration, the steps are:

1. Add the database name to `DisableBigIntMigrationSchemaCachingForRepoDbNames`
2. Wait 30 minutes (worst case for existing caches to expire)
3. Run Phase 1 through 4
4. Remove the database name from `DisableBigIntMigrationSchemaCachingForRepoDbNames`

After step 3, the application automatically detects that the database now uses `BIGINT` (because caching is disabled for this database) and switches to the appropriate DbContext. No restart needed.

## Things to consider

Like any complex migration, there are some things that kept me up at night:

- **The expression rewriting in EF6 is genuinely dark magic.** It works, it's tested, but if a future developer stumbles into it without context, they're going to have questions. We've marked all of it with `// BEGIN Temporary bigint workaround` and `// END` comments so it's clearly identifiable as temporary code.

- **The triggers add overhead during the backfill.** Every write to the old table now also writes to the new table. We monitored this carefully and it was acceptable, but it's worth being aware of.

- **The 20-minute cache means there's a window after migration where some requests might still use the old code path.** Since the old DbContext would try to talk to a table that no longer exists (it was renamed to `_Old`), we disable caching for the database being migrated to avoid this.

- **This is all temporary.** Once every database is migrated, we can rip out the dual code paths, the schema checking, the caching, all of it. The verbose naming convention (`DisableBigIntMigrationSchemaCachingForRepoDbNames`) is a deliberate reminder that this code has an expiration date.

## Wrapping up

Is this over-engineered? Maybe. Is it the kind of thing that makes you question your career choices at 2 AM? Absolutely. But it works, it's been tested extensively on staging environments, and it will let us migrate dozens of production databases without taking the system offline or restarting anything.

If you ever find yourself staring at an `INT` identity column approaching its limit, I hope this gives you some ideas. And if you're one of those people who always uses `BIGINT` from the start... well, you're smarter than us. But where's the adventure in that?

Happy coding!
