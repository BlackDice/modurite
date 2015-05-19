# Modurite #

[![Join the chat at https://gitter.im/BlackDice/modurite](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/BlackDice/modurite?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

Powerful, yet quite simple way to create applications with loosely coupled modular architecture that imposes very few constrains.

## In development ##

This project is at early phases of a development. Current plan is to write complete documentation first followed by full unit tests. Only after that a functional code will be written.

Framework will be written using ES6 and [http://babeljs.io/]. There should be a very few of other runtime dependencies.

## Basic usage

To use Modurite, just require it in your code.

```javascript
    var modurite = require('modurite');
```

Very basic module is merely just a function.

```javascript
    modurite(function() {
        // here goes module code - initialization
    });
```

Doesn't look very useful, right? Luckily modules can be nested.

```javascript
    var mainModule = function() {
        this.add(firstModule);
        this.add(secondModule);  
    };
    var firstModule = function() {
        console.log("I am 1st module of my parent");
    }
    var secondModule = function() {
        console.log("I am 2nd module of my parent");
    }
    modurite(mainModule); // both messages are logged in the console after this
```

## Lifecycle phase ##

This may sound surprising, but there is **no predefined lifecycle** at all. You can have as many phases as you need. Simply return object with a map of lifecycle methods from the module initialization. This is completely optional and you don't need to return anything.

```javascript
    var lifeModule = function() {
        return {
            start: function(options) {
                // Do some stuff when module is started
            },
            stop: function() {
                // Perhaps setting module to default state ?
            }
        }
    }
```

You cannot access these methods from the outside. Instead you can execute lifecycle phases simply like this:

```javascript
    // grab the invoker object
    var lifeModuleInvoker = modurite(lifeModule);
    
    // invoke start phase with some optional data
    lifeModuleInvoker.invoke('start', {data: 'foo'});
    
    // invoke stop phase with a delay
    setTimeout(lifeModuleInvoker.invokeBind('stop'), 5000);
```

## Nested lifecycle phases ##

Lets combine two features explained so far. Since lifecycle methods are completely optional, you don't need to worry about how many (if any) of your nested modules actually uses specified lifecycle phase. It will propagate through whole hiearchy and invoke available lifecycle methods.

```javascript
    var topModule = function() {
        this.add(helloModule);
    };
    var helloModule = function() {
        return {
            hello: function(str) {
                console.log("Hello, " + str); // "Hello, master"
            }
        }
    };
    var topInvoker = modurite(topModule);
    topInvoker('hello', 'master');
```

### Execution order ###

Once your nested hierarchy gets more complex, it might be important to know that lifecycle methods are executed from the bottom to the top sequentially. This helps to ensure that lower placed modules will be prepared before their parents.

Consider following example where every shown module in a hiearchy has common lifecycle method named `start`. Modules without such method are not considered for the invocation, but nested modules inside of them will be processed. Bracketed numbers represents the execution order.

    root [9]
        first [5]
            child1 [1]
            child2 [2]
            child3 [4]
                subchild [3]
        second [8]
            child4 [6]
            child5 [7]

## Updates to a hiearchy ##

Generally it's recommended to add all your nested modules during module initialization phase. Adding module later (like during lifecycle phase) may impose some chaotic situation for you. Consider following example.

```javascript
    var lateModule = function() {
        return {
            setup: function() {
                // this is not invoked !
            }
        }
    }
    var topModule = function() {
        return {
            setup: function() {
                this.add(lateModule);
            }
        }
    }
    modurite(topModule).invoke('setup');
```

Since the `lateModule` wasn't part of hiearchy when `setup` phase has been invoked, it wont have its `setup` method called.

### Removing nested module ###

Since memory footprint of modules is generally very low, they are meant to stay in there during application lifetime and just do its job. In case you really need a module with rather short lifespan, it's possible to remove it, but you should never forget about proper cleanup.

```javascript
    var innerModule = function() {
        return {
            destroy: function() {
                // do some cleanup here as this called first
            }
        }
    };
    var topModule = function() {
        this.add(innerModule);
        return {
            destroy: function() {
                this.remove(innerModule);
            }
        }
    };
```

Removed module is popped out of the hiearchy together with any childs it may have and stops responding to any livecycle phases. Since the execution order of lifecycle methods is going from the bottom, you can use `destroy` phase here to actually take care of cleanup for the `innerModule`.

## Error handling ##

The Modurite generally catches all unhandled exceptions in a module code. These are wrapped with some additional meta information into customized `ModuriteError` object. Resulting error is re-thrown unless you sign up for its handling with one of the following methods.

```javascript
    modurite.onError(function(err) {
    });
```

This is very basic and rather insufficient way of handling errors. It's meant only to handle anything that escapes your attention in a global manner. This way you can eg. log your errors and appologize to the user. It cannot ensure that your application isn't broken when this happens.

To handle errors related to a single hiearchy of a modules, you can use similar approach.

```javascript
    var badInvoker = modurite(errorneousModule);
    badInvoker.onError(function(err) {
        // handles all errors for the whole hiearchy 
    });
    badInvoker.onError('start', function(err) {
        // handles errors during start lifecycle phase whenever it is invoked
    });
    badInvoker.invoke('start');
```

You may have noticed, that errors during initialization of modules cannot be handled this way, because handler is added after initialization is done. Such errors are usually considered app breaking and should appear only during development. Global handler mentioned above will serve well to catch these errors.

## Do you like `this` ? ##

If you have some experience with code minification/obfluscation, you may know, that `this` cannot be mangled in any way. To help you battle with this, first argument of the module initialization function is identical to `this` keyword.

```javascript
    modurite(function(smallerModule) {
        smallerModule.add(otherModule);
    });
```

## Naming a module ##

You may think that by naming your module you can get access it from the anywhere you like. Sorry, but you don't. Why would even want that? Are you interested in loosely coupled modules or not?

Name is here merely as information to ease a development. It makes easier to identify what module you are editing right now. It is also used for debugging (stack traces) and logging purposes.

```javascript
    modurite(function() {
        this.name = 'mClient';
    });
```

Modurite tries to be somewhat smart on this. If you specify first argument to a function as mentioned above, it will be set as a module name for you. Thus avoid useing generic names in there, eg. "myModule", or simply override the name with anything you like.

```javascript
    modurite(function(mClient) {
        this.name === "mClient" //true
    });
```

## Advanced usage

Now we have covered all basic functionality, but there are many other options and tools to aid you with even more decoupling and communication between modules.

 * [Support for async modules](docs/async.md)
 * [Module configuration](docs/config.md)
 * [Logging and debugging](docs/logging.md)
 * [Event emitter](docs/events.md)
 * [Requests and commands](docs/radio.md)
 * [Synchronization director](docs/director.md)

## Final thoughts ##

### Module isolation ###

Every initialized module exists as completely separate entity. It doesn't need to care about its placement in hierachy. It simply does its job when requested.

### Module reuse ###

Single module function can be reused as much as you want in different other hiearchies. The Modurite only makes sure that you don't use same module function more than once in a single hierarchy.

### Multiple hierarchies ###

You are not limited to a **single `modurite` call**. If you like you can even have a root module that creates several *sub-modules* with an internal `modurite` calls. That way you are creating another completely separate hiearchies with their own lifecycle phases. You can make nice pluggable architecture with this!

### Unit testing ###

Since the module is basically just a plain function, its very easy to write unit tests for it. Instead of testing whole hierarchy of modules, you can just wrap a single module with a `modurite` call.
