## Flavors used
- Dev (main-dev)
- Staging (main-staging)
- Prod (main-prod)

## Installation

 Add These to  pubspec.yaml as a dependency
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
## To get Build
### Android 
Dev - `flutter build appbundle --flavor dev -t lib/main-dev.dart`
Staging - `flutter build appbundle --flavor staging -t lib/main-staging.dart`
Prod - `flutter build appbundle --flavor prod -t lib/main-prod.dart`  

#### APK
Dev - `flutter build apk --flavor dev -t lib/main-dev.dart`
Staging - `flutter build apk --flavor staging -t lib/main-staging.dart`
Prod - `flutter build apk --flavor prod -t lib/main-prod.dart`

### IOS 
Dev - `flutter build ipa --flavor dev -t lib/main-dev.dart`
Staging - `flutter build ipa --flavor staging -t lib/main-staging.dart`
Prod - `flutter build ipa --flavor prod -t lib/main-prod.dart`


