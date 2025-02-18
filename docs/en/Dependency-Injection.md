# Dependency Injection

ABP's Dependency Injection system is developed based on Microsoft's [dependency injection extension](https://medium.com/volosoft/asp-net-core-dependency-injection-best-practices-tips-tricks-c6e9c67f9d96) library (Microsoft.Extensions.DependencyInjection nuget package). So, its documentation is valid in ABP too.

> While ABP has no core dependency to any 3rd-party DI provider. However, it's required to use a provider that supports dynamic proxying and some other advanced features to make some ABP features properly work. Startup templates come with [Autofac](https://autofac.org/) installed. See [Autofac integration](Autofac-Integration.md) document for more information.

## Modularity

Since ABP is a modular framework, every module defines its own services and registers via dependency injection in its own separate [module class](Module-Development-Basics.md). Example:

````C#
public class BlogModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        //register dependencies here
    }
}
````

## Conventional Registration

ABP introduces conventional service registration. You need not do anything to register a service by convention. It's automatically done. If you want to disable it, you can set `SkipAutoServiceRegistration` to `true` in the constructor of your module class. Example:

````C#
public class BlogModule : AbpModule
{
    public BlogModule()
    {
        SkipAutoServiceRegistration = true;
    }
}
````

Once you skip the auto registration, you should manually register your services. In that case, ``AddAssemblyOf`` extension method can help you to register all your services by convention. Example:

````c#
public class BlogModule : AbpModule
{
    public BlogModule()
    {
        SkipAutoServiceRegistration = true;
    }

    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        context.Services.AddAssemblyOf<BlogModule>();
    }
}
````

The sections below explain the conventions and configurations.

### Inherently Registered Types

Some specific types are registered to dependency injection by default. Examples:

* Module classes are registered as singleton.
* MVC controllers (inherit ``Controller`` or ``AbpController``) are registered as transient.
* MVC page models (inherit ``PageModel`` or ``AbpPageModel``) are registered as transient.
* MVC view components (inherit ``ViewComponent`` or ``AbpViewComponent``) are registered as transient.
* Application services (inherit ``ApplicationService`` class or its subclasses) are registered as transient.
* Repositories (implement ``BasicRepositoryBase`` class or its subclasses) are registered as transient.
* Domain services (implement ``IDomainService`` interface or inherit ``DomainService`` class) are registered as transient.

Example:

````C#
public class BlogPostAppService : ApplicationService
{
}
````

``BlogPostAppService`` is automatically registered with transient lifetime since it's derived from a known base class.

### Dependency Interfaces

If you implement these interfaces, your class is registered to dependency injection automatically:

* ``ITransientDependency`` to register with transient lifetime.
* ``ISingletonDependency`` to register with singleton lifetime.
* ``IScopedDependency`` to register with scoped lifetime.

Example:

````C#
public class TaxCalculator : ITransientDependency
{
}
````

``TaxCalculator`` is automatically registered with a transient lifetime since it implements ``ITransientDependency``.

### Dependency Attribute

Another way of configuring a service for dependency injection is to use ``DependencyAttribute``. It has the following properties:

* ``Lifetime``: Lifetime of the registration: ``Singleton``, ``Transient`` or ``Scoped``.
* ``TryRegister``: Set ``true`` to register the service only if it's not registered before. Uses TryAdd... extension methods of IServiceCollection.
* ``ReplaceServices``: Set ``true`` to replace services if they are already registered before. Uses Replace extension method of IServiceCollection.

Example:

````C#
[Dependency(ServiceLifetime.Transient, ReplaceServices = true)]
public class TaxCalculator
{
}
````

``Dependency`` attribute has a higher priority than other dependency interfaces if it defines the ``Lifetime`` property.

### ExposeServices Attribute 

``ExposeServicesAttribute`` is used to control which services are provided by the related class. Example:

````C#
[ExposeServices(typeof(ITaxCalculator))]
public class TaxCalculator: ICalculator, ITaxCalculator, ICanCalculate, ITransientDependency
{
}
````

``TaxCalculator`` class only exposes ``ITaxCalculator`` interface. That means you can only inject ``ITaxCalculator``, but can not inject ``TaxCalculator`` or ``ICalculator`` in your application.

### Exposed Services by Convention

If you do not specify which services to expose, ABP expose services by convention. So taking the ``TaxCalculator`` defined above:

* The class itself is exposed by default. That means you can inject it by ``TaxCalculator`` class.
* Default interfaces are exposed by default. Default interfaces are determined by naming convention. In this example, ``ICalculator`` and ``ITaxCalculator`` are default interfaces of ``TaxCalculator``, but ``ICanCalculate`` is not. A generic interface (e.g. `ICalculator<string>`) is also considered as a default interface if the naming convention is satisfied.

### Combining All Together

Combining attributes and interfaces is possible as long as it's meaningful.

````C#
[Dependency(ReplaceServices = true)]
[ExposeServices(typeof(ITaxCalculator))]
public class TaxCalculator : ITaxCalculator, ITransientDependency
{

}
````

### ExposeKeyedService Attribute 

`ExposeKeyedServiceAttribute` is used to control which keyed services are provided by the related class. Example:

````C#
[ExposeKeyedService<ITaxCalculator>("taxCalculator")]
[ExposeKeyedService<ICalculator>("calculator")]
public class TaxCalculator: ICalculator, ITaxCalculator, ICanCalculate, ITransientDependency
{
}
````

In the example above, the `TaxCalculator` class exposes the `ITaxCalculator` interface with the key `taxCalculator` and the `ICalculator` interface with the key `calculator`. That means you can get keyed services from the `IServiceProvider` as shown below:

````C#
var taxCalculator = ServiceProvider.GetRequiredKeyedService<ITaxCalculator>("taxCalculator");
var calculator = ServiceProvider.GetRequiredKeyedService<ICalculator>("calculator");
````

Also, you can use the [`FromKeyedServicesAttribute`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.fromkeyedservicesattribute?view=dotnet-plat-ext-8.0) to resolve a certain keyed service in the constructor:

```csharp
public class MyClass
{
    //...

    public MyClass([FromKeyedServices("taxCalculator")] ITaxCalculator taxCalculator)
    {
        TaxCalculator = taxCalculator;
    }
}
```

> Notice that the `ExposeKeyedServiceAttribute` only exposes the keyed services. So, you can not inject the `ITaxCalculator` or `ICalculator` interfaces in your application without using the `FromKeyedServicesAttribute` as shown in the example above. If you want to expose both keyed and non-keyed services, you can use the `ExposeServicesAttribute` and `ExposeKeyedServiceAttribute` attributes together as shown below:

````C#
[ExposeKeyedService<ITaxCalculator>("taxCalculator")]
[ExposeKeyedService<ICalculator>("calculator")]
[ExposeServices(typeof(ITaxCalculator), typeof(ICalculator))]
public class TaxCalculator: ICalculator, ITaxCalculator, ICanCalculate, ITransientDependency
{
}
````

### Manually Registering

In some cases, you may need to register a service to the `IServiceCollection` manually, especially if you need to use custom factory methods or singleton instances. In that case, you can directly add services just as [Microsoft documentation](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection) describes. Example:

````C#
public class BlogModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        //Register an instance as singleton
        context.Services.AddSingleton<TaxCalculator>(new TaxCalculator(taxRatio: 0.18));

        //Register a factory method that resolves from IServiceProvider
        context.Services.AddScoped<ITaxCalculator>(
            sp => sp.GetRequiredService<TaxCalculator>()
        );
    }
}
````

### Replace a Service

If you need to replace an existing service (defined by the ABP framework or another module dependency), you have two options;

1. Use the `Dependency` attribute of the ABP framework as explained above.
2. Use the `IServiceCollection.Replace` method of the Microsoft Dependency Injection library. Example:

````csharp
public class MyModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        //Replacing the IConnectionStringResolver service
        context.Services.Replace(
            ServiceDescriptor.Transient<
            	IConnectionStringResolver,
                MyConnectionStringResolver
            >());
    }
}
````

## Injecting Dependencies

There are three common ways of using a service that has already been registered.

### Constructor Injection

This is the most common way of injecting a service into a class. For example:

````C#
public class TaxAppService : ApplicationService
{
    private readonly ITaxCalculator _taxCalculator;

    public TaxAppService(ITaxCalculator taxCalculator)
    {
        _taxCalculator = taxCalculator;
    }

    public async Task DoSomethingAsync()
    {
        //...use _taxCalculator...
    }
}
````

``TaxAppService`` gets ``ITaxCalculator`` in its constructor. The dependency injection system automatically provides the requested service at runtime.

Constructor injection is preffered way of injecting dependencies to a class. In that way, the class can not be constructed unless all constructor-injected dependencies are provided. Thus, the class explicitly declares it's required services.

### Property Injection

Property injection is not supported by Microsoft Dependency Injection library. However, ABP can integrate with 3rd-party DI providers ([Autofac](https://autofac.org/), for example) to make property injection possible. Example:

````C#
public class MyService : ITransientDependency
{
    public ILogger<MyService> Logger { get; set; }

    public MyService()
    {
        Logger = NullLogger<MyService>.Instance;
    }

    public async Task DoSomethingAsync()
    {
        //...use Logger to write logs...
    }
}
````

For a property-injection dependency, you declare a public property with public setter. This allows the DI framework to set it after creating your class.

Property injected dependencies are generally considered as **optional** dependencies. That means the service can properly work without them. ``Logger`` is such a dependency, ``MyService`` can continue to work without logging.

To make the dependency properly optional, we generally set a default/fallback value to the dependency. In this sample, NullLogger is used as fallback. Thus, ``MyService`` can work but does not write logs if DI framework or you don't set Logger property after creating ``MyService``.

One restriction of property injection is that you cannot use the dependency in your constructor, since it's set after the object construction.

Property injection is also useful when you want to design a base class that has some common services injected by default. If you're going to use constructor injection, all derived classes should also inject depended services into their own constructors which makes development harder. However, be very careful using property injection for non-optional services as it makes it harder to clearly see the requirements of a class.

#### DisablePropertyInjection Attribute

You can use `[DisablePropertyInjection]` attribute on classes or their properties to disable property injection for the whole class or some specific properties.

````C#
// Disabling for all properties of the MyService class
[DisablePropertyInjection]
public class MyService : ITransientDependency
{
    public ILogger<MyService> Logger { get; set; }
    
    public ITaxCalculator TaxCalculator { get; set; }
}

// Disabling only for the TaxCalculator property
public class MyService : ITransientDependency
{
    public ILogger<MyService> Logger { get; set; }

    [DisablePropertyInjection]
    public ITaxCalculator TaxCalculator { get; set; }
}
````

#### IInjectPropertiesService

You can use the `IInjectPropertiesService` service to inject properties of an object. Generally, it is a service outside of DI, such as manually created services.

````C#
var injectPropertiesService = serviceProvider.GetRequiredService<IInjectPropertiesService>();
var instance = new TestService();

// Set any properties on instance that can be resolved by IServiceProvider.
injectPropertiesService.InjectProperties(instance);

// Set any null-valued properties on instance that can be resolved by the IServiceProvider.
injectPropertiesService.InjectUnsetProperties(instance);
````

### Resolve Service from IServiceProvider

You may want to resolve a service directly from ``IServiceProvider``. In that case, you can inject `IServiceProvider` into your class and use the ``GetService`` or the `GetRequiredService` method as shown below:

````C#
public class MyService : ITransientDependency
{
    private readonly ITaxCalculator _taxCalculator;

    public MyService(IServiceProvider serviceProvider)
    {
        _taxCalculator = serviceProvider.GetRequiredService<ITaxCalculator>();
    }
}
````

### Dealing with multiple implementations

You can register multiple implementations of the same service interface. Assume that you have an `IExternalLogger` interface with two implementations:

````csharp
public interface IExternalLogger
{
    Task LogAsync(string logText);
}

public class ElasticsearchExternalLogger : IExternalLogger
{
    public async Task LogAsync(string logText)
    {
        //TODO...
    }
}

public class AzureExternalLogger : IExternalLogger
{
    public Task LogAsync(string logText)
    {
        throw new System.NotImplementedException();
    }
}
````

In this example, we haven't registered any of the implementation classes to the dependency injection system yet. So, if we try to inject the `IExternalLogger` interface, we get an error indicating that no implementation found.

If we register both of the `ElasticsearchExternalLogger` and `AzureExternalLogger` services for the `IExternalLogger` interface, and then try to inject the `IExternalLogger` interface, then the last registered implementation will be used.

An example service injecting the `IExternalLogger` interface:

````csharp
public class MyService : ITransientDependency
{
    private readonly IExternalLogger _externalLogger;

    public MyService(IExternalLogger externalLogger)
    {
        _externalLogger = externalLogger;
    }

    public async Task DemoAsync()
    {
        await _externalLogger.LogAsync("Example log message...");
    }
}
````

Here, as said before, we get the last registered implementation. However, how to determine the last registered implementation?

If we implement one of the dependency interfaces (e.g. `ITransientDependency`), then the registration order will be uncertain (it may depend on the namespaces of the classes). The *last registered implementation* can be different than you expect. So, it is not suggested to use the dependency interfaces to register multiple implementations.

You can register your services in the `ConfigureServices` method of your module:

````csharp
public override void ConfigureServices(ServiceConfigurationContext context)
{
    context.Services.AddTransient<IExternalLogger, ElasticsearchExternalLogger>();
    context.Services.AddTransient<IExternalLogger, AzureExternalLogger>();
}
````

In this case, you get an `AzureExternalLogger` instance when you inject the `IExternalLogger` interface, because the last registered implementation is the `AzureExternalLogger` class.

When you have multiple implementation of an interface, you may want to work with all these implementations. Assume that you want to write log to all the external loggers. We can change the `MyService` implementation as the following:

````csharp
public class MyService : ITransientDependency
{
    private readonly IEnumerable<IExternalLogger> _externalLoggers;

    public MyService(IEnumerable<IExternalLogger> externalLoggers)
    {
        _externalLoggers = externalLoggers;
    }

    public async Task DemoAsync()
    {
        foreach (var externalLogger in _externalLoggers)
        {
            await externalLogger.LogAsync("Example log message...");
        }
    }
}
````

In this example, we are injecting `IEnumerable<IExternalLogger>` instead of `IExternalLogger`, so we have a collection of the `IExternalLogger` implementations. Then we are using a `foreach` loop to write the same log text to all the `IExternalLogger` implementations.

If you are using `IServiceProvider` to resolve dependencies, then use its `GetServices` method to obtain a collection of the service implementations:

````csharp
IEnumerable<IExternalLogger> services = _serviceProvider.GetServices<IExternalLogger>();
````

### Releasing/Disposing Services

If you used a constructor or property injection, you don't need to be concerned about releasing the service's resources. However, if you have resolved a service from ``IServiceProvider``, in some cases, you might need to take care about releasing the service resources.

ASP.NET Core releases all services at the end of a current HTTP request, even if you directly resolved from ``IServiceProvider`` (assuming you injected `IServiceProvider`). But, there are several cases where you may want to release/dispose manually resolved services:

* Your code is executed outside of ASP.NET Core request and the executer hasn't handled the service scope.
* You only have a reference to the root service provider.
* You may want to immediately release & dispose services (for example, you may creating too many services with big memory usages and don't want to overuse the memory).

In any case, you can create a service scope block to safely and immediately release services:

````C#
using (var scope = _serviceProvider.CreateScope())
{
    var service1 = scope.ServiceProvider.GetService<IMyService1>();
    var service2 = scope.ServiceProvider.GetService<IMyService2>();
}
````

Both services are released when the created scope is disposed (at the end of the `using` block).

### Cached Service Providers

ABP provides two special services to optimize resolving services from `IServiceProvider`. `ICachedServiceProvider` and `ITransientCachedServiceProvider` both inherits from the `IServiceProvider` interface and internally caches the resolved services, so you get the same service instance even if you resolve a service multiple times.

The main difference is the `ICachedServiceProvider` is itself registered as scoped, while the `ITransientCachedServiceProvider` is registered as transient to the dependency injection system.

The following example injects the `ICachedServiceProvider` service and resolves a service in the `DoSomethingAsync` method:

````csharp
public class MyService : ITransientDependency
{
    private readonly ICachedServiceProvider _serviceProvider;

    public MyService(ICachedServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public async Task DoSomethingAsync()
    {
        var taxCalculator = _serviceProvider.GetRequiredService<ITaxCalculator>();
        // TODO: Use the taxCalculator
    }
}

````

With such a usage, you don't need to deal with creating service scopes and disposing the resolved services (as explained in the *Releasing/Disposing Services* section above). Because all the services resolved from the `ICachedServiceProvider` will be released once the service scope of the `MyService` instance is disposed. Also, you don't need to care about memory leaks (because of creating too many `ITaxCalculator` instances if we call `DoSomethingAsync` too many times), because only one `ITaxCalculator` instance is created, and it is reused.

Since `ICachedServiceProvider` and `ITransientCachedServiceProvider` extends the standard `IServiceProvider` interface, you can use all the extension method of the `IServiceProvider` interface on them. In addition, they provides some other methods to provide a default value or a factory method for the services that are not found (that means not registered to the dependency injection system). Notice that the default value (or the value returned from your factory method) is also cached and reused.

Use `ICachedServiceProvider` (instead of `ITransientCachedServiceProvider`) unless you need to create the service cache per usage. `ITransientCachedServiceProvider` guarantees that the created service instances are not shared with any other service, even they are in the same service scope. The services resolved from `ICachedServiceProvider` are shared with other services in the same service scope (in the same HTTP Request, for example), so it can be thought as more optimized.

> ABP Framework also provides the `IAbpLazyServiceProvider` service. It does exists for backward compatibility and works exactly same with the `ITransientCachedServiceProvider` service. So, use the `ITransientCachedServiceProvider` since the `IAbpLazyServiceProvider` might be removed in future ABP versions.

## Advanced Features

### IServiceCollection.OnRegistered Event

You may want to perform an action for every service registered to the dependency injection. In the `PreConfigureServices` method of your module, register a callback using the `OnRegistered` method as shown below:

````csharp
public class AppModule : AbpModule
{
    public override void PreConfigureServices(ServiceConfigurationContext context)
    {
        context.Services.OnRegistered(ctx =>
        {
            var type = ctx.ImplementationType;
            //...
        });
    }
}
````

`ImplementationType` provides the service type. This callback is generally used to add interceptor to a service. Example:

````csharp
public class AppModule : AbpModule
{
    public override void PreConfigureServices(ServiceConfigurationContext context)
    {
        context.Services.OnRegistered(ctx =>
        {
            if (ctx.ImplementationType.IsDefined(typeof(MyLogAttribute), true))
            {
                ctx.Interceptors.TryAdd<MyLogInterceptor>();
            }
        });
    }
}
````

This example simply checks if the service class has `MyLogAttribute` attribute and adds `MyLogInterceptor` to the interceptor list if so.

> Notice that `OnRegistered` callback might be called multiple times for the same service class if it exposes more than one service/interface. So, it's safe to use `Interceptors.TryAdd` method instead of `Interceptors.Add` method. See [the documentation](Dynamic-Proxying-Interceptors.md) of dynamic proxying / interceptors.

### IServiceCollection.OnActivated Event

The `OnActivated` event is raised once a service is fully constructed. Here you can perform application-level tasks that depend on the service being fully constructed - these should be rare.

````csharp
var serviceDescriptor = ServiceDescriptor.Transient<MyServer, MyServer>();
services.Add(serviceDescriptor);
if (setIsReadOnly)
{
    services.OnActivated(serviceDescriptor, x =>
    {
        x.Instance.As<MyServer>().IsReadOnly = true;
    });
}
````

> Notice that `OnActivated` event can be registered multiple times for the same `ServiceDescriptor`.

## 3rd-Party Providers

While ABP has no core dependency to any 3rd-party DI provider, it's required to use a provider that supports dynamic proxying and some other advanced features to make some ABP features properly work. 

Startup templates come with Autofac installed. See [Autofac integration](Autofac-Integration.md) document for more information.

## See Also

* [ASP.NET Core Dependency Injection Best Practices, Tips & Tricks](https://medium.com/volosoft/asp-net-core-dependency-injection-best-practices-tips-tricks-c6e9c67f9d96)
