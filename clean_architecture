# Flutter App/Module Architecture
The document focuses on the architectural patterns and best practices to use with Flutter Projects. 

The document only focuses on implementation and usage patterns. To know more about Clean Architecture, [here](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) is a nice write up.

A good architecture allows you to decouple units of code and improve maintainability of code. But at the same time, a very complex architecture might also result in having opposite effect. The idea of Clean Architecture is to keep things simple where it focuses on separation of code based on layers and enforces for outer layers to depend on the inners and the vice-versa not possible.

BLOCK DIAGRAM:

![Image](./app_arch_block.png?raw=true)

### Presentation/UI layer: 
This is the layer that allows to interact with the User. This layer usually consists of UI/UX and interactions elements that allows a user to communicate with the app. Usually they go into the views and can only interact with the *connectors*.

### Use Cases/ Interactors: 
This layer also mostly couples with the Views, but despite that they are also responsible for app navigation, deeplinks and all other kinds of interations. Even they mostly communicate with only *Connectors*.

### Domain:
This layer usually forms the business logic, mostly the models and the Bloc. This layer has a wrapper called Connectors which abstracts out any direct access to this layer from views

### Data:
Data is responsible an abstract definition of different data sources the app has access to. Basically they form AppServices, ServiceImpl and API layers

 
