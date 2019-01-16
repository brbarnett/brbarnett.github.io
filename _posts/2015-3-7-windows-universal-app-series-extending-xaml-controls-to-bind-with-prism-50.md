---
layout: post
title: Windows Universal App Series - Extending XAML Controls to Bind with Prism 5.0
redirect_from: "/blog/2015/03/windows-universal-app-series-extending-xaml-controls-to-bind-with-prism-50/"
tags: windows-universal-apps
---

### Introduction
While I was completing a project for a client of mine, I was working with a ListView that we wanted to create interactions on the screen, such as displaying details of the selected item in bound controls. We were using Prism, however, which naturally (and very deliberately) blocks the developer from making concrete references to Views from ViewModels. The issue that creates is that you cannot subscribe to events generated from a control like a ListView, which is exactly what we needed to do.

<!--more-->

Previous posts in this series:

[Windows Universal App Series – Creating a Windows Universal App with Prism 5.0 & SQLite for Offline Persistence]({{ site.baseurl }}/blog/2015/03/windows-universal-app-series-creating-a-windows-universal-app-with-prism-50-sqlite-for-offline-persistence/)

[Windows Universal App Series – Implementing the CQRS Pattern with a Segregated Data Model to Persist & Sync Offline Data to Azure DB via Azure Mobile Services]({{ site.baseurl }}/blog/2015/03/windows-universal-app-series-implementing-the-cqrs-pattern-with-a-segregated-data-model-to-persist-sync-offline-data-to-azure-db-via-azure-mobile-services/)

### Hooking up the View and ViewModel by way of DelegateCommand Binding
The idea is that we want to set up a mechanism for adding a handler specifically to the ListView’s SelectionChanged event, without adding any additional code to a View’s code-behind. Since the SelectionChanged XAML property only accepts a string that’s meant to refer directly to a handler in the View’s code-behind, this is a problem. So how do we hook this up?

Well, other controls do have this functionality. The one we used to model it is the XAML Button control, which has Command and CommandParameter properties which accept an ICommand and object respectively. It turns out that if we want to model the ListView after the Button, we need to implement similar constructs as DependencyProperties (which we want to do because DependencyProperties have the ability to notify the UI on any changes). Take a look at a few Dependency Properties that we have recreated on a custom extended ListView class:

```
// these are dependency properties that we make available on the XAML controls that
// the developer can interact with directly in XAML. they are bindable, which means
// that you can bind DelegateCommands directly to something like a ListView, which
// is not possible out of the box
private static readonly DependencyProperty SelectionChangedCommandProperty =
    DependencyProperty.Register("SelectionChangedCommand", typeof (ICommand),
        typeof (ListView), null);

public ICommand SelectionChangedCommand
{
    get { return (ICommand) this.GetValue(SelectionChangedCommandProperty); }
    set { this.SetValue(SelectionChangedCommandProperty, value); }
}

private static readonly DependencyProperty SelectionChangedCommandParameterProperty =
    DependencyProperty.Register("SelectionChangedCommandParameter",
        typeof (object), typeof (ListView), null);

public object SelectionChangedCommandParameter
{
    get { return this.GetValue(SelectionChangedCommandParameterProperty); }
    set { this.SetValue(SelectionChangedCommandParameterProperty, value); }
}
```

As you can see, these will raise the bindable properties that we want in order for the developer to be able to bind directly to them. Once we have exposed these as properties from our extended control, we need to subscribe their values to the firing of the SelectionChanged event for subsequent execution.

We do this by creating an event handler within the extended control, and subscribing it upon construction to the inherited SelectionChanged event. The handler then determines whether the Command can execute, and if so, it fires off the Command with its Parameter, which in turn delegates to the ViewModel. Here’s the full extended control example:

```
public class ListView : Windows.UI.Xaml.Controls.ListView
{
	// these are dependency properties that we make available on the XAML controls that
	// the developer can interact with directly in XAML. they are bindable, which means
	// that you can bind DelegateCommands directly to something like a ListView, which
	// is not possible out of the box
	private static readonly DependencyProperty SelectionChangedCommandProperty =
		DependencyProperty.Register("SelectionChangedCommand", typeof (ICommand),
			typeof (ListView), null);

	public ICommand SelectionChangedCommand
	{
		get { return (ICommand) this.GetValue(SelectionChangedCommandProperty); }
		set { this.SetValue(SelectionChangedCommandProperty, value); }
	}

	private static readonly DependencyProperty SelectionChangedCommandParameterProperty =
		DependencyProperty.Register("SelectionChangedCommandParameter",
			typeof (object), typeof (ListView), null);

	public object SelectionChangedCommandParameter
	{
		get { return this.GetValue(SelectionChangedCommandParameterProperty); }
		set { this.SetValue(SelectionChangedCommandParameterProperty, value); }
	}

	public ListView()
	{
		// handle SelectionChanged
		this.SelectionChanged -= this.ListViewSelectionChanged;
		this.SelectionChanged += this.ListViewSelectionChanged;
	}

	// this handler intercepts the SelectionChanged event that fires when a new list item is selected,
	// and translates it into an ICommand that it fires with a Command parameter. bound DelegateCommand<T>
	// in the ViewModel will pick this up and execute as if it were an ICommand from a Button
	private void ListViewSelectionChanged(object sender, SelectionChangedEventArgs e)
	{
		if (this.SelectionChangedCommand == null) return;
		if (!this.SelectionChangedCommand.CanExecute(null)) return;

		// the execution of this ICommand is handled by the ViewModel's DelegateCommand instances,
		// just like it would in the case of a Button
		this.SelectionChangedCommand.Execute(this.SelectionChangedCommandParameter);
	}
}
```

The implementation of the `DelegateCommand` would look as follows:

```
private DelegateCommand<ListView> _listItemSelected;

public DelegateCommand<ListView> ListItemSelected
{
	get { return this._listItemSelected; }
	set { this.SetProperty(ref this._listItemSelected, value); }
}
		
...

this.ListItemSelected = new DelegateCommand<ListView>(this.OnListItemSelected);

...

private void OnListItemSelected(ListView listView)
{
	this.SelectedListItem = 
		(BoundListItemType) // this is whatever type you bound as data source
			listView.SelectedItem; 

	// do something with it (if binding doesn't already take care of that for you)
}
``` 

This methodology will give you access to whatever item has been selected. Alternatively, on a multi-select list, you can use SelectedItems instead of SelectedItem. In this case, SelectedItem will be the most recent item, and SelectedItems will contain the entire list.

To complete this binding, here is the XAML that we might use in this situation:

```
<xc:ListView
	  ItemsSource="{Binding BoundListItemType, Mode=OneWay}"
	  SelectionMode="Single"
	  SelectionChangedCommand="{Binding ListItemSelected}" 
	  SelectionChangedCommandParameter="{Binding RelativeSource={RelativeSource Self}}"
	  ItemTemplate="{StaticResource DefaultTemplate}" />
```

Note the reference to Self being passed to the Command Parameter - this will pass the ListView in as your type, which means you can use it to find its SelectedItem to complete your logic.

### Summary
You’re now equipped to keep on using Prism as it was intended by keeping loose references between your Views and ViewModels in the case of controls where you would like more interaction. Additionally, this process could be applied to ANY event that you would like to handle within your ViewModels, but hadn’t been able to before without adding code into your code-behind, which is a no-no. Happy binding!