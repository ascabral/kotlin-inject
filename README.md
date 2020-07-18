# kotlin-inject
[![CircleCI](https://circleci.com/gh/evant/kotlin-inject.svg?style=svg&circle-token=8792fa19911be92d6a1d66dd45ece3bf6712f778)](https://circleci.com/gh/evant/kotlin-inject)

A compile-time dependency injection library for kotlin.

```kotlin
@Component abstract class AppComponent {
    abstract val repo: Repository
    
    @Provides protected fun jsonParser() = JsonParser()

    @Provides protected val RealHttp.http: Http get() = this
}

interface Http
@Inject class RealHttp
@Inject class Api(private val http: Http, private val jsonParser: JsonParser)
@Inject class Repository(private val api: Api)
```
```kotlin
val appComponent = AppComponent::class.create()
val repo = appComponent.repo
```

## Download

```groovy
plugins {
    id 'org.jetbrains.kotlin.jvm'
    id 'org.jetbrains.kotlin.kapt'
}

repositories {
  maven { url "https://oss.sonatype.org/content/repositories/snapshots/" }
}

dependencies {
    kapt "me.tatarka.inject:kotlin-inject-compiler-kapt:0.0.1-SNAPSHOT"
    implementation "me.tatarka.inject:kotlin-inject-runtime:0.0.1-SNAPSHOT"
}
```

### Expiramental KSP Support 

You can use [ksp](https://github.com/android/kotlin/tree/ksp/libraries/tools/kotlin-symbol-processing-api) instead of 
kapt. Currently everything except type aliases and function injection is implemented.

`settings.gradle`
```groovy
pluginManagement {
    resolutionStrategy {
        eachPlugin {
            switch (requested.id.id) {
                case "kotlin-ksp":
                case "org.jetbrains.kotlin.kotlin-ksp":
                case "org.jetbrains.kotlin.ksp":
                    useModule("org.jetbrains.kotlin:kotlin-ksp:${requested.version}")
            }
        }
    }

    repositories {
            gradlePluginPortal()
            maven { url "https://dl.bintray.com/kotlin/kotlin-eap" }
            google()
    }
}
```

`build.gradle`
```groovy
plugins {
    id 'org.jetbrains.kotlin.jvm' version "1.4-M1"
    id 'org.jetbrains.kotlin.ksp' version "1.4-M1-dev-experimental-20200716"
}

repositories {
    mavenCentral()
    maven { url "https://dl.bintray.com/kotlin/kotlin-eap" }
    google()
    maven { url "https://oss.sonatype.org/content/repositories/snapshots/" }
}

dependencies {
    ksp "me.tatarka.inject:kotlin-inject-compiler-ksp:0.0.1-SNAPSHOT"
    implementation "me.tatarka.inject:kotlin-inject-runtime:0.0.1-SNAPSHOT"
}
```

## Usage

Let's go through the above example line-by line and see what it's doing.

```kotlin
@Component abstract class AppComponent {
```
The building block of kotlin-inject is a component which you declare with an `@Component` annotation on an abstract class. An
implementation of this component will be generated for you.

```kotlin
    abstract val repo: Repository
```
In you component you can declare abstract read-only properties or functions to return an instance of a given type. This is
where the magic happens. kotlin-inject will figure out how to construct that type for you in it's generated
implementation. How does it know how to do this? There's a few ways:

```kotlin
    @Provides protected fun jsonParser() = JsonParser()
```
For external dependencies, you can declare a function or read-only property in the component to create an instance for a 
certain type. kotlin-inject will use the return type to provide this instance where it is requested.

```kotlin
    @Provides protected val RealHttp.http: Http get() = this
```
You can declare arguments to a providing function/property to help you construct your instance. Here we are taking in an
instance of `RealHttp` and providing it for the interface `Http`. You can see a little sugar with this as the receiver 
type for an extension function/property counts as an argument. Another way to write this would be:
`fun provides(http: RealHttp): Http = http`.

```kotlin
@Inject class RealHttp
@Inject class Api(private val http: Http, private val jsonParser: JsonParser)
@Inject class Repository(private val api: Api)
```
For your own dependencies you can simply annotate the class with `@Inject`. This will use the primary constructor to
create an instance, no other configuration required!

```kotlin
val appComponent = AppComponent::class.create()
val repo = appComponent.repo
```

Finally, you can create an instance of your component with the generated `.create()` extension function.

## Features

### Component Arguments

If you need to pass any instances into your component you can declare them as constructor args. You can then pass them into
the generated create function. You can optionally annotate it with `@Provides` to provide the value to the dependency graph.

```kotlin
@Component abstract class MyComponent(@Provides protected val foo: Foo)
```

```kotlin
MyComponent::class.create(Foo())
```

If the argument is another component, you can annotate it with `@Component` and it's dependencies will also be available to the 
child component. This allows you to compose them into a graph.

```kotlin
@Component abstract class ParentComponent {
    protected fun provideFoo(): Foo = ...
}

@Component abstract class ChildComponent(@Component val parent: ParentComponent) {
    abstract val foo: Foo
}
```

```kotlin
val parent = ParentComponent::class.create()
val child = ChildComponent::class.create(parent)
```

### Type Alias Support

If you have multiple instances of the same type you want to differentiate, you can use type aliases. They will be 
treated as separate types for the purposes of injection.

```kotlin
typealias Dep1 = Dep
typealias Dep2 = Dep

@Component abstract class MyComponent {
    @Provides fun dep1(): Dep1 = Dep("one")
    @Provides fun dep2(): Dep2 = Dep("two")

    protected fun provides(dep1: Dep1, dep2: Dep1) = Thing(dep1, dep2)
}

@Inject class InjectedClass(dep1: Dep1, dep2: Dep2)
```

### Function Injection

You can also use type aliases to inject into top-level functions. Annotate your function with `@Inject` and create a 
type alias with the same name.

```kotlin
typealias myFunction = () -> Unit

@Inject fun myFunction(dep: Dep) {
}
```

You can then use the type alias anywhere and you will be provided with a function that calls the top-level one with the
requested dependencies.

```kotlin
@Inject class MyClass(val myFunction: myFunction)

@Component abstract class MyComponent {
    abstract val myFunction: myFunction
}
```

You can optionally pass explicit args as the last arguments of the function.

```kotlin
typealias myFunction = (String) -> String

@Inject fun myFunction(dep: Dep, arg: String): String = ...
```

### Scopes

By default kotlin-inject will create a new instance of a dependency each place it's injected. If you want to re-use an
instance you can scope it to a component. The instance will live as long as that component does.

First create your scope annotation.
```kotlin
@Scope annotation class MyScope
```

Then annotate your component with that scope annotation.

```kotlin
@MyScope @Component abstract class MyComponent()
```

Finally, annotate your provides and `@Inject` classes with that scope.

```kotlin
@MyScope @Component abstract class MyComponent {
    @MyScope @Provides
    protected fun provideFoo() = ...
}

@MyScope @Inject class Bar()
```

### Component Inheritance

You can define `@Provides` and scope annotations on an interface or abstract class that's not annotated with `@Component`.
This allows you to have multiple implementations, which is useful for things like testing. For example, you can have an
abstract class like

```kotlin
@NetworkScope abstract class NetworkComponent {
    @NetworkScope @Provides
    abstract fun api(): Api 
}
```

Then you can have multiple implementations

```kotlin
@Component abstract class RealNetworkComponent : NetworkComponent() {
    override fun api(): Api = RealApi()
}

@Component abstract class TestNetworkComponent : NetworkComponent() {
    override fun api(): Api = FakeApi()
}
```

Then you can provide the abstract class to your app component

```
@Component abtract class AppComponent(@Component val network: NetworkComponent)
```

Then in your app you can do

```kotlin
AppComponent::class.create(RealNetworkComponent::class.create())
```

an in tests you can do

```kotlin
AppComponent::class.create(TestNetworkComponent::class.create())
```

### Multi-bindings

You can collect multiple bindings into a `Map` or `Set` by using the `@IntoMap` and `@IntoSet` annotations respectively.

For a set, return the type you want to put into a set, then you can inject or provide a `Set<MyType>`.

```kotlin
@Component abstract class MyComponent {
    abstract val allFoos: Set<Foo>

    @IntoSet @Provides protected fun provideFoo1(): Foo = Foo("1")
    @IntoSet @Provdies protected fun provideFoo2(): Foo = Foo("2")
}
```

For a map, return a `Pair<Key, Value>`.

```kotlin
@Component abstract class MyComponent {
    abstract val fooMap: Map<String, Foo>
    
    @IntoMap @Provides protected fun provideFoo1(): Pair<String, Foo> = "1" to Foo("1")
    @IntoMap @Provides protected fun provideFoo2(): Pair<String, Foo> = "2" to Foo("2")
}
```

### Function Support

Sometimes you want to delay the creation of a dependency or provide additional params manually. You can do this by 
injecting a function that returns the dependency instead of the dependency directly.

The simplest case is you take no args, this gives you a function that can create the dep.

```kotlin
@Inject class Foo

@Inject class MyClass(fooCreator: () -> Foo) {
    init {
        val foo = fooCreator()
    }
}
```

If you define args, you can use these to assist the creation of the dependency. These are passed in as the _last_ 
arguments to the dependency.

```kotlin
@Inject class Foo(bar: Bar, arg1: String, arg2: String)

@Inject class MyClass(fooCreator: (arg1: String, arg2: String) -> Foo) {
    init {
        val foo = fooCreator("1", "2")
    }
}
```

### Lazy

Similarly, you can inject a `Lazy<MyType>` to construct and re-use and instance lazily.

```kotlin
@Inject class Foo

@Inject class MyClass(lazyFoo: Lazy<Foo>) {
    val foo by lazyFoo
}
```
