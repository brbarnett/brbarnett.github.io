---
layout: post
title: Windows Universal App Series - Creating a Windows Universal App with Prism 5.0 & SQLite for Offline Persistence
redirect_from: "/blog/2015/03/windows-universal-app-series-creating-a-windows-universal-app-with-prism-50-sqlite-for-offline-persistence/"
tags: windows-universal-apps
---

### Introduction
I recently had the pleasure of working with a client who wanted to upgrade the platform of their Windows Mobile 6.5 Line of Business, inventory management app to something a bit more recent and cutting-edge. Their goal was to modernize their devices (the old device was a Motorola ES400), and update their software to something significantly more maintainable, scalable and extensible.

<!--more-->

Since the Windows Mobile OS, Microsoft has gone through a few iterations of 7.X and 8.0 releases to their mobile operating systems. Those operating systems – from a development perspective – were specific to the mobile platform (as opposed to desktop or tablet), so developers created projects and used APIs that were targeted at working only with mobile devices. Now, with the advent of Windows 8.1 (and the impending release of Windows 10 later this year), there have been far more advancements that make Windows apps much more device-agnostic: enter, the Windows Universal App. These are apps that are built on the fundamentals of the MVVM pattern, targeting a cohesive and shared set of Windows APIs between desktop, tablet and phone alike while allowing developers to present content differently between platform-specific projects in the same solution. It is for this reason that we chose to build the application as a Universal App.

### The Windows Universal Apps Architecture
Creating a "Store Apps" project automatically generates three grouped projects: ones for Windows 8.1, Windows Phone 8.1 and a common Shared library, which ends up working together something like this:

![_config.yml]({{ site.baseurl }}/images/2015-3-5-windows-universal-app-series-creating-a-windows-universal-app-with-prism-50-sqlite-for-offline-persistence/wua1.png)

The architecture here lends itself very well to the MVVM pattern, where the platform-specific projects serve as repositories for Views and Controls (think Barcode Scanner, possibly only available on a phone), and the Shared library houses their common Controls and ViewModels that are a loosely-coupled version of what would otherwise be consumption and orchestration of service and command calls in code-behinds (see: Smart UI anti-pattern). While we're at it, it's a good idea to abstract all of the application-specific functionality into an application layer, and inject its services as dependencies (via an IoC container like Unity) into your ViewModels.

### MVVM with Prism 5.0
I have found that a stock solution for a Universal App comes with a lot of extra code overhead, where the developer must be very explicit in handling of events like application startup and suspension. This works for the purist who craves explicit logic, but to me it feels like an easy way to get yourself into trouble. To keep things simple, we implemented the super-MVVM-friendly library Prism 5.0, which is a service locator-based application wrapper that ensures loose coupling of your Views and ViewModels (using naming conventions to auto-wire them in the background). It also cleans up the App.xaml.cs and navigation logic between ViewModels by abstracting most of the otherwise explicit logic into virtual, overrideable methods in the background. Here is my simplified version of an App.xaml for my project, and then its respective App.xaml.cs partial:

```
<prism:MvvmAppBase
    x:Class="Rightpoint.Client.App"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:prism="using:Microsoft.Practices.Prism.Mvvm"
    xmlns:local="clr-namespace:Microsoft.Practices.Prism.Mvvm.MvvmAppBase"
    xmlns:system="using:System">
        
	<!-- this is a great spot to load your application resource libraries -->
		
</prism:MvvmAppBase>
```

Note that we're inheriting from Prism base classes now instead of the standard Application class. The Prism base class here is what drives most of the auto-wiring, which we'll do a bit of our own implementation (overriding) of in our App partial:

```
public sealed partial class App: MvvmAppBase
{
	public App()
	{
		this.InitializeComponent();
	}

	protected override async Task OnLaunchApplicationAsync(LaunchActivatedEventArgs args)
	{
		// Unity registrations of application singletons
		UnityContainerFactory.RegisterInstanceOnContainer(this.NavigationService);
			
		// use previousState to update your logic based on whether you're launching fresh, or 
		// are resuming from suspension. in this case, I start over unless it's a resume, which
		// automatically picks up where the user left off
		ApplicationExecutionState previousState = args.PreviousExecutionState;

		switch (previousState)
		{
			case ApplicationExecutionState.Terminated:
			case ApplicationExecutionState.ClosedByUser:
			case ApplicationExecutionState.Running:
				break;
			default:
				this.NavigationService.Navigate("Main", null);
				break;
		}
	}

	// this translates PageToken strings into strongly-typed Views. this is particularly important due to our
	// separation of the ViewModels to the Shared library, which is a different project than where our Views reside
	protected override Type GetPageType(string pageToken)
	{
		string appType = "WindowsPhone";
#if WINDOWS_APP
		appType = "Windows";
#endif
		Type type =
			Type.GetType(string.Format(CultureInfo.InvariantCulture,
				this.GetType()
					.GetTypeInfo()
					.AssemblyQualifiedName
					.Replace(this.GetType().FullName, this.GetType().Namespace + "." + appType + ".Views.{0}Page"), (object) pageToken));

		if (type != null) return type;

		throw new ArgumentException(
			string.Format(CultureInfo.InvariantCulture,
				ResourceLoader.GetForCurrentView("/Microsoft.Practices.Prism.StoreApps/Resources/")
					.GetString("DefaultPageTypeLookupErrorMessage"), (object) pageToken,
				(object) (this.GetType().Namespace + ".Views")), "pageToken");
	}

	// a bit of extra code to work with Unity as an IoC container
	protected override Task OnInitializeAsync(IActivatedEventArgs args)
	{
		// wire up Unity to work with View Models
		ViewModelLocationProvider.SetDefaultViewModelFactory(UnityContainerFactory.ResolveFromContainer);
		ViewModelLocationProvider.SetDefaultViewTypeToViewModelTypeResolver(this.ViewTypeToViewModelTypeResolver);

		return base.OnInitializeAsync(args);
	}

	
	// this translates strongly-typed Views into their respective ViewModel
	private Type ViewTypeToViewModelTypeResolver(Type type)
	{
		string namespaceFormat = this.GetType().Namespace + ".ViewModels.{0}ViewModel";
		Type viewModelType = Type.GetType(String.Format(namespaceFormat, type.Name));
		return viewModelType;
	}
}
```

That's the entire App.xaml.cs. Compare with the stock, and you can see why Prism is a favorite among Universal App developers. For example, it allows navigation with a simple call to `this.NavigationService.Navigate("Main", null)`, which will bring up your MainPage.xaml on the device. **IMPORTANT: Make special note that I have registered NavigationService (a singleton of INavigationService) with Unity, so I can inject the INavigationService dependency directly into my ViewModels. You otherwise lose access to this NavigationService in your ViewModels, which is crucial to navigating between pages of your app.**

### Creating Your First Pages
Let's get started and actually create a few pages (one main one, and a second to provide a navigation example). Create a page called MainPage.xaml in the WP8.1 project, which will automatically generate a partial in the background. Quick note: to make the same page work on the desktop/tablet, you'll need to create a View with the same name in the Windows 8.1 project, but you can share the ViewModel. Here is some basic XAML:

```
<prism:VisualStateAwarePage
    x:Class="Rightpoint.Client.WindowsPhone.Views.MainPage"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:prism="using:Microsoft.Practices.Prism.StoreApps"
    xmlns:mvvm="using:Microsoft.Practices.Prism.Mvvm"
    mvvm:ViewModelLocator.AutoWireViewModel="True">

    <Grid>
	<StackPanel>
		<TextBlock Text="{Binding HelloWorld}" />
		<Button Command="{Binding NavigateCommand}" Content="Second Page" />
	</StackPanel>
    </Grid>
</prism:VisualStateAwarePage>
```

Notice that we're telling Prism to auto-wire the ViewModel using naming conventions, which we set up in our App.xaml.cs. The partial behind MainPage is amazingly simple, and you have Prism to thank for it:

```
namespace Rightpoint.Client.WindowsPhone.Views
{
    public sealed partial class MainPage : VisualStateAwarePage
    {
        public MainPage()
        {
            this.InitializeComponent();
        }
    }
}
``` 

That's almost no code! In fact, the only reason you really keep this class around is because the service locator needs a concrete instance to load up the View to begin with – talk about light weight. In this page, we're displaying “hello world” text in a TextBlock control, as well as creating a Button to navigate to our SecondPage View. Let's walk through how to wire that all up with our aptly-named MainPageViewModel:

```
namespace Rightpoint.Client.ViewModels
{
    public class MainPageViewModel: ViewModel
    {
	 // capture all injected dependencies from Unity. we're really just extending the good loose-coupling
	 // practices that we've already started by using an IoC container to abstract at least an Application layer,
	 // but likely also a Domain and Infrastructure
	 private readonly IDataService _dataService;
        private readonly INavigationService _navigationService;

	 // these properties are bindable by your View. be careful to use SetProperty in the setter
	 // in order to notify the View of its changing so it can update appropriately
        private string _helloWorld;
        public string HelloWorld
        {
            get { return this._helloWorld; }
            set { this.SetProperty(ref this._helloWorld, value); }
        }

	 // delegate commands are used to bind command functionality from a ViewModel to a View. they are
	 // bound the same way as regular properties. if you need to pass a CommandParameter, these will be of
	 // type DelegateCommand<T>, and your implementing methods will take in a T-typed parameter
        private DelegateCommand _navigateCommand;
        public DelegateCommand NavigateCommand
        {
            get { return this._navigateCommand; }
            set { SetProperty(ref this._navigateCommand, value); }
        }

        public MainPageViewModel(INavigationService navigationService, IDataService dataService)
        {
	 	if (dataService == null) throw new ArgumentNullException("dataService");
	 	if (navigationService == null) throw new ArgumentNullException("navigationService");
	 	
	 	this._dataService = dataService;
	 	this._navigationService = navigationService;
			
	 	// wire up our delegate command to its implementation method
	 	this.NavigateCommand = new DelegateCommand(this.OnNavigateCommand);
        }
		
	 // implement the command logic here. again, if you used a DelegateCommand<T>, this would have taken in
	 // a T-typed parameter that you could use in your logic here
	 private void OnNavigateCommand()
	 {
	 	NavigationParameterPoco poco = new NavigationParameterPoco();
	 	poco.Message = "I am a standard POCO, which needs to be serialized for Prism to remember my state.";
			
	 	// make sure to serialize all of your navigation parameters, to be deserialized in OnNavigatedTo on the
	 	// recipient page. Prism tries to hold all of your navigation parameters in memory, but will give you a null
	 	// if you're not passing a string - it's NOT good at holding non-serialized POCOs. note: instead of referencing JsonConvert
	 	// everywhere, I prefer to create an extension method on "object" to serialize (and, conversely, an extension on 
	 	// string to deserialize into a generic type T)
	 	this._navigationService.Navigate("Second", JsonConvert.SerializeObject(poco));
	 }

	 // add all data load logic here. use the navigationParameter to pass data between pages. technically, Prism comes
	 // with a [RestorableState] attribute that you can use to decorate your properties, which will *automatically* add
	 // these entities into memory, which will come back in the viewModelState parameter. for our purposes here, however, we'll
	 // work with local storage (SQLite) and load in data every time
	 public async override void OnNavigatedTo(object navigationParameter, 
            NavigationMode navigationMode, Dictionary<string, object> viewModelState)
        {
            base.OnNavigatedTo(navigationParameter, navigationMode, viewModelState);
			
	 	// special note here: if you want your app UI to be responsive when you're pulling data from your services,
	 	// you need to think about using the async/await keywords. this spawns threads outside of your main UI thread,
	 	// which keeps the UI responsive in the meantime
	 	this.HelloWorld = await this._dataService.GetHelloWorld();
        }
		
	 // this fires prior to page navigation, but is often not used. the most 
	 // common usage is storing data in the viewModelState on app suspending
        public async override void OnNavigatedFrom(Dictionary<string, object> viewModelState, bool suspending)
        {
            base.OnNavigatedFrom(viewModelState, suspending);
        }
    }
}
```

So now we have created a path where we can navigate between pages, so let's catch the navigation on the OnNavigatedTo of our SecondPage:

```
// here is the override for OnNavigatedTo on the SecondPageViewModel. note that we're deserializing the navigationParameter, which
// we serialized on the navigation call from MainPageViewModel
public async override void OnNavigatedTo(object navigationParameter, 
       NavigationMode navigationMode, Dictionary<string, object> viewModelState)
{
	base.OnNavigatedTo(navigationParameter, navigationMode, viewModelState);
	
	if(navigationParameter == null) throw new ArgumentNullException("navigationParameter");
	
	NavigationParameterPoco poco = JsonConvert.DeserializeObject<T>(navigationParameter);
}
```

One of the more interesting features here is that if you navigate away from SecondPage now, and then navigate back (think back button), navigationParameter will still contain the same value that it does now. Prism auto-hydrates it by holding the previous Navigate call in memory, including your manually-serialized navigationParameter string (which is why you serialized it to a string to begin with).

When considering async/await, note that you need to return a `Task` or `Task<T>` from your DataService implementation to make it awaitable. I'll go into more depth later on the concrete implementation, but here's a quick interface that you'll need to implement for this case:

```
public interface IDataService
{
	Task<string> GetHelloWorld();
}
```

`Task<T>` is awaitable, which means that when you call await, it will return whatever type `T` is (in this case, it's a string). If you use an untyped `Task`, it's comparable to awaiting a void method. I won't go any further into async/await here, but it might be a good idea to read up on what it does because it has the potential to get you in trouble if you don't know what its black box behind the scenes is doing.

### Local DB via SQLite
So this is great, but it doesn't make it an “application” yet, really. Let's fix that by setting up a local data store in SQLite to allow persistence and read (truly, any CRUD operations) of your data. Start by loading up SQLite.Net PCL and SQLite.Net PCL – WinRT Platform from NuGet, which will enable you to create data tables using annotated classes.

First off, you need a data context. This is a lot like creating a DbContext with Entity Framework, but it's specific to a SQLiteConnection instead (and the syntax is a little different) – similar effect, though. Here's an example of a simple data context that we'll use to populate our UI's HelloWorld TextBlock above:

```
public class LocalDatabaseClient : SQLiteConnection
{
	private const string Db = "local.db";

	public LocalDatabaseClient()
		: base(new SQLitePlatformWinRT(), Db)
	{
		// you are required to call this for every single table you're creating, much like
		// you would in a EF DbContext
		base.CreateTable<UiString>();
	}
}
```

We're not going to get into CQRS quite yet in this post, but I will make the point right up front that I like to create a read-only version of these data contexts to make sure we stick to our patterns that we chose up front – you can inject the read-only version via an interface with Unity, so that nobody can write to these tables explicitly unless you separately or instead inject the writeable version. Create a new class and implement this interface to make it read-only:

```
public interface ILocalReadOnlyDatabaseClient
{
	IEnumerable<T> GetLocalTableReadOnly<T>() where T : new();
}
```

Everything comes out as an `IEnumerable`, which allows both lazy-loading and read-only all in one shot.

Now we need to create our class to store data that we'll pull in our concrete implementation of `DataService`. These classes are also similar to what we'd create for Entity Framework, in that we can use annotations to tell SQLite how to store the data.

```
public class UiString
{
	[PrimaryKey]
	public string Id { get; set; }

	public string Text { get; set; }
	
	// this property will *not* be persisted as a column in your data store due to the [Ignore] attribute.
	// I am including this only as an example - perhaps this would be good for something like a created 
	// DateTime that you don't actually want to store
	[Ignore]
	public string IgnoredProperty { get; set; }
}
``` 

When the DB context calls `CreateTable<T>`, it automatically creates a place in local storage that's in the shape of type `T` in which to store your data. This will give us access to query the data, which is actually rather simple with our `ILocalReadOnlyDatabaseClient` that we can inject into our Application layer. Let's take a look at the DataService from before as a concrete implementation:

```
public class DataService : IDataService
{
	private readonly ILocalReadOnlyDatabaseClient _localReadOnlyDatabaseClient;
	
	public DataService(ILocalReadOnlyDatabaseClient localReadOnlyDatabaseClient)
	{
		if(localReadOnlyDatabaseClient == null) throw new ArgumentNullException("localReadOnlyDatabaseClient");
		
		this._localReadOnlyDatabaseClient = localReadOnlyDatabaseClient;
	}
	
	// remember that async is important here, as is making the return time Task<T> to get
	// your results when you await the call in the ViewModel
	public async Task<string> GetHelloWorld()
	{
		// access database table, which gets us an IEnumerable<T> of the result. if you'd like to see
		// results right away, simply call an execution method at the end like .ToList() to enumerate
		// the collection real-time
		IEnumerable<UiString> uiStrings = this._localReadOnlyDatabaseClient.GetLocalTableReadOnly<UiString>();
		
		// we can now pull our result directly out of the IEnumerable<T>
		return uiStrings.Single(uis => uis.Id == "HelloWorldString").Text;
	}
}
```

Notice that we injected `ILocalReadOnlyDatabaseClient` because we're not planning on doing any writing through our Application layer (eventually, we'll do that via CQRS' command pattern instead). If you're interested in the writeable side of things, you can do that by injecting your `SQLiteConnection` instance. Take a peek within the SQLiteConnection class (thankfully open source, for those who don't have DotPeek) for a pretty solid CRUD API. This allows calls such as `this._localWriteableDatabaseClient.Insert(uiString)` or `this._localWriteableDatabaseClient.Update(uiString)` to commit data to the database.

### Summary
This should get you started at least on creating an app that will work on Windows desktop, tablet and phone. What's more, you are now set up for success with a scalable architecture (thanks to Unity and Prism) that allows you to persist your users' data locally to their machines and/or devices, which means it serves as a true application (to what degree, I'll let you use your imagination). Stay tuned for more posts that discuss other advanced features, such as working with Azure's Mobile Services to synchronize client data with your cloud Azure DB.