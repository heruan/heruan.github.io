---
layout: post
title:  "Aurelia Property Injection"
date:   2016-05-01 10:00:00 +0200
author: Giovanni Lovato
categories: aurelia
tags: aurelia plugin depencency-injection di
---

[Aurelia](http://aurelia.io) bundles in its core a simple yet powerful
[DI](https://en.wikipedia.org/wiki/Dependency_injection) engine which provides a
hierarchy of containers able to manage your code dependencies in a
[loosely coupled](https://en.wikipedia.org/wiki/Loose_coupling) system.
This means that if you need an instance of the current `Router` in your active
VM, you can simply follow the convention of having a static array property named
`inject` in your VM code including the type `Router`; this convention will
instruct the container to pass the current `Router` instance to the VM
constructor. For example:

```javascript
import {Router} from "aurelia-router";

class MyViewModel {

    static inject = [ Router ];

    constructor(router) {
        // router will be the current Router instance
    }

}
```

Aurelia provides also a decorator which will take care of the conventions and
keep your code signature clean:

```javascript
import {inject} from "aurelia-dependency-injection";
import {Router} from "aurelia-router";

@inject(Router)
class MyViewModel {

    constructor(router) {
        // router will be the current Router instance
    }

}

```

## Types of injection

The whole process of depenency injection works making the assumption of an
additional convention: having the class constructor to be the entry point of
dependencies. This is often referenced as *constructor injection* [1] and suits
well for the vast majority of use cases, but it also implicitly tights the
constructor's arguments to the list of dependencies, which means it cannot
accept any argument that could/should not be injected and—when dependencies are
numerous—the constructor signature may become heavy.
For example:

```javascript
// imports

@inject(BindingEngine, TaskQueue, EventAggregator, Router, HttpClient, DialogService, I18N, Validation, Logger, Cache)
class MyViewModel {

    constructor(bindingEngine, taskQueue, eventAggregator, router, httpClient, dialogService, i18n, validation, logger, cache) {
        this.bindingEngine = bindingEngine;
        this.taskQueue = taskQueue;
        this.eventAggregator = eventAggregator;
        this.router = router;
        this.httpClient = httpClient;
        this.dialogService = dialogService;
        this.i18n = i18n;
        this.validation = validation;
        this.logger = logger;
        this.cache = cache;

        // This constructor is tighten to the class dependencies and
        // cannot accept other arguments, for example from child classes
    }

}
```

The DI pattern comes in help with another type of injection:
*property injection* (or *setter injection*).

## Extending Aurelia's DI Container for property injection

Aurelia's core DI system doesn't implement this type of injection, but it offers
hooks that can be used to extend its capabilities to other types of injection.
Let's take a look at how Aurelia's Container creates instances of the target
class with its dependencies:

1. will check for the presence of the conventional `inject` property to collect
    declared dependencies;
2. will select a strategy for invoking the target class constructor
    (an `Invoker`), possibly optimized for the number of dependencies;
3. will create an `InvocationHandler`, i.e. the object storing the information
    needed to create the target class instance;
4. if defined, will call the invocation handler creation callback, that will
    accept the `InvocationHandler` and return the new one the  container will
    use;
5. the `InvocationHandler` will produce the instance and return it to the
    container for injection.

The invocation handler creation callback will be the hook we use to implement
property injection.

First, we need to know which properties should be injected with which
dependency; following the *convention over configuration* paradigm, we assume
property dependencies being defined as a static `injectProperties` enumeration
of `property: Type`, such as:

```javascript
// imports

class MyViewModel {

    injectProperties = {
        router: Router
    };

}
```

Then, we implement an `InvocationHandlerWrapper` which will wrap the default
`InvocationHandler` and inject the declared dependencies into the specified
properties:

```javascript
import {InvocationHandler} from "aurelia-dependency-injection";

class InvocationHandlerWrapper extends InvocationHandler {

    invoke(container, dynamicDependencies) {
        let instance = super.invoke(container, dynamicDependencies);
        return this.injectProperties(container, instance);
    }

    injectProperties(container, instance) {
        if ("injectProperties" in this.fn) {
            let dependencies = this.fn["injectProperties"];
            for (let property in dependencies) {
                instance[property] = container.get(dependencies[property]);
            }
        }
        return instance;
    }

}
```

Finally, we can simply set the callback on the container:

```javascript
container.setHandlerCreatedCallback(handler => {
    return new InvocationHandlerWrapper(handler.fn, handler.invoker, handler.dependencies);
});
```

Now all instances provided by the container will be checked against the presence
of declared property dependencies, which will then be injected on the instance.

As one can evince from the implementation, this type of injection satisfy
dependencies *after* the target instance is created, which means the target
class constructor won't have dependencies available yet—that could be fine if
you don't need them there, but may be an issue when some logic needs to be
handled with them right after injection.
This is a natural consequence of this type of injection—well known in many DI
implementation—which have a common workaround: delegating dependencies logic to
another class method, which will be invoked by the container before returning
the target instance.

In the wake of *convention over configuration*, we can instruct our
`InvocationHandlerWrapper` to also check for the presence of a conventional
`afterConstructor` method on the target class, and invoke it right before
returning the instance to the container.

```javascript
import {InvocationHandler} from "aurelia-dependency-injection";

class InvocationHandlerWrapper extends InvocationHandler {

    invoke(container, dynamicDependencies) {
        let instance = super.invoke(container, dynamicDependencies);
        return this.injectProperties(container, instance);
    }

    injectProperties(container, instance) {
        if ("injectProperties" in this.fn) {
            let dependencies = this.fn["injectProperties"];
            for (let property in dependencies) {
                instance[property] = container.get(dependencies[property]);
            }
        }
        if ("afterConstructor" in instance) {
            instance.afterConstructor.call(instance);
        }
        return instance;
    }

}

```

Now, we can write our VM like this:

```javascript
// imports

class MyViewModel {

    injectProperties = {
        router: Router
    };

    constructor(somethingFromChildClasses) {
        // now we can pass other arguments to the constructor
        this.somethingFromChildClasses = somethingFromChildClasses;

        // here this.router is still undefined
    }

    afterConstructor() {
        // here this.router is the instance of the current Router
    }

}
```

## The extension as a plugin

I've written an Aurelia's module to make this extension available as a plugin
[2], which also provides extended versions of the `inject` decorator and its
TypeScript counterpart `autoinject` and can be used like this:

```javascript
import {inject} from "aurelia-property-injection";
// other imports

class MyViewModel {

    @inject(BindingEngine)
    bindingEngine;

    @inject(TaskQueue)
    taskQueue;

    @inject(EventAggregator)
    eventAggregator;

}
```

and, in TypeScript:

```typescript
import {autoinject} from "aurelia-property-injection";
// other imports

class MyViewModel {

    @autoinject
    protected bindingEngine: BindingEngine;

    @autoinject
    protected taskQueue: TaskQueue;

    @autoinject
    protected eventAggregator: EventAggregator;

}
```

## Questions and feedbacks

We have seen how Aurelia's modular architecture permits the extension of its
core parts like the DI container with a simple hook.
Feel free to ask questions, suggest improvements and submit pull-requests at
[2].
To be updated with Aurelia's active development, I suggest you to follow its
blog at [3].

[1]: https://en.wikipedia.org/wiki/Dependency_injection#Three_types_of_dependency_injection "Three types of dependency injection"
[2]: https://github.com/heruan/aurelia-property-injection "aurelia-property-injection"
[3]: http://blog.durandal.io "Aurelia's blog"

*[DI]: Dependency Injection
*[VM]: View Model
