---
layout: post
title: Windows Universal App Series – Implementing the CQRS Pattern with a Segregated Data Model to Persist & Sync Offline Data to Azure DB via Azure Mobile Services
redirect_from: "/blog/2015/03/windows-universal-app-series-implementing-the-cqrs-pattern-with-a-segregated-data-model-to-persist-sync-offline-data-to-azure-db-via-azure-mobile-services/"
tags: windows-universal-apps
---

### Introduction
In the last post in my Windows Universal App Series, I talked about building a Line of Business app for an amazing client of ours that is used for on-site inventory management with their respective client(s). One of the major challenges that our client faced was that when they went on-site, it’s typically in large buildings whose structures aren’t conducive to 3G/4G signal, and therefore wanted an application that would allow their reps to download data to their devices before beginning their work – while they are outside and had signal. They wanted the device to queue all changes and synchronize at the rep’s request once finishing their service call.

<!--more-->

The client’s legacy application was on the Windows Mobile 6.5 OS, which still supported technologies like database replication directly from SQL Server. The catch is that DB replication to SQL CE died after 2008 R2, which means implementing such a feature on the existing replication framework would make our software outdated right out of the gate – let alone all of the limitations of forcing reps to log directly into a SQL Server from their phones to download their data, and the work it takes to keep those replications up-to-date with the active workforce.

This is a really interesting use case for offline synchronization capabilities, and we thought it would be a great time to try out Azure and their Mobile Services – specifically the [MobileServiceClient](https://github.com/Azure/azure-mobile-services/blob/master/sdk/Managed/src/Microsoft.WindowsAzure.MobileServices/MobileServiceClient.cs) – which acts as a WebAPI-consumption client that connects with a Mobile Services instance in Azure. Furthermore, the client simplifies downloading data from WebAPI controller endpoints, as well as storing in SQLite and synchronizing data back up to the Mobile Services instance to be stored in an Azure DB instance. In essence, Azure’s offline data sync capabilities marks an even more robust recreation of SQL Server 2008 R2’s database replication feature.

Previous posts in this series:

[Windows Universal App Series – Creating a Windows Universal App with Prism 5.0 & SQLite for Offline Persistence]({{ site.baseurl }}/blog/2015/03/windows-universal-app-series-creating-a-windows-universal-app-with-prism-50-sqlite-for-offline-persistence/)

### Setting up the DB Context to Build Out your Data Schema
As a prerequisite, please complete the [Get started with Mobile Services](http://azure.microsoft.com/en-us/documentation/articles/mobile-services-dotnet-backend-windows-store-dotnet-get-started/) tutorial.

So now that you’re set up with a new environment, let’s implement everything else directly in your Universal App. We’re going EF Code-First, so no need to make any database schema updates directly through SSMS (though you may prefer to use SSMS to load some test data).

The first thing we need to do is to assign the dbo schema to your newly created user in your Azure DB instance. This is really nice for multiple environments, because you don’t have to change your SSIS (or alternative) sync packages for multiple schemas. Go into SSMS (I know, I just told you that you didn’t need it…) and run the following against your Azure DB using your main account:

```
USE [AzureDbInstance]

ALTER USER [nLlrTgVqEULogin_AzureDbInstanceUser] WITH DEFAULT_SCHEMA = dbo;

GRANT INSERT ON SCHEMA :: dbo TO [nLlrTgVqEULogin_AzureDbInstanceUser]
GRANT SELECT ON SCHEMA :: dbo TO [nLlrTgVqEULogin_AzureDbInstanceUser]
GRANT ALTER ON SCHEMA :: dbo TO [nLlrTgVqEULogin_AzureDbInstanceUser]
GRANT DELETE ON SCHEMA :: dbo TO [nLlrTgVqEULogin_AzureDbInstanceUser]
GRANT UPDATE ON SCHEMA :: dbo TO [nLlrTgVqEULogin_AzureDbInstanceUser]
GRANT CONTROL ON SCHEMA :: dbo TO [nLlrTgVqEULogin_AzureDbInstanceUser]
GRANT EXECUTE ON SCHEMA :: dbo TO [nLlrTgVqEULogin_AzureDbInstanceUser]
```

So now you’re all set with dbo. It’s time to jump into your mobile service project and build out the data model and context. We can set this up now, and we’ll discuss CQRS afterward and why we’re using it. Let’s start with an Entity Framework Code-First DbContext:

```
public class MobileServiceContext : DbContext
{
	// keep this as-is - it uses the connection string set on Azure
	private const string ConnectionStringName = "Name=MS_TableConnectionString";
	public string Schema { get; private set; }
	
	public MobileServiceContext()
		: base(ConnectionStringName)
	{
		// note that we're setting this, but we're not explicitly telling it to use the mobile service
		// name, which is what it tries to do by default. if we leave this as-is, it will actually create the
		// tables in the default dbo schema, which is great if you want to launch multiple instances of 
		// this service (think preprod/prod)
		this.Schema = ServiceSettingsDictionary.GetSchemaName();
	}

	// define your read-only tables here. we're not going to use any special data annotations in your classes,
	// but instead will use some Type nomenclature and Fluent API to define relationships. these are read-only
	// by CQRS pattern definition
	public DbSet<VehicleManufacturer> VehicleManufacturers { get; set; } // notice the pluralization
	public DbSet<VehicleModel> VehicleModels { get; set; }
	
	// let's say we have an application where we want to be able to update the number of accolades that a vehicle
	// model has received. under the CQRS model, we would create a set of segregated tables here that are meant for writing
	// data from the application. we will only write to these tables via a command interface
	public DbSet<VehicleModel> VehicleModelUpdates { get; set; }

	protected override void OnModelCreating(DbModelBuilder modelBuilder)
	{
		// we remove this convention since we're doing it ourselves
		modelBuilder.Conventions.Remove<PluralizingTableNameConvention>();
		
		modelBuilder.Conventions.Add(
			new AttributeToColumnAnnotationConvention<TableColumnAttribute, string>(
				"ServiceTableColumn", (property, attributes) => attributes.Single().ColumnType.ToString()));

		// Fluent API
		this.SetupVehicleManufacturer(modelBuilder);
		this.SetupVehicleModel(modelBuilder);
		this.SetupVehicleModelUpdate(modelBuilder);
		base.OnModelCreating(modelBuilder);
	}

	// Fluent API Mappings
	private void SetupVehicleManufacturer(DbModelBuilder modelBuilder)
	{
		// properties
		modelBuilder.Entity<VehicleManufacturer>()
			.ToTable("VehicleManufacturer");

		modelBuilder.Entity<VehicleManufacturer>()
			.Property(x => x.Name)
			.HasColumnType("nvarchar")
			.HasMaxLength(25);

		modelBuilder.Entity<VehicleManufacturer>()
			.Property(x => x.Country)
			.HasColumnType("nvarchar")
			.HasMaxLength(100);

		// here we need to create the relationship between VehicleManufacturer and VehicleModel
		// entities, where we explicitly tell EF how to map them
		modelBuilder.Entity<VehicleManufacturer>()
			.HasMany(x => x.VehicleModels)
			.WithRequired(x => x.VehicleManufacturer)
			.HasForeignKey(x => x.VehicleManufacturerId)
			.WillCascadeOnDelete(false);
	}

	private void SetupVehicleModel(DbModelBuilder modelBuilder)
	{
		// properties
		modelBuilder.Entity<VehicleModel>()
			.ToTable("VehicleModel");

		modelBuilder.Entity<VehicleModel>()
			.Property(x => x.Name)
			.HasColumnType("nvarchar")
			.HasMaxLength(50);

		modelBuilder.Entity<VehicleModel>()
			.Property(x => x.Year)
			.HasColumnType("int");
			
		modelBuilder.Entity<VehicleModel>()
			.Property(x => x.AccoladeCount)
			.HasColumnType("int");	

		// we only need to define the relationship on one side, so no
		// need for an additional mapping here
	}
	
	private void SetupVehicleModelUpdates(DbModelBuilder modelBuilder)
	{
		// properties
		modelBuilder.Entity<VehicleModelUpdate>()
			.ToTable("VehicleModelUpdate");
	}
}
```

Along with this context, we’ll of course need to create a few classes to define our data model (as the definitions and relationships we create will automatically be written as a schema to your Azure DB). Here are the definitions of the classes used above:

```
public class VehicleManufacturer 
	: EntityData // this is a required mobile service class that adds additional properties for use with sync
{
	public string Name { get; set; }
	
	public string Country { get; set; }
	
	public virtual ICollection<VehicleModel> VehicleModels { get; set; }
}

public class VehicleModel : EntityData
{
	public string Name { get; set; }
	
	public int Year { get; set; }
	
	public int AccoladeCount { get; set; }
	
	public string VehicleManufacturerId { get; set; }
	
	public virtual VehicleManufacturer VehicleManufacturer { get; set; }
}

public class VehicleModelUpdate : EntityData
{
	public string VehicleModelId { get; set; }
	
	public virtual VehicleModel VehicleModel { get; set; }
	
	public int AccoladeCount { get; set; }
}
```

So now that we have these all created, let’s talk a bit about Azure and its Push/Pull capabilities. Currently in the client, we are able to Pull (download) data by table from the Mobile Service by passing a query, which is great because it means we can get only what we want, and not have to worry as much about data usage for large payloads (which honestly are JSON strings anyway, so you shouldn’t have much to worry about). The Push (upload/sync) functionality is a bit different, however. When you Push to the server, it’s an all-or-nothing proposition, as it’s called on the entire sync context. This makes sense because under the covers, the client is actually sending up WebAPI requests that play nicely with your foreign keys – it’d be difficult to do this otherwise.

The issue we were finding is that there is no "undo" functionality on data updates, so once you dirty a table on the local DB, it’s going to synchronize next time you call PushAsync on the sync context. To get around this limitation, we found that the CQRS (Command Query Responsibility Segregation) pattern fit quite nicely, because it’s typically implemented in its purest form with segregated Command- and Query-based data models (in this case, we’re combining them in the same database), which sometimes even have different schemas (to better serve their purposes). The idea behind using CQRS here is that if we write all changes to update-specific tables, we have created the “undo” functionality by deleting those records prior to Pushing the data to Azure. It does require a synchronization mechanism on the server side to keep the read/write tables in sync, but you probably needed something like that anyway for your purposes if you’re using offline sync.

### Completing the Mobile Service to Enable Offline Data Sync
Now that you have a schema ready to go, it’s worth noting that the Mobile Service project is really just a fancy WebAPI wrapper. The key is that you’re going to need a controller for each of your entities that you’re shipping back and forth to your client devices, because it downloads the data through API endpoints as JSON strings.

My favorite implementation of these controllers is to create a base class that implements all the tough stuff, and then you can explicitly make any changes you need for your controllers if you’d like. Here’s my base controller:

```
public class BaseTableController<T> : TableController<T> where T : EntityData
{
	protected override void Initialize(HttpControllerContext controllerContext)
	{
		base.Initialize(controllerContext);
		MobileServiceContext context = new MobileServiceContext();
		this.DomainManager = new EntityDomainManager<T>(context, base.Request, base.Services);
	}

	// GET tables/T
	public virtual IQueryable<T> GetAllT()
	{
		return base.Query();
	}

	// GET tables/T/48D68C86-6EA6-4C25-AA33-223FC9A27959
	public virtual SingleResult<T> GetT(string id)
	{
		return base.Lookup(id);
	}

	// PATCH tables/T/48D68C86-6EA6-4C25-AA33-223FC9A27959
	public virtual Task<T> PatchT(string id, Delta<T> patch)
	{
		return base.UpdateAsync(id, patch);
	}

	// POST tables/T
	public virtual async Task<IHttpActionResult> PostT(T item)
	{
		T current = await base.InsertAsync(item);
		return base.CreatedAtRoute("Tables", new {id = current.Id}, current);
	}

	// DELETE tables/T/48D68C86-6EA6-4C25-AA33-223FC9A27959
	public virtual Task DeleteT(string id)
	{
		return base.DeleteAsync(id);
	}
}
```

And quickly, the `EntityDomainManager` noted above:

```
public class EntityDomainManager<TDto, TModel> : MappedEntityDomainManager<TDto, TModel> where TDto : Entity where TModel : Entity
{
	private readonly MobileServiceContext _context;

	public EntityDomainManager(MobileServiceContext context, HttpRequestMessage request, ApiServices services) : base(context, request, services)
	{
		this._context = context;
	}

	public EntityDomainManager(MobileServiceContext context, HttpRequestMessage request, ApiServices services, bool enableSoftDelete)
		: base(context, request, services, enableSoftDelete)
	{
		this._context = context;
	}

	public override async Task<bool> DeleteAsync(string id)
	{
		return await this.DeleteItemAsync(id);
	}

	public override SingleResult<TDto> Lookup(string id)
	{
		return this.LookupEntity(s => s.Id == id);
	}

	public override async Task<TDto> UpdateAsync(string id, Delta<TDto> patch)
	{
		return await this.UpdateEntityAsync(patch, id);
	}

	protected override void SetOriginalVersion(TModel model, byte[] version)
	{
		this._context.Entry(model).OriginalValues["Version"] = version;
	}
}
```

The cool part of having this as your base class is that now all of your other controllers are as easy as this to set up:

```
public class VehicleManufacturerController : BaseTableController<VehicleManufacturer>
{
}
```

This controller will automatically implement (by way of inheritance) the CRUD operations that you need to run your application. Create controllers for the other two entities that we created above, and you’ll be all set. OK - almost there!

Now we need to bootstrap WebAPI via its Register method in `WebApiConfig`. Here’s mine:

```
public static void Register()
{
	// Use this class to set WebAPI configuration options
	HttpConfiguration config = ServiceConfig.Initialize(new ConfigBuilder(options));

	// To display errors in the browser during development, uncomment the following
	// line. Comment it out again when you deploy your service for production use.
	config.IncludeErrorDetailPolicy = IncludeErrorDetailPolicy.Always; // TODO: remove this for prod

	// this sets up configuration of your database, including your namespace and automatic migration settings
	DbMigrator migrator = new DbMigrator(new Configuration());
	migrator.Update();
}
```

And then of course we need to configure EF to do what we want it to do – in this case, let's go with Automatic Migrations to keep things simple. This means that publishing the web project here to our Mobile Service instance will automatically build out your DB schema, which is what we want. Here’s the Configuration class:

```
internal sealed class Configuration : DbMigrationsConfiguration<MobileServiceContext>
{
	public Configuration()
	{
		base.AutomaticMigrationsEnabled = true;
		base.AutomaticMigrationDataLossAllowed = true;
		base.ContextKey = "Rightpoint.Server.AzureMobileService.Models.MobileServiceContext"; // path to MobileServiceContext
		base.ContextType = typeof(MobileServiceContext);
	}
}
```

This concludes the steps to set up your web project. Go ahead and publish this to your Mobile Service instance (right click > Publish) and you now have the server side complete.

### Setting up the MobileServiceClient to Access your Azure DB Data
The client portion of this solution is a bit different, because it abstracts most of the HTTP part to Azure’s client that you get to use out-of-the-box. You just need to be concerned with the APIs that they provide as part of that client. Go ahead and create a class that inherits from Azure’s `MobileServiceClient` that looks something like this:

```
public class MobileServiceDatabaseContext : MobileServiceClient
{
	static readonly Uri Uri = new Uri("https://mobile-service-name.azure-mobile.net/", UriKind.Absolute);
	const string Passkey = "PASSKEYPASSKEYPASSKEYPASSKEY";
	
	public MobileServiceDatabaseContext() : base(Uri, Passkey)
	{
		base.SerializerSettings = new MobileServiceJsonSerializerSettings()
		{
			CamelCasePropertyNames = true
		};

		// setup SQLite storage
		MobileServiceSQLiteStore syncSqlLite = new MobileServiceSQLiteStore("sync.db");

		// here we define all tables that we want to use as read-only. these classes will be more or less exact
		// matches to what is currently defined in your Mobile Service project. the big difference is that they can't
		// inherit from Azure's EntityData base class, but you don't need most of those properties anyway
		syncSqlLite.DefineTable<VehicleManufacturer>();
		syncSqlLite.DefineTable<VehicleModel>();
		
		// read/write - same reason as in the Mobile Service, as this creates the write-based models that are meant
		// for the command channel of our CQRS implementation
		syncSqlLite.DefineTable<VehicleModelUpdate>();

		// Sync Handler
		SyncHandler syncHandler = new SyncHandler();
		base.SyncContext.InitializeAsync(syncSqlLite, syncHandler).Wait();
	}
}
```

**IMPORTANT: It’s worth noting here that you have decisions to make on whether you want data to be syncable between your server and client.  ANY table you include in this context will attempt to sync back up to Azure – if you don’t want that, you WILL need a separate data context for just pure SQLite storage (see my previous post on how to set that up).  These contexts are mutually exclusive – everything in pure SQLite will not sync to Azure, and anything within the MobileServiceClient context will sync.**

The model is a bit simpler, so we can create entities that inherit from a different base class

```
public class Entity
{
	public string Id { get; set; }
}
```

Also, note that you can use `IEnumerable<T>` on these ones (as opposed to `ICollection<T>` for EF) since we’re separated from EF now.

When we want to sync the data that we have collected, there needs to be some sort of conflict resolution because you have many different users potentially taking the same actions at the same time. For this reason, we include a SyncHandler which executes our logic of choice.

```
public class SyncHandler : IMobileServiceSyncHandler
{
	// this is a simplified conflict resolution method, mostly due to the CQRS pattern and the sync
	// rules which we have chosen to follow. this method will always take the latest data if there is
	// a conflict, so "last in" wins. you could write it differently if you have more complicated logic
	public async Task<JObject> ExecuteTableOperationAsync(IMobileServiceTableOperation operation)
	{
		MobileServiceInvalidOperationException error = null;

		do
		{
			error = null;

			try
			{
				JObject result = await operation.ExecuteAsync();
				return result;
			}
			catch (MobileServiceConflictException ex)
			{
				error = ex;
			}
			catch (MobileServicePreconditionFailedException ex)
			{
				error = ex;
			}
			catch (Exception e)
			{
				Debug.WriteLine(e.ToString());
				throw e;
			}
		} while (error == null);

		return null;
	}

	public Task OnPushCompleteAsync(MobileServicePushCompletionResult result)
	{
		return Task.FromResult(0);
	}
}
```

And finally, I love the idea of having a read-only version of the DB context that we can inject via Unity, because it locks our development team into following the CQRS pattern where everything is read-only except for our Command interface (which we’ll talk about later on). Here’s an easy wrapper around your context:

```
public interface IReadOnlyMobileServiceDatabaseContext
{
	Task<IEnumerable<T>> GetSyncTableReadOnly<T>();
}
```

So without knowing it, you have now implemented the Query channel, which should be directly accessible by your Application layer, if not your client directly. Reading data in your local database is now as easy as calling your implementation of `GetSyncTableReadOnly<T>(…)`, where it is returning an awaited `base.GetSyncTable<T>().ToEnumerableAsync()`.

### Creating the Command Channel to Write to your Local Sync Tables
What we haven’t talked about yet is the download process, but that’s because your client does not have access to write to those tables since it can only use the injected read-only context. We are going to create two types of handlers for your Commands that you will execute for writeable changes:

1. A `CommandHandler` that has read-only access to your local sync tables. These commands are very specific to an operation, and the `ICommand` entities that you send with them contain all required information to complete the operation. Consider this a separate model for Command-only
2. A `TypedCommandHandler<T>`, which is more of a generic handler for operations meant for any table you would like. This means you can use it generically to sync down any table of your choosing, including passing a query to pull partial data sets

First, let’s create the `TypedCommandHandler` to help you sync down your data from the Mobile Service instance. Here are the interfaces we’ll need to implement:

```
public interface ITypedCommand<T>
{
}

public interface ITypedCommandHandler<in TCommand, T> where TCommand : ITypedCommand<T>
{
	Task<object> Handle(TCommand command);
}
```

So basically, our application creates `ITypedCommand<T>` instances and executes them, which are handled by an `ITypedCommandHandler<TCommand, T>`. Here’s how I put together a Command that I called `SyncDownTableCommand<T>`:

```
public class SyncDownTableCommand<T> : TypedCommand<T>
{
	public string QueryId { get; set; }

	public Expression<Func<T, bool>> Query { get; set; }

	public SyncDownTableCommand(string queryId, Expression<Func<T, bool>> query)
	{
		if (query == null) throw new ArgumentNullException("query");

		this.QueryId = queryId;
		this.Query = query;
	}
}

public class TypedCommand<T> : ITypedCommand<T>
{
	public string Id { get; private set; }

	public TypedCommand()
	{
		this.Id = Guid.NewGuid().ToString();
	}
}
```

So like I mentioned before, the Command itself contains all of the information necessary to execute itself once we get to the handler. Executing commands, by the way, is as simple as creating extension methods for `ITypedCommand<T>` and `ICommand` (which we’ll get to later):

```
public static class Commands
{
	public async static Task<object> Execute<TCommand>(this TCommand command) where TCommand : ICommand
	{
		try
		{
			ICommandHandler<TCommand> handler = ServiceLocator.Current.GetInstance<ICommandHandler<TCommand>>();

			return await handler.Handle(command);
		}
		catch (InvalidOperationException)
		{
			// When service locator is not set, ignore it.

			return null;
		}
	}

	public async static Task<object> Execute<TCommand, T>(this TCommand command) where TCommand : ITypedCommand<T>
	{
		try
		{
			ITypedCommandHandler<TCommand, T> handler =
				ServiceLocator.Current.GetInstance<ITypedCommandHandler<TCommand, T>>();

			return await handler.Handle(command);
		}
		catch (InvalidOperationException ex)
		{
			// When service locator is not set, ignore it.

			return null;
		}
	}
}
```

This means that when you create an instance of a Command, you can simply call `Execute(…)` knowing that it will be handled by the implementation of `ITypedCommandHandler`. Using generics here is also great because you can simply keep listing commands that this class implements, and it will force you to create a `Handle(…)` method for each of those commands.

```
public class TypedCommandHandler<T> :
	ITypedCommandHandler<SyncDownTableCommand<T>, T>
{
	private readonly IMobileServiceClient _mobileServiceClient;

	public TypedCommandHandler(IMobileServiceClient mobileServiceClient)
	{
		if (mobileServiceClient == null) throw new ArgumentNullException("mobileServiceClient");

		this._mobileServiceClient = mobileServiceClient;
	}

	// this is the implementation of the command handler for a single command. the service locator finds it
	// and will execute it, returning the result to the call site. this and the CommandHandler classes are the
	// only two allowed to have access to your actual database context, so ALL write operations (anything not returning
	// IEnumerable<T> collections) occur in these classes.
	public async Task<object> Handle(SyncDownTableCommand<T> command)
	{
		IMobileServiceSyncTable<T> table = this._mobileServiceClient.GetSyncTable<T>();

		if (!NetworkInterface.GetIsNetworkAvailable()) return false;

		try
		{
			await table.PullAsync(command.QueryId, table.Where(command.Query));
		}
		catch (MobileServiceInvalidOperationException ex)
		{
			string uri = ex.Request.RequestUri.ToString();
		}

		return true;
	}
}
```

You now have access to download data into your local tables! The setup is a bit complicated, so I’ll run you through an example of how to use it here:

```
await new SyncDownTableCommand<VehicleManufacturer>("vehicleManufacturers", 
       vm => vm.Country == "USA")
       .Execute<SyncDownTableCommand<VehicleManufacturer>, VehicleManufacturer>();
```

Note for Unity users: we had some significant challenges trying to figure out how to wire up these Command handlers (specifically the dual-type parameter `ITypedCommandHandler`) for the service locator to find, so I thought I’d share what we came up with. Here are the Unity registrations:

```
private static void RegisterCommands(IUnityContainer container)
{
	container.RegisterType(typeof (ICommandHandler<>), typeof (CommandHandler),
		new HierarchicalLifetimeManager());
		
	container.RegisterType(typeof (ITypedCommandHandler<,>), new HierarchicalLifetimeManager(),
		new InjectionFactory(
			(cont, type, obj) =>
				cont.Resolve(typeof (TypedCommandHandler<>)
                            .MakeGenericType(type.GenericTypeArguments[1]), obj)));
}

private static void RegisterMisc(IUnityContainer container)
{
	UnityServiceLocator locator = new UnityServiceLocator(container);
	ServiceLocator.SetLocatorProvider(() => locator);
}
```

That’s the complicated part – I’ll let you figure how the implementation of the `ICommandHandler`. Hint - it’s very similar, though without the additional `T` parameter.

### Synchronizing Local Data to your Azure Environment
Since you have now set up the entire framework, the synchronization portion is essentially complete. It’s worth noting that CQRS has saved you a few steps in additional complications. For example, calling `PullAsync(…)` on a table would fail if you already have pending change operations within that table. Since you have a set of read-only tables, however, that won’t be an issue. You’re never going to call `PullAsync(…)` on your write tables, and there will never be pending operations on your read-only tables.

The next step here, to conclude the round trip from your Azure DB, is to call `PushAsync()` on your `MobileServiceDatabaseContext`’s SyncContext. This will immediately trigger the application to start pushing any pending operations up to the server, and will deal with conflicts using the SyncHandler you set up.

Since it is not a read-only operation I have implemented a `PushAllDataToServer` as an `ICommand`, which is handled in my `CommandHandler` class within my Infrastructure layer. That’s it!

### Summary
You now have the infrastructure to maintain a database in Azure that all of your Universal App clients (technically it’s broader than that, but it’s outside the scope of this post) can download for offline purposes, make any changes they need to, and push up a fresh version to your servers. This is great for something like a Line of Business app that we built because our client’s reps can download data at their leisure, complete their work without worrying about network connectivity, and asynchronously push all of their data changes up to Azure without worrying about data conflicts. I would be interested to hear of other use cases for this technology!