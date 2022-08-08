
### Installation

The first thing we need to do is add the bloc package to our pubspec.yaml as a dependency.

```dart
dependencies:
  flutter:
    sdk: flutter
  built_value: ^7.0.9
  flutter_bloc: ^5.0.1
  cupertino_icons: ^0.1.3
  logger: ^0.9.2

dev_dependencies:
  flutter_test:
    sdk: flutter
  built_value_generator: ^7.0.9
  build_runner: ^1.7.4
```

### 1. Data Layer

The data layer's responsibility is to retrieve/manipulate data from one or more sources. This layer is the lowest level of the application and interacts with databases, network requests, and other asynchronous data sources.

Now let us create a `DataService`. Our data service is going to be a very simple class which has two methods:

1. `int incrementData({currValue})` - Which takes in the current counter value and increment it by 1.
2. `int resetData()` - Which resets the Counter to 0.

Our `data_service.dart` will be as follows:

```dart
abstract class DataService {
  //Increment Data
  void incrementData({int currValue});

  // Reset Data
  int resetData();
}
```

**NOTE:** The reason for having the `DataService` as an abstract class is to be independent of third-party dependencies, this is one of the entities or core models in clean architecture,

Now let's implement our `DataService` class in a `LocalService` class. The `local_service.dart` will look like:

```dart
import 'package:counter_using_bloc/data/data_service.dart';

class LocalService implements DataService {

  @override
  int incrementData({int currValue}) {
    return currValue + 1;
  }

  @override
  int resetData() {
    return 0;
  }
}

```

Now we have successfully implemented our data layer. Note that the data layer is completely devoid of any business logic or any other application logics. It simply acts as an abstract reference on which other layers can depend upon.

### 2. Business Logic Layer

Let us move on to the Business Logic Layer. In order to abstract out the dependencies on external bloc packages and customise the behavior of the BLoCs, we can make use of an abstract implementation of the BLoC called as `ApplicationBloc` and have all our other blocs extend this class.
`application_bloc.dart`

```dart
import 'package:built_value/built_value.dart';
import 'package:counter_using_bloc/utils/app_logger.dart';
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

abstract class ApplicationBlocState<E extends ApplicationBlocEvent> {
  bool get loading;

  @nullable
  @BuiltValueField(serialize: false)
  Function(Object) get dispatch;

  ApplicationBlocState withDispatcher(Function(E) dispatcher);
}

abstract class ApplicationBlocEvent {}

class StateReducer<E extends ApplicationBlocEvent,
    S extends ApplicationBlocState> {
  StateReducer(this.handler) : assert(handler != null);

  final Stream<S> Function(E) handler;

  Stream<S> call(E event) => handler(event);
}

abstract class ApplicationBloc<E extends ApplicationBlocEvent,
    S extends ApplicationBlocState> extends Bloc<E, S> {
  AppLogger _log;

  ApplicationBloc(S initialState) : super(initialState);

  bool get enableLogging => false;

  AppLogger get log {
    _log ??= AppLogger(tag: '$runtimeType');
    return _log;
  }

  @override
  void onTransition(Transition<E, S> transition) {
    log.i(transition.currentState);
    super.onTransition(transition);
  }

  @override
  void onError(Object error, StackTrace stacktrace) {
    super.onError(error, stacktrace);
    print('$error\n$stacktrace');
  }

  @protected
  Map<Type, StateReducer<E, S>> get handlers;

  @override
  @protected
  S get state => super.state.withDispatcher(add);

  @protected
  S setLoading(bool isLoading);

  @override
  Stream<S> mapEventToState(E event) => stateFromHandlers(event);

  @protected
  Stream<S> stateFromHandlers(E event) {
    return _stateFromHandlers(event).asBroadcastStream();
  }

  Stream<S> _stateFromHandlers(E event) async* {
    log.i('Started: $event');
    yield setLoading(true);
    final StateReducer<E, S> reducer = handlers[event.runtimeType];
    if (reducer != null) {
      yield* reducer(event);
    } else {
      log.e('No handler defined for $event in $runtimeType');
    }
    yield setLoading(false);
    log.i('Completed: $runtimeType');
  }
}

```

Now that we have our ApplicationBloc configured and ready to go, let us start with our CounterBloc. First we need to configure the State of the CounterBloc. The `CounterState` will implement the `CounterViewModel` class which consists of all the details which will be required in the views. This way, we are providing our views with the latest values of our state using the ViewModel. Let's implement our state : 

`counter_state.dart`

```dart
import 'package:built_value/built_value.dart';
import 'package:counter_using_bloc/bloc/counter_event.dart';
import 'package:counter_using_bloc/connector/counter_connector.dart';

import 'application_bloc_delegate/application_bloc.dart';

part 'counter_state.g.dart';

abstract class CounterState
    implements
        Built<CounterState, CounterStateBuilder>,
        ApplicationBlocState<CounterEvent>,
        CounterViewModel {
  CounterState._();

  factory CounterState([void Function(CounterStateBuilder) updates]) =
      _$CounterState;

  static CounterState initState() => CounterState((CounterStateBuilder b) {
        b
          ..counterValue = 0
          ..loading = false;
      });

  CounterState withDispatcher(Function(CounterEvent) dispatcher) {
    assert(dispatcher != null);
    return rebuild((b) => b.dispatch = dispatcher);
  }

  void increment() => dispatch(IncrementCounterEvent());

  void reset() => dispatch(ResetCounterEvent());
}

```

States are immutable, meaning that we should not be able to modify the contents of a State in between transitions. This will result in undefined states and cause serious errors or loss of data. In order to avoid this, we use `BuiltType` for states so that each state will be immutable once that is reached.

Let's move on to the `CounterEvent`. We have two Events which can happen - update value and reset value. So we can have two Events for the same.

`counter_event.dart`

```dart
import 'package:counter_using_bloc/bloc/application_bloc_delegate/application_bloc_event.dart';

abstract class CounterEvent extends ApplicationBlocEvent {}

class IncrementCounterEvent implements CounterEvent {}

class ResetCounterEvent implements CounterEvent {}
```

Now that we have our states and events configured, let's move on to the `CounterBloc`. We will have two methods in our bloc to implement the two events.

`counter_bloc.dart`

```dart
import 'package:counter_using_bloc/bloc/counter_event.dart';
import 'package:counter_using_bloc/bloc/counter_state.dart';
import 'package:counter_using_bloc/data/data_service.dart';
import 'package:flutter/material.dart';

import 'application_bloc_delegate/application_bloc.dart';

class CounterBloc extends ApplicationBloc<CounterEvent, CounterState> {
  CounterBloc({
    @required DataService dataService,
  })  : assert(dataService != null),
        _dataService = dataService,
        super(CounterState.initState());

  final DataService _dataService;

  @override
  void onError(Object error, StackTrace stacktrace) {
    super.onError(error, stacktrace);
  }

  @override
  Map<Type, StateReducer<CounterEvent, CounterState>> get handlers =>
      <Type, StateReducer<CounterEvent, CounterState>>{
        IncrementCounterEvent:
            StateReducer<IncrementCounterEvent, CounterState>(
          _incrementCounter,
        ),
        ResetCounterEvent: StateReducer<ResetCounterEvent, CounterState>(
          _resetCounter,
        ),
      };

  @override
  CounterState setLoading(bool isLoading) {
    return (state.toBuilder()..loading = isLoading).build();
  }

  Stream<CounterState> _incrementCounter(IncrementCounterEvent event) async* {
    yield (state.toBuilder()
          ..counterValue =
              _dataService.incrementData(currValue: state.counterValue))
        .build();
  }

  Stream<CounterState> _resetCounter(ResetCounterEvent event) async* {
    yield (state.toBuilder()..counterValue = _dataService.resetData()).build();
  }
}

```

That's it. The final Step in the Business LogicLayer is to implement the connectors for this bloc. Connectors are basically a bridge between the ViewModels and Views where the ViewModel is immutable. The ViewModel consists of all the state variable getters and dispatch requests to perform the events in the bloc. The connectors are used to map these events and compare between two different viewModels: pre-event and post-event. More importantly, these connectors can be used as widgets from views to gain access to the viewModel. Let us implement our CounterConnector as follows:

`counter_connector.dart`

```dart

import 'package:counter_using_bloc/bloc/counter_bloc.dart';
import 'package:counter_using_bloc/bloc/counter_state.dart';
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

class CounterConnector extends StatelessWidget {
  const CounterConnector({Key key, this.builder, this.condition})
      : super(key: key);
  final BlocWidgetBuilder<CounterViewModel> builder;
  final BlocBuilderCondition<CounterViewModel> condition;

  @override
  Widget build(BuildContext context) {
    final CounterBloc _bloc = BlocProvider.of<CounterBloc>(context);

    return BlocBuilder<CounterBloc, CounterState>(
      cubit: _bloc,
      buildWhen: condition,
      builder: builder,
    );
  }
}

```

### 3. Presentation Layer

The presentation layer consists of the views and viewModels. ViewModel is just an abstract definiton of all the required data which will be implement by the state of the particular bloc and the most recent values will be maintained. These viewModels can then be accessed from our views to update the view based on state changes. Let us configure our viewModels:

`connector_view_model.dart`

```dart
abstract class CounterViewModel {
  bool get loading;

  int get counterValue;

  void increment();

  void reset();
}
```

Our viewModel consists of all the details which might be required in the view about our data and the state. We can perform any operations on the state or the data by dispatching requests from views using the methods defined in the viewModel.

Let's visualize the functioning using views:

`main.dart`

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
      home: BlocProvider<CounterBloc>(
        create: (BuildContext context) => CounterBloc(
          dataService: LocalService(),
        ),
        child: MyHomePage(title: 'Flutter Demo Home Page'),
      ),
    );
  }
}

class MyHomePage extends StatefulWidget {
  MyHomePage({Key key, this.title}) : super(key: key);

  final String title;

  @override
  _MyHomePageState createState() => _MyHomePageState();
}

class _MyHomePageState extends State<MyHomePage> {
  @override
  Widget build(BuildContext context) {
    return CounterConnector(
      builder: (BuildContext context, CounterViewModel viewModel) {
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
                  '${viewModel.counterValue}',
                  style: Theme.of(context).textTheme.headline4,
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
                  viewModel.increment();
                },
                tooltip: 'Increment',
                child: Icon(Icons.add),
              ),
              FloatingActionButton(
                onPressed: viewModel.reset,
                tooltip: 'Reset',
                child: Icon(Icons.close),
              ),
            ],
          ),
        );
      },
    );
  }
}
```

For more details on flutter_bloc, [visit the official documentation](https://bloclibrary.dev/#/)
