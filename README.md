# Provider Architecture - Deprecated on 21 April 2020

## Notice

### V2 of this package is renamed to [Stacked](https://pub.dev/packages/stacked)

Stacked is the name of the architecture that was originally inspired by MVVM for the first implementation. It has some name changes but everything else is the same so a migration would be painless. There are multiple things that it improves on. The two main things are ViewModel management (creating only when required, not creating it everytime it's rebuilt), responsive services which allows viewmodels to register for service changes. A series will be released that covers the entire architecture stack used for production development along with stacked.

## Migrating from provider_architecture to Stacked

Lets start with a statement to ease your migration panic 😅 stacked is the exact same code from provider_architecture with naming changes and removal of some old deprecated properties. If you don't believe me, open the repo's side by side and look at the lib folders. Well, up till yesterday (22 April 2020) I guess, when I updated the BaseViewModel. I wanted to do this to show that stacked is production ready from the go. It's a new package but it's been used by all of you and the FilledStacks development team for months in the form of provider_architecture. With that out of the way lets start the migrate.

### ViewModelProvider Migration

This class has now been more appropriately named `ViewModelBuilder`. This is to match it's functionality more closely. Building UI FROM the ViewModel. The ViewModel is used to drive the state of the reactive UI.

Migrations to take note of:

- `ViewModelProvider` -> `ViewModelBuilder`
- Named constructor `withoutConsumer` is now called `nonReactive`
- Named constructor `withConsumer` is now called `reactive`
- Instead of passing a constructed `ViewModel` which was constructing every rebuilder we pass a `viewModelBuilder`. A function that returns a `ChangeNotifier`.
- `reuseExisting` has changed to `disposeViewModel` and now has a default value of true. If you used `reuseExisting=true` it has to change to `disposeViewModel=false`.

Lets look at that in code. We'll go over withoutConsumer / nonReactive first

```dart
class HomeViewMultipleWidgets extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ViewModelProvider<HomeViewModel>.withoutConsumer(
      viewModel: HomeViewModel(),
      onModelReady: (model) => model.initialise(),
      reuseExisting: true,
      builder: (context, model, _) => Scaffold(
        floatingActionButton: FloatingActionButton(
          onPressed: () {
            model.updateTitle();
          },
        ),
        body: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[TitleSection(), DescriptionSection()],
        ),
      ),
    );
  }
}
```

Will Change to

```dart
class HomeViewMultipleWidgets extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ViewModelBuilder<HomeViewModel>.nonReactive( // Take note here
      viewModelBuilder: () => HomeViewModel(), // Take note here
      disposeViewModel: false, // Take note here
      onModelReady: (model) => model.initialise(),
      builder: (context, model, _) => Scaffold(
        floatingActionButton: FloatingActionButton(
          onPressed: () {
            model.updateTitle();
          },
        ),
        body: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[TitleSection(), DescriptionSection()],
        ),
      ),
    );
  }
}
```

For the withConsumer function we do the following

```dart
class HomeView extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ViewModelProvider<HomeViewModel>.withConsumer(
    );
  }
}
```

Changes to

```dart
class HomeView extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ViewModelBuilder<HomeViewModel>.reactive( // Take note here

    );
  }
}
```

### ProviderWidget Migration

The only change here was the name.

```dart
class DuplicateNameWidget extends ProviderWidget<Human> {

}

// Becomes

class DuplicateNameWidget extends ViewModelWidget<Human> {

}
```

The rest of the package is all new functionality which can be seen above. Please check out the issues for tasks we'd like to add. If you would like to see any functionality in here please create an issue and I'll asses and provide feedback.

## End of notice

This package contains base widgets that can be used to implement the MvvmStyle provider architecture as laid out by [FilledStacks](https://www.youtube.com/filledstacks).

## ViewModelProvider

The `ViewModelProvider` was first built in the [Provider Architecture Tutorial](https://youtu.be/kDEflMYTFlk) where it was titled BaseView. The `ViewModelProvider` is used to create the "binding" between a ViewModel and the View. There is no two-way binding in this architecture, which is why I don't want to say it's an Mvvm implementation. The `ViewModelProvider` wraps up all the `ChangeNotifierProvider` code which allows us to trigger a rebuild of a widget when calling `notifyListeners` within the ViewModel.

A ViewModel is simply a dart class that extends `ChangeNotifier`. The `ViewModelProvider` has 2 constructors, one with a builder and one without a builder. The tutorial mentioned above emulates the default implementation which has been put into the `.withConsumer` named constructor. The `.withoutConsumer` constructor is for UI that does not require the model at the constructor level. The withoutConsumer construction was born in [this tutorial](https://youtu.be/HUSqk0OrR7I?t=224) where we wanted to reduce the boiler plate when the same data has to go to multiple widgets.

### With Consumer

An example of this would be the traditional View-ViewModel setup we have.

```dart

// View
class HomeView extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    // Using the withConsumer constructor gives you the traditional viewmodel
    // binding which will rebuild when notifyListeners is called. This is used
    // when the model does not have to be consumed by multiple different UI's.
    return ViewModelProvider<HomeViewModel>.withConsumer(
      viewModelBuilder: () => HomeViewModel(),
      onModelReady: (model) => model.initialise(),
      builder: (context, model, child) => Scaffold(
        floatingActionButton: FloatingActionButton(
          onPressed: () {
            model.updateTitle();
          },
        ),
        body: Center(
          child: Text(model.title),
        ),
      ),
    );
  }
}

// ViewModel
class HomeViewModel extends ChangeNotifier {
  String title = 'default';

  void initialise() {
    title = 'initialised';
    notifyListeners();
  }

  int counter = 0;
  void updateTitle() {
    counter++;
    title = '$counter';
    notifyListeners();
  }
}

```

When `notifyListeners` is called in the ViewModel the builder is triggered allowing you to rebuild your UI with the new updated ViewModel state. The process here is you update your data then call `notifyListeners` and rebuild your UI.

### Without Consumer

The `.withoutConsumer` constructor is best used for providing your ViewModel to multiple children widgets. It was created to make it easier to build and provide the same ViewModel to multiple UI's. It was born out of my [Responsive UI architecture](https://youtu.be/HUSqk0OrR7I) where we would have to provide the same ViewModel to all the different responsive layouts. Here's a simple example.

```dart
// Viewmodel in the above code

// View
class HomeViewMultipleWidgets extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ViewModelProvider<HomeViewModel>.withoutConsumer(
      viewModelBuilder: () => HomeViewModel(),
      onModelReady: (model) => model.initialise(),
      builder: (context, model, _) => Scaffold(
        floatingActionButton: FloatingActionButton(
          onPressed: () {
            model.updateTitle();
          },
        ),
        body: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[TitleSection(), DescriptionSection()],
        ),
      ),
    );
  }
}

class TitleSection extends ProviderWidget<HomeViewModel> {
  @override
  Widget build(BuildContext context, HomeViewModel model) {
    return Row(
      children: <Widget>[
        Text(
          'Title',
          style: TextStyle(fontSize: 20),
        ),
        Container(
          child: Text(model.title),
        ),
      ],
    );
  }
}

class DescriptionSection extends ProviderWidget<HomeViewModel> {
  @override
  Widget build(BuildContext context, HomeViewModel model) {
    return Row(
      children: <Widget>[
        Text(
          'Description',
          style: TextStyle(fontSize: 14, fontWeight: FontWeight.w700),
        ),
        Container(
          child: Text(model.title),
        ),
      ],
    );
  }
}
```

So what we're doing here is providing the ViewModel to the children of the builder function. The builder function itself won't retrigger when `notifyListeners` is called. Instead we will extend from `ProviderWidget` in the widgets that we want to rebuild from the ViewModel. This allows us to easily access the ViewModel in multiple widgets without a lot of repeat boilerplate code. We already extend from a `StatelessWidget` so we can change that to `ProviderWidget` and we always have a build function so we simply add the ViewModel as a parameter to that. This is the same as calling `Provider<ViewModel>.of` in every widget we want to rebuild.

### Reusing Viewmodel Instance

An example of how to use one viewmodel instance across the application with the help of [get_it](https://github.com/fluttercommunity/get_it).

```dart
// Registering the viewmodel inside the get_it service locator
GetIt locator = GetIt.instance;

setupServiceLocator() {
  // Singleton of the viewmodel
  locator.registerLazySingleton<HomeViewModel>(() => HomeViewModel());
}

// View
class HomeView extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    // Using the withConsumer constructor gives you the traditional viewmodel
    // binding which will rebuild when notifyListeners is called. But instead
    // of creating a new instance of the viewmodel, the singleton instance from
    // the get_it locator is passed through.
    return ViewModelProvider<HomeViewModel>.withConsumer(
      viewModelBuilder: () => locator<HomeViewModel>(),
      onModelReady: (model) => model.initialise(),
      reuseExisting: true,
      builder: (context, model, child) => Scaffold(
        floatingActionButton: FloatingActionButton(
          onPressed: () {
            model.updateTitle();
          },
        ),
        body: Center(
          child: Text(model.title),
        ),
      ),
    );
  }
}

// ViewModel
class HomeViewModel extends ChangeNotifier {
  String title = 'default';

  void initialise() {
    title = 'initialised';
    notifyListeners();
  }

  int counter = 0;
  void updateTitle() {
    counter++;
    title = '$counter';
    notifyListeners();
  }
}

```

Note that the `ViewModelProvider` constructor is called with parameter `reuseExisting: true`. This enables us to pass an existing instance of a viewmodel.
In this example we register the viewmodel as lazy singleton using [get_it](https://github.com/fluttercommunity/get_it) and inside the `ViewModelProvider`
constructor we simply reference the instance of the viewmodel from the locator. This way we can access the viewmodel data (in this example `counter`) across
the application from different views.

## Provider Widget

The provider widget is an implementation of a widget class that provides us with the provided value as a parameter in the build function of the widget. Above is an example of using the widget but here's another one that doesn't make use of a ViewModel. Lets say for instance you have a data model you want to use in multiple widgets. We can use the `Provider.value` call to supply that value and inside the multiple widgets we inherit from the ProviderWidget and make use of the data.

```dart

// View
class HomeViewProviderWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Provider.value(
        value: Human(name: 'Dane', surname: 'Mackier'),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[FullNameWidget(), DuplicateNameWidget()],
        ),
      ),
    );
  }
}

// Model
class Human {
  final String name;
  final String surname;

  Human({this.name, this.surname});
}

// consuming widget 1
class FullNameWidget extends ProviderWidget<Human> {
  @override
  Widget build(BuildContext context, Human model) {
    return Row(
      children: <Widget>[
        Container(
          child: Text(
            model.name,
            style: TextStyle(fontWeight: FontWeight.bold, fontSize: 30),
          ),
        ),
        SizedBox(
          width: 50,
        ),
        Container(
          child: Text(
            model.surname,
            style: TextStyle(fontWeight: FontWeight.bold, fontSize: 30),
          ),
        ),
      ],
    );
  }
}

// consuming widget 2
import 'package:example/datamodel/human.dart';
import 'package:flutter/material.dart';
import 'package:provider_architecture/provider_architecture.dart';

class DuplicateNameWidget extends ProviderWidget<Human> {
  @override
  Widget build(BuildContext context, Human model) {
    return Row(
      children: <Widget>[
        Container(
          child: Text(model.name),
        ),
        SizedBox(
          width: 50,
        ),
        Container(
          child: Text(model.name),
        ),
      ],
    );
  }
}
```

The package do not implement the architecture for you but it definitely helps the implementation.

### Non-updating Provider Widget

Sometimes you want a widget to have access to the viewmodel but you don't want it to rebuild. In the case of a button that has to call a function on the viewmodel but uses none of the viewmodel state for the UI. In this case you can set the listen value to false for the super constructor of the `ProviderWidget`

```dart
class UpdateTitleButton extends ProviderWidget<HomeViewModel> {
  UpdateTitleButton({
    Key key,
  }) : super(key: key, listen: false);

  @override
  Widget build(BuildContext context, model) {
    return FloatingActionButton(
      onPressed: () {
        model.updateTitle();
      },
    );
  }
}
```
