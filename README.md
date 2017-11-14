# Xcode VIPER Templates
[The VIPER-Templates Wiki has tons of information](../../wiki) to get you started.

VIPER is a 6 tier architecture that abstracts module tasks into each tier such that everything has a single responsibility. Its conforms to SOLID design principles and is an implementation of Clean Architecture concepts.

These templates are written in Swift for use with Xcode.

Head over to the [VIPER-Templates Wiki](../../wiki) for some in-depth information on VIPER and using these templates.

[Did I mention there is a VIPER-Templates Wiki](../../wiki), you should check it out!

# TODO

- Keep it sexy
- Write "Creating a VIPER Stack"
- Write script to generate README from wiki pages (currently manual process)

# Contents

* [VIPER](../../wiki/VIPER) - An overview of VIPER
* (NOT IMPLEMENTED) [Creating a VIPER Stack](../../wiki/Creating-a-VIPER-Stack) - step by step process of creating an example VIPER stack
* The VIPER Layers Explained
  * [Wireframe](../../wiki/Wireframe) - instantiation and navigation
  * [Presenter](../../wiki/Presenter) - business logic
  * [View](../../wiki/View) - user interface
  * [Interactor](../../wiki/Interactor) - data logic
  * [Services](../../wiki/Services) - web endpoint interactions
  * [Entities](../../wiki/Entities) - data objects
* [Installing](../../wiki/Installing) - how to install the templates in less than a minute
* [Installation Troubleshooting](../../wiki/Installation-Troubleshooting) - a few possible problems and solutions with installing the templates
* [Updating](../../wiki/Updating) - how to update with new changes to the templates
* [Using the Templates](../../wiki/Using-the-Templates) - how to use the templates to create your VIPER stacks
* [Organizing the Stack](../../wiki/Organizing-the-Stack) - one way of organizing the VIPER files in an Xcode project
* [What are These?](../../wiki/What-are-These?) - a general description of the files created by the templates
* [Mocking Made Easy](../../wiki/Mocking-Made-Easy) - short guide showing you how to use the mock files for testing
* [Alternative Resources](../../wiki/Alternative-Resources) - some other perspectives and implementations of VIPER

# VIPER

iOS is still a very young technology and as you may have found out an MVC architecture is anything but useful for long running applications. The fuboTV iOS application is adopting the VIPER architecture throughout.

## What VIPER Is
VIPER is a 6 tier architecture that abstracts module tasks into each tier such that everything has a single responsibility. Its conforms to SOLID design principles and is an implementation of Clean Architecture concepts.

Main Goals Of VIPER:
  - Make code easy to iterate on
  - Make projects collaboration-friendly
  - Create reusable modules with separated concerns
  - Make code easy to test

A very simplistic representation would be:
  - [Wireframe (Router) - instantiation and navigation](../../wiki/Wireframe)
  - [Presenter - business logic](../../wiki/Presenter)
  - [Interactor - data logic](../../wiki/Interactor)
  - [View - user interface](../../wiki/View)
  - [Service - retrieves entities](../../wiki/Service)
  - [Entity - data object](../../wiki/Entities)

![A VIPER Stack](../../wiki/images/viper_arch_picture.png)

Most connections are two way and each direction is abstracted into an interface. For example:
  - Wireframe talks to the Presenter through the WireframeToPresenterInterface
  - Presenter talks to the Wireframe through the PresenterToWireframeInterface

The abstracted interfaces provide us with a way to easily conform to the interface (Goal 2 & 3). This lets us easily replace objects by creating new objects that only need to implement the interfaces (Goal 1 and 3). This also lets us create mock objects that can easily be injected for testing (Goal 4).

## What VIPER Is NOT
VIPER is not the end all of architectures. It solves many problems that arise from MVC, but sometimes the technology of our IDE's doesn't mix well (ex: storyboards and segues). If you see a way to improve VIPER, please be vocal, we want it to improve!

# Wireframe
The wireframe is responsible for instantiation and navigation. It is the interface into the VIPER stack. When creating a VIPER stack, you instantiate the wireframe and it instantiates all the other layers and connects them properly. A wireframe constructor could look like this:

```swift
lazy var moduleInteractor = Interactor()
lazy var modulePresenter = Presenter()
lazy var moduleView = View()

init() {
    super.init()

    let i = moduleInteractor
    let p = modulePresenter
    let v = moduleView

    i.presenter = p

    p.interactor = i
    p.view = v
    p.wireframe = self

    v.presenter = p
}
```
The wireframe maintains strong references to module layers so they do not get deallocated. It initializes all of them, and connects each one to the other, correctly.

With navigation, its specifically navigation to the stack or away from the stack. Lets say you want to display a login stack from the home stack of the application.
```swift
//HomeWireframe.swift
lazy loginModule: Login = LoginWireframe()
func presentLogin() {
    loginModule.present(onViewController: moduleView)
}

//LoginWireframe.swift
func present(onViewController viewController: UIViewController) {
    viewController.present(moduleView, animated: true)
    // Here, you could also notify the presenter that the stack 
    //    began presenting, but for login, there is no initial setup 
    //    for this to be needed since the text fields will be empty
    // presenter.beganPresenting()
}
```
The home wireframe has been told to present the login stack (from the `HomePresenter`). This function calls the login's wireframe, which implements the module's interface protocol, presentation method. The login's wireframe implements the presentation method by telling the view controller that is passed in to present the login module's view controller.

# Presenter

The `Presenter` is where business logic lives. It is what drives all the other layers, making decisions based on events that happen in the other layers. Think of the `Presenter` like a manager, it knows what needs to happen to get certain task done, it knows who is best to do the job, and it tells those in its employ to do them, but doesn`t do any of the heavy lifting itself.

It has outlets to the other components of the VIPER stack, something like this:
```swift
weak var delegate: Delegate?
weak var interactor: PresenterToInteractorInterface!
weak var view: PresenterToViewInterface!
weak var wireframe: PresenterToWireframeInterface!
var moduleWireframe: Login {
     get {
         return self.wireframe as! Login
     }
}
```

#### Communicating with the Interactor
Lets take a look at a typical flow. Lets say your user wants to login to the application, so they have entered their username and password in the `View`, and now they tap the `Login` button.
```swift
//View.swift
@IBAction func loginTapped(sender: AnyObject) {
    let username = usernameTextField.text
    let password = passwordTextField.text
    presenter.userTappedLogin(withUsername: username, andPassword: password)
}

//Presenter.swift
func userTappedLogin(withUsername username: String, andPassword password: String) {
     interactor.login(withUsername: username, andPassword: password)
}
```
The [[View]] will tell the `Presenter` of the user event, and pass the related information. When the `Presenter` gets it, it will tell the [[Interactor]] that it needs to call a service to login the user with the username and password the user entered.

#### Communicating with the Delegate
Lets say the call to the login service succeeded, and the module now needs to tell the `Delegate` the user has been logged in.
```swift
//Interactor.swift
func loggedIn(withUser user: User) {
    presenter.loginSucceeded()
}

//Presenter.swift
func loginSucceeded() {
    delegate?.loggedIn(login: moduleWireframe)
}
```
The [[Interactor]] will tell the `Presenter` of the success, and the presenter decides to tell the `Delegate` that login succeeded.

#### Communicating with the View
What if the login failed? Maybe the username doesn't exist, or the password was incorrect.
```swift
//Interactor.swift
func failedLogin(withError error: Error) {
    presenter.loginFailed(withError: error)
}

//Presenter.swift
func loginFailed(withError error: Error) {
    view.displayLoginError(withDescription: error.description)
}
```
The [[Interactor]] will tell the `Presenter` that login failed, and pass the error along. The presenter decides to tell the [[View]] to display a login error with the description received from backend. The [[View]] can then decide how it displays said error, maybe with an alert, or just a label, what ever it wants to do.

#### Communicating with the Wireframe
Maybe the user forgot their password and the reset password module needs to be presented.
```swift
//View.swift
@IBAction func resetPasswordTapped(sender: AnyObject) {
    presenter.userTappedResetPassword()
}

//Presenter.swift
func userTappedResetPassword() {
    wireframe.presentResetPassword()
}
```
Here, the user event is reported from the [[View]] to the `Presenter`, since there is navigation away from the login stack, to the reset password stack, the [[Wireframe]] needs to be notified. The `Presenter` tells the [[Wireframe]] to present that module, however it needs to.

# View

An `View` is responsible for the user interface. It is the layer that retrieves information and events from the user and relates that to the [[Presenter]].

It has outlets only to the [[Presenter]] of the VIPER stack, something like this:
```swift
weak var presenter: ViewToPresenterInterface!
```

It is important to understand that the `View` is dumb, it does not drive interactions of any kind. This is typically a very difficult concept for people new to VIPER, as with MVC, we are used to responding to `View` events like `viewDidLoad` or `viewDidAppear`. In VIPER, these events are handled by the [[Presenter]], and the [[Presenter]] is what tells the `View` what to do.

#### Being a Reactive View
The `View` in a VIPER stack is reactive, not proactive. It only updates the UI in response to a command from the [[Presenter]]. This is important to understand as this is what causes the UI to be independent of data flow and easily changed. Lets say there is a jogging application, and the module has been told to present a screen that shows all the users jogging sessions. Somehow, the [[Presenter]] is told that some jogs were fetched.
```swift
//Presenter.swift
func fetchedJogs(jogs: [Jog]) {
    view.display(jogs: jogs)
}

//View.swift
func display(jogs newJogs: [Jog]) {
     jogs = newJogs
     tableView.reloadData()
}
```
When the [[Presenter]] receives jogs in someway, it then knows it needs to tell the `View` to display them, so it calls the `display(jogs:)` method on the `View`. This particular `View` uses a `UITableView` to display the jogs, so it just saves the jogs and tells the `tableView` to reload its data.

What if you wanted to change this implementation to use a `UICollectionView`? The `display(jogs:)` function would stay the same, and the [[Presenter]]/[[Interactor]]/[[Wireframe]] would never need to be touched. You could create a new `View` object that conforms to the same `PresenterToViewInterface`, but this one uses a `UICollectionView` implementation. Then this new `View` is just dropped into the place and you're all done!

#### Using View Objects
A big key of the [[VIPER]] architecture is being able to easily change layers without them affecting others. So what if we changed the `Jog` object to something like a `Run` object? Consequently, we would need to change all the layers of the [[VIPER]] stack to use this new `Run` object interface. What would be a better way?

We could create a data object that is specifically for this `View` layer that has only the fields we require to display. Lets say this `View` only needs to display the distance, date, and time of the `Jog`.
```swift
//Jog.swift
class Jog {
    var date: Date?
    var distance: Double?
    var location: Location?
    var time: Int?
    var user: User?
}

//ViewObject.swift
class ViewObject {
    var date: Date?
    var distance: Double?
    var time: Int?
    
    // Initializers
    init(fromJog jog: Jog) {
        date = jog.date
        distance = jog.distance
        time = jog.time
    }
}

//View.swift
func display(viewObjects newViewObjects: [ViewObject]) {
    viewObjects = newViewObjects
    tableView.reloadData()
}
```
Easy enough, right? Ok, now the backend starts returning `Run` objects. All we need to do is make the `ViewObject` have an `init(fromRun:)`
```swift
//Run.swift
class Run {
    var date: Date?
    var distance: Double?
    var endTime: Date?
    var location: Location?
    var startTime: Date?
    var user: User?
}

//ViewObject.swift
class ViewObject {
    var date: Date?
    var distance: Double?
    var time: Int?
    
    // Initializers
    init(fromJog jog: Jog) {
        // fromJog implementation
    }
    
    init(fromRun run: Run) {
        date = run.date
        distance = run.distance
        time = run.endTime - run.startTime
    }
}
```
All done! The `View` can keep using the same `ViewObject` to display the UI, and nothing needs to be changed on the `View` layer to handle this new data type.

#### Communicating with a Presenter
Ok, now lets say your user wants to login to the application, so the `View` is displaying two text fields, one for username entry, and the other for password. The user types in their username and password, then presses a `Login` button.
```swift
//View.swift
@IBAction func loginTapped(sender: AnyObject) {
    let username = usernameTextField.text
    let password = passwordTextField.text
    presenter.userTappedLogin(withUsername: username, andPassword: password)
}

//Presenter.swift
func userTappedLogin(withUsername username: String, andPassword password: String) {
     interactor.login(withUsername: username, andPassword: password)
}
```
Here, the `View` tells the [[Presenter]] of the user event, and communicates the information that it gathered (username and password). The [[Presenter]] then decides what to do with the user event. Notice this flow isn't initiating a login call to the backend. It is just notifying the [[Presenter]] of the user event.

# Interactor

An `Interactor` is responsible for data logic. It is the layer that retrieves the data from any source it needs to. For instance, it can communicate with a [Service](Services) or a local data store such as CoreData.

It has outlets only to the [[Presenter]] of the VIPER stack, something like this:
```swift
weak var presenter: InteractorToPresenterInterface!
```

#### Communicating with a Service
Lets take a look at a typical flow. Lets say your user wants to login to the application, so the presenter has been told the user wants to login with their username and password:
```swift
//Presenter.swift
func userTappedLogin(withUsername username: String, andPassword password: String) {
     interactor.login(withUsername: username, andPassword: password)
}

//Interactor.swift
lazy var loginService: LoginService = LoginService()
func login(withUserName username: String, andPassword password: String) {
    loginService.login(withUsername: username, andPassword: password,
        success: { (user: User) in
             self.loggedIn(withUser: user)
        },
        failure: { (error: Error) in
             self.failedLogin(withError: error)
        })
}
```
Here, the `Interactor` is told to login with the username and password by the [[Presenter]]. The `Interactor` knows that it needs to make a call to the `loginService` to login the user and get the `user` object from the web service. It can then implement the `success` and `failure` completion blocks as it needs to.

#### Communicating with the Presenter
So the `loginService` succeeded, and the success block of the [Service](Services) is ran, and now this information needs to be conveyed to the [[Presenter]].
```swift
//Interactor.swift
func loggedIn(withUser: user) {
     presenter.logginSucceeded()
}
```

#### Communicating with a DataStore
So what if instead of calling a [Service](Services), you instead want to get information from some sort of data manager like Realm? Lets say we are making a jogging application that records the user's jogging sessions.
```swift
//Presenter.swift
func beganPresenting() {
    interactor.fetchJogs()
}

//Interactor.swift
func fetchJogs() {
    let realm = try! Realm()
    let allJogs = realm.objects(Jog.self)
    presenter.fetchedJogs(Array(allJogs))
}
```
Notice the interface to the `Interactor` from the [[Presenter]] is the same as if the `Interactor` was going to call a [Service](Services). The [[Presenter]] has no idea how the `Interactor` fetches jogs (or logs in the user). The `Interactor` is responsible for this interaction.

# Services


A `Service` is a modularized object that typically connects to a web interface to get data. This allows the connection to a web endpoint to be completely abstracted and easily changeable. The [[Interactor]] will call the `Services` and handle the responses, whether it is a success or failure.

Continuing with the Login flow example, lets say the [[Interactor]] has been told to login with a username and password.
```swift
//Interactor.swift
lazy var loginService: LoginService = LoginService()
func login(withUserName username: String, andPassword password: String) {
    loginService.login(withUsername: username, andPassword: password,
        success: { (user: User) in
             self.loggedIn(withUser: user)
        },
        failure: { (error: Error) in
             self.failedLogin(withError: error)
        })
}

//LoginService.swift
func login(withUsername username: String,
           andPassword password: String,
           success completion: (user: User),
           failure: ((error: Error) -> Void)) {
    let parameters: Parameters = [
        "username": username,
        "password": password,
    ]
    let request = Alamofire.request("http://www.myserver.com/login,
                      method: Method.get,
                      parameters: parameters,
                      encoding: JSONEncoding.default)
    request.responseJSON { (response: Response) in
         switch response.result {
             case .Success(let value):
                 let user = User(fromJson: value)
                 completion(user: user)
             case .Failure(let error):
                 failure(error: error)
         }
    }
}
```
Here, we use [Alamofire](https://github.com/Alamofire/Alamofire) to handle the HTTP request to our server and login the user with username and password provided. The response is JSON, that is then parsed into a `User` object by the `User` entities init from JSON method. It is then returned to the completion handler block.

Similarly, if there was an error (maybe the username or password was incorrect), the failure block is called with the `Error` that was received.

So, with this implementation, notice that we can easily swap out [Alamofire](https://github.com/Alamofire/Alamofire) for any other networking API. The [[Interactor]] will still call the login service the same way, and receive the `User` or `Error` object through the same handler blocks, allowing this implementation to change on the fly without affecting any other part of our application!

#Entities

`Entities` are data objects that we use throughout the application. They can be used anywhere, and are typically created by [[Services]]. They can be passed around any of the [[VIPER]] layers and used as needed.

Lets take a look a typical `User` entity
```swift
//User.swift
class User {
    // Identifier
    let userId: Int

    // Instance Variables
    let gender: String?
    let password: String?
    let username: String?
    
    // Initializers
    init(withUserId newUserId: Int) {
         userId = newUserId 
    }
    
    init?(fromJson json: [String: AnyObject]) {
        let json = JSON(jsonDictionary)
        let identifier = json["userId"].int

        guard identifier != nil else {
            return nil
        }

        self.init(withUserId: identifier)

        gender = json["gender"].string
        password = json["password"].string
        username = json["username"].string
    }
}
```
Here, we have a basic `User` object. In the `init(withJson:)` method we use [SwiftyJSON](https://github.com/SwiftyJSON/SwiftyJSON) to easily parse the JSON's values into the objects instance variables.

