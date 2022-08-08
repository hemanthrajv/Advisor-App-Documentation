# Flutter App/Clean Architecture

The document focuses on the architectural patterns and best practices to use with Flutter Projects.

The document only focuses on implementation and usage patterns. To know more about Clean
Architecture, [here](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
is a nice write up.

A good architecture allows you to decouple units of code and improve maintainability of code. But at
the same time, a very complex architecture might also result in having opposite effect. The idea of
Clean Architecture is to keep things simple where it focuses on separation of code based on layers
and enforces for outer layers to depend on the inners and the vice-versa not possible.

BLOCK DIAGRAM:

![Image](./clean_architecture.png?raw=true)

### Entities (Enterprise Business Rules):

Entities is the core layer of the architecture where all the data will be available, Entities should
be written in core folder(to which view model will be interacted to get data from the third party)
and states should be written in models folder.

### Use Cases (Application Business Rules):

This layer represents business actions.It encapsulate and implements all the use cases in the
system.

### Interface/Adapters (Controllers, Gateways, Presenters):

This layer can be used to communicate with Ui and Third party dependency as per requirements.Only
this layer should be responsible for state changes and services.

### Frameworks and Drivers (UI, Database, Web, Devices):

This layer represents UI and other third party dependency that are needed which will be interacted
with view model

### Installation

The first thing we need to do is add the bloc package to our pubspec.yaml as a dependency.

```dart
dependencies:flutter:sdk: flutter# State
provider: ^5.0.0
state_notifier: ^0.7.0
flutter_state_notifier: ^0.7.0
built_value: ^7.0.9
built_collection: ^5.0.0


dev_dependencies:flutter_test:sdk: flutter
built_value_generator: ^7.0.9
build_runner: ^1.7.4

```

### Entities Layer:

This is also called as core layer. This primarily consists of core interfaces that the whole
application depends on from which implementation for external services like database,API etc... can
be achieved. Lets create a abstract class named Service

```dart
abstract class Service {
  Future<void> init();
}
```

lets a new service class named ApiService which inherits Service class so this child class will also
have properties of its parent class Service with its additional features.

```dart
abstract class ApiService extends Service {
  //Lets assume count is always saved to server 
  Future<int> getInitialData();

  Future<void> saveData({int data});
}

```

Now if a class implements ApiService it will all the properties of ApiService and its parent Class.
In Implementation class we can write all the implementations that as to be done to interact with the
third party.

The Service will be later implemented in Use Case layer.

*AppState*
AppState contains values used will be used by the ui.

```dart
part 'app_state.g.dart';

abstract class AppState implements Built<AppState, AppStateBuilder> {
  factory AppState([void Function(AppStateBuilder)? updates]) = _$AppState;

  AppState._();

  Map<String, dynamic>? toJson() {
    return serializers.serializeWith(AppState.serializer, this)
    as Map<String, dynamic>?;
  }

  static AppState? fromJson(Map<String, dynamic> json) {
    return serializers.deserializeWith(AppState.serializer, json);
  }

  static Serializer<AppState> get serializer => _$appStateSerializer;


  int get value;
}

```

### UseCase :

This layers consists of all the implementation that has to be done with the external services

Here we have implemented the methods written in the ApiService class. Our project is not directly
dependent on this implementation but with ApiService class.

```dart
class ApiServiceImpl implements ApiService {
  @override
  void init() {
    ...
  }


  @override
  Future<int> getInitialData() async {
    ...

  }

  @override
  Future<void> saveData({int data}) async {
    ...
  }
}
```

### Interface Adapters:

This layer represents viewModel that will control the state and UI reacts to state changes . This
has to be written in View Model Lets create a class AppViewModel that will change state when Api is
called .

```dart
abstract class AppStateNotifier<T> extends StateNotifier<T> {
  AppStateNotifier(T state) : super(state);

  T getState() => state;

  Stream<T> getStream() => stream;

  void init() {}

  ;
}

class AppViewModel extends AppStateNotifer<AppViewModel> {
  AppViewModel() : super(AppState());

  ApiService service = ApiServiceImpl();

  Future<void> init() async {
    final value = await service.getInitialData();// getInitialData method from ApiService class be will invoked
    state = state.rebuild((b) => b.value = value);
  }

  Future<void> increment() async {
    state = state.rebuild((b) => b.value = b.value++);
    await service.saveData({data: state.value}); // saveData method from ApiService class be will invoked
  }

  Future<void> reset() {
    state = state.rebuild((b) => b.value = 0);
    await service.saveData({data: 0});

  }

}

```

ViewModel acts as a link between Entities and View .State should be changed only using viewModel.
When State get changed UI will react accordingly.

The Ui can be accessed using context, but we can have extensions and Mixins for easy accessibility

```dart
extension ProviderUtils on BuildContext {
  AppViewModel get appViewModel => read<AppViewModel>();


  AppState get appState => watch<AppState>();

}
mixin AppProviderMixin<S extends StatefulWidget> on State<S> {
  // accessors

  AppViewModel get appViewModel => context.read<AppViewModel>();

  // listeners
  AppState get appState => context.watch<AppState>();
}
```

### Frameworks & Drivers

Frameworks and Drivers are the UI components and other external parties like database, API, web etc

For UI we use a simple counter example

```dart
import 'package:flutter/material.dart';

void main() {
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
        title: 'Flutter Demo',
        theme: ThemeData(
          primarySwatch: Colors.blue,
          visualDensity: VisualDensity.adaptivePlatformDensity,
        ),
        home: StateNotifierProvider<AppViewModel, AppState>(
          create: (_) => AppViewModel(),
          child: child,
        );
        child: MyHomePage(title: 'Flutter Demo Home Page'),)
    ,
    );
  }
}

class MyHomePage extends StatefulWidget {
  MyHomePage({Key key, this.title}) : super(key: key);

  final String title;

  @override
  State<MyHomePage> createState() => _MyHomePageState();
}
/*
AppProviderMixin is used to access extension like appViewModel and appState
to access the methods in AppViewModel and access data from AppState respectively
 */
class _MyHomePageState extends State<MyHomePage> with AppProviderMixin {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(widget.title),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            Text(
              'You have pushed the button this many times:',
            ),

            Text(
              '${appState.value}', // When value in the state changes it will reflected here in the UI
              style: Theme
                  .of(context)
                  .textTheme
                  .headline4,
            ),
          ],
        ),
      ),
      floatingActionButton: Column(
        mainAxisSize: MainAxisSize.min,
        mainAxisAlignment: MainAxisAlignment.end,
        children: <Widget>[
          FloatingActionButton(
            onPressed: () {
              appViewModel
                  .increment(); // When this method is called the value in the  will be incremented and UI will be changed accordingly 
            },
            tooltip: 'Increment',
            child: Icon(Icons.add),
          ),
          FloatingActionButton(
            onPressed: appviewModel.reset, // When this method is called the value in the  will be set to Zero and it will reflected back in the UI 
            tooltip: 'Reset',
            child: Icon(Icons.close),
          ),
        ],
      ),
    );
  }

}

```






