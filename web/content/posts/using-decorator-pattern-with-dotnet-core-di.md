+++
date = '2018-02-23T00:00:00Z'
draft = true
title = 'Using Decorator pattern with .Net Core dependency injection container'
categories = [".NET"]
+++

### TL;DR

Configure a decorator with DI as simple as this:

```java
services.AddDecorator<IEmailMessageSender, EmailMessageSenderWithRetryDecorator>(
    decorateeServices => 
    {
        decorateeServices.AddScoped<IEmailMessageSender, SmtpEmailMessageSender>();
    });
```

You can find the full code as well as demo application at https://github.com/andriysavin/Blog/tree/main/DecoratorAndDotNetDI.

### Main()

Imagine you need to wrap some service with a [decorator](https://en.wikipedia.org/wiki/Decorator_pattern) to add some functionality over it, and you need to do this by configuring your DI container. While many popular DI/IoC containers provide this functionality out of the box, this is not true for .Net Core's DI (`IServiceCollection`). Let's consider particular example.

Here is our service:

```csharp
interface IEmailMessageSender
{
    void SendMessage(EmailMessage message);
}
```

And here is the main implementation:

```csharp
class SmtpEmailMessageSender : IEmailMessageSender
{
    public void SendMessage(EmailMessage message)
    {
        // Put your real implementation here.
        Console.WriteLine(message.Body);
    }
}
```

Now I want to decorate this service with retry logic:

```csharp
class EmailMessageSenderWithRetryDecorator : IEmailMessageSender
{
    private readonly IEmailMessageSender innerSender;

    // Hardcoded for simplicity.
    private const int RetryCount = 3;
    private const int DelayBetweenRetriesMs = 1000;

    public EmailMessageSenderWithRetryDecorator(
        IEmailMessageSender innerSender
        /* Additionally pass an ILogger, IOptions etc. */)
    {
        this.innerSender = innerSender;
    }

    public void SendMessage(EmailMessage message)
    {
        for (int i = 1; i <= RetryCount; i++)
        {
            Console.WriteLine($"Sending message, retry {i} of {RetryCount}...");

            try
            {
                innerSender.SendMessage(message);

                Console.WriteLine($"Message sent!");

                return;
            }
            catch (Exception) // Catch concrete exception type in real code
            {
                if (i == RetryCount)
                {
                    throw;
                }
            }

            Thread.Sleep(DelayBetweenRetriesMs);
        }
    }
}
```

Essentially, what you need to do is tell the DI container that when `IEmailMessageSender` service is requested, create decorator `EmailMessageSenderWithRetryDecorator` instance, create the original service implementation `SmtpEmailMessageSender`, pass the latter to the former, and return the decorator instance. Logically this looks like this:

```csharp
new EmailMessageSenderWithRetryDecorator(
	innerSender: new SmtpEmailMessageSender(/*dependencies*/)
	/*, other dependencies*/);
```

Basically, decorator creation involves resolving the original service implementation to pass it as a dependency (among others) to the decorator. This is beautiful: both the decorator and the "decoratee" (I'll use this word throughout this text) creation is the job of the DI container. But there is no way to tell the container when the service interface must be resolved to the original implementation or to the decorator. Usually, the last configured mapping wins. The answer is: **"hide" the decoratee from the list of service implementers and explicitly control its creation**.

Let's first look at how this could be configured and run:

```csharp
var services = new ServiceCollection();

services.AddDecorator<IEmailMessageSender, EmailMessageSenderWithRetryDecorator>(
    decorateeServices => decorateeServices.AddScoped<IEmailMessageSender, SmtpEmailMessageSender>());

var serviceProvider = services.BuildServiceProvider();

var emailSender = serviceProvider.GetRequiredService<IEmailMessageSender>();

emailSender.SendMessage(new EmailMessage { Body = "Hello, decorator!" });
```

The `AddDecorator` is an extension method we will create for `IServiceCollection`. Basically, it configures decorator implementation injection and its inner service implementation injection. Then, we ask the service provider for `IEmailMessageSender` service in the usual way, and get the decorator instance with a decoratee instance injected to it.

Now it's left to see how that `AddDecorator` works:

```csharp
/*
 * Copyright (c) 2026 Andriy Savin
 *
 * This code is licensed under the MIT License.
 * See the LICENSE file in the repository root (https://github.com/andriysavin/Blog/) 
 * for full license text.
 * 
 * Attribution is appreciated when reusing this code.
 * 
 */

using System;
using System.Collections.Generic;
using System.Linq;

namespace Microsoft.Extensions.DependencyInjection
{
    public static class DecoratorServiceCollectionExtensions
    {
        public static void AddDecorator<TService, TDecorator>(
            this IServiceCollection serviceCollection,
            Action<IServiceCollection> configureDecorateeServices)
            where TDecorator : class, TService
            where TService : class
        {
            var decorateeServices = new ServiceCollection();

            // This calls back to the decoratee configuration lambda.
            configureDecorateeServices(decorateeServices);

            var decorateeDescriptor =
                // For now, support defining only single decoratee.
                // TODO: To support cases such as composite decorators 
                // (accepting multiple decoratees and representing them as single)
                // implement handling multiple decoratee configurations.
                decorateeServices.SingleOrDefault(sd => sd.ServiceType == typeof(TService));

            if (decorateeDescriptor == null)
            {
                throw new InvalidOperationException("No decoratee configured!");
            }

            // We will replace this descriptor with a tweaked one later.
            decorateeServices.Remove(decorateeDescriptor);

            // Add all remaining services to main collection.
            serviceCollection.Add(decorateeServices);

            // This factory allows us to pass some dependencies 
            // (the decoratee instance) manually,
            // which is not possible with something like GetRequiredService. 
            var decoratorInstanceFactory = ActivatorUtilities.CreateFactory(
                typeof(TDecorator), new[] { typeof(TService) });

            Type decorateeImplType = decorateeDescriptor.GetImplementationType();

            Func<IServiceProvider, TDecorator> decoratorFactory = sp =>
            {
                // Note that we query the decoratee by its implementation type,
                // avoiding any ambiguity. 
                var decoratee = sp.GetRequiredService(decorateeImplType);
                // Pass the decoratee manually. All other dependencies are resolved as usual.
                var decorator = (TDecorator)decoratorInstanceFactory(sp, new[] { decoratee });
                return decorator;
            };

            // Decorator inherits decoratee's lifetime.
            var decoratorDescriptor = ServiceDescriptor.Describe(
                typeof(TService),
                decoratorFactory,
                decorateeDescriptor.Lifetime);

            // Re-create the decoratee without original service type (interface).
            // This allows to create decoratee instances via
            // service provider, utilizing its lifetime scope
            // control finctionality.
            decorateeDescriptor = RefactorDecorateeDescriptor(decorateeDescriptor);

            serviceCollection.Add(decorateeDescriptor);
            serviceCollection.Add(decoratorDescriptor);
        }

        /// <summary>
        /// The goal of this method is to replace the service type (interface)
        /// with the implementation type in any kind of service descriptor.
        /// Actually, we build new service descriptor.
        /// </summary>
        private static ServiceDescriptor RefactorDecorateeDescriptor(ServiceDescriptor decorateeDescriptor)
        {
            var decorateeImplType = decorateeDescriptor.GetImplementationType();

            if (decorateeDescriptor.ImplementationFactory != null)
            {
                decorateeDescriptor =
                    ServiceDescriptor.Describe(
                    serviceType: decorateeImplType,
                    decorateeDescriptor.ImplementationFactory,
                    decorateeDescriptor.Lifetime);
            }
            else
            if (decorateeDescriptor.ImplementationInstance != null)
            {
                decorateeDescriptor =
                    ServiceDescriptor.Singleton(
                    serviceType: decorateeImplType,
                    decorateeDescriptor.ImplementationInstance);
            }
            else
            {
                decorateeDescriptor =
                    ServiceDescriptor.Describe(
                    decorateeImplType, // Yes, use the same type for both.
                    decorateeImplType,
                    decorateeDescriptor.Lifetime);
            }

            return decorateeDescriptor;
        }

        /// <summary>
        /// Infers the implementation type for any kind of service descriptor
        /// (i.e. even when implementation type is not specified explicitly).
        /// </summary>
        private static Type GetImplementationType(this ServiceDescriptor serviceDescriptor)
        {
            if (serviceDescriptor.ImplementationType != null)
                return serviceDescriptor.ImplementationType;

            if (serviceDescriptor.ImplementationInstance != null)
                return serviceDescriptor.ImplementationInstance.GetType();

            // Get the type from the return type of the factory delegate.
            // Due to covariance, the delegate object can have more concrete type
            // than the factory delegate defines (object).
            if (serviceDescriptor.ImplementationFactory != null)
                return serviceDescriptor.ImplementationFactory.GetType().GenericTypeArguments[1];

            // This should not be possible, but just in case.
            throw new InvalidOperationException("No way to get the decoratee implementation type.");
        }
    }
}
```

Essentially, we intercept decoratee services configuration and tweak the decoratee service descriptor to not announce it implements the service interface. After that the decoratee can only tell that it "implements itself". We can create its instance via `IServiceProvider`. This is an important point, because the service provider  not only creates instances ***with respect to configured lifetime scope***, but also calls `Dispose` when the lifetime ends (if `IDisposable` is implemented, of course).  Also, service provider allows other familiar dependency resolution approaches, such as providing a factory delegate or existing object.

### Bonus

You can also configure multiple levels of decorators like this:

```java
services.AddDecorator<IEmailMessageSender, EmailMessageSenderWithRetryDecorator>(
    decorateeServices =>
    {
        decorateeServices.AddDecorator<IEmailMessageSender, SecondEmailSenderDecorator>(
            decorateeServices2 =>
                decorateeServices2.AddScoped<IEmailMessageSender, SmtpEmailMessageSender>());
    });
```

You can find the full code as well as demo application at https://github.com/andriysavin/Blog/tree/main/DecoratorAndDotNetDI.

Try it yourself and let me know what you think!