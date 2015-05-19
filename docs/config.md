# Module configuration #

Consider following simple module for a database access.

```javascript
    var databaseModule = function() {
        var config = {
            port: 8000,
            host: "localhost"
        };
        var connectionString = buildConnectingString(config.host, config.port);
    };
```

As you can imagine, this may become an issue if you need to change these values, perhaps conditionally depending on environment your code is running at. How to solve this without need for a lot of changes? 

```javascript
    var databaseModule = function() {
        var config = this.configure({
            port: 8000,
            host: "localhost"
        });
        var connectionString = buildConnectingString(config.host, config.port);
    };
    // pass in your configuration object if you have one
    modurite(configuredModule, passInConfigFromExternalSource());
```

Small difference with rather big impact. Just wrap your config data with call of the `configure` method and suddenly you are getting configuration from external source while keeping defaults for proper module functioning! Now you can easily configure your modules without ever changing code inside.

## How the `configure` works? ##

It simply iterates over keys of object passed in and looks for same keys in configuration object for a hiearchy (if any). If the key is not found, it will use value provided. If object is found at the requested key, a deep copy of this object is made instead of keeping reference.

Resulting value is always a copy with no ties to global configuration thus you can change it freely. 

You can call `configure` method as much as you want to separate to different objects that can be passed around to your own functions. 

## Complex configuration tree ##

Consider following configuration object being passed to `modurite` call.

```javascript
    var configuration = {
        top: {
            subtop: {
                nested: {
                    target: "i want this",
                    goal: "this too"
                }
            }
        }
    };
```

Instead of simple key, you can specify path to that key instead. Resulting config contains only last part of the path.

```javascript
    var nestedModule = function() {
        var config = this.configure({
           "top.subtop.nested.target": "default",
           "top.subtop.nested.goal": "big"
        });
        config.target == "i want this" //true
    };
```

That would be rather painful. Instead you can simply set common relative root for lookup.

```javascript
    var nestedModule = function() {
        var config = this.configure({
            target: "default",
            goal: "big"
        }, "top.subtop.nested");
    };
```

Or perhaps combine both solutions...

```javascript
    var nestedModule = function() {
        var config = this.configure({
           "nested.target": "default"
           "nested.goal": "big"
        }, "top.subtop");
        config.target == "i want this" //true
    };
```

In overall you might consider avoiding such deeply nested configurations as it makes your modules dependant on some specific structure of configuration object.

## Configuring nested modules ##

Another way how to fight with nested configuration tree is letting parent module to configure its child modules.

```javascript
    var nestedModule = function() {
        var config = this.configure({
           target: "default",
           goal: "big"
        });
        config.target == "i want this" //true
        this.add(mostNestedModule);
    };
    var parentModule = function(
        this.add(nesteModule, "top.subtop.nested");
    );
```

Moving resposibility to parent makes nested modules much more clean and independent. With a clever layering of modules, you can create single top module that splits your big clumsy configuration into small subsets.

Note that any nested modules in the one you have changed configuration for, will use this relative configuration instead of global one. Following rom previous example...

```javascript
    var mostNestedModule = function() {
        var config = this.configure({
           "top.subtop.nested.target": "default",
           goal: "small"
        });
        config.target == "default" //true
        config.goal == "this too"
    };
```

Instead of passing string root when adding module, you can also pass an object and assemble configuration for the nested module on your own.

```javascript
    var parentModule = function() {
        this.add(nestedModule, this.configure({
           target: "nested-default",
           goal: "extra large"
        }, "top.subtop.nested"))
    };
```

## Type checking ##

Sometimes it might happen that you change your original configuration to suit needs of another module without remembering some old module that expects different format.

```javascript
    var configuredModule = function() {
        var config = this.configure({
            source: "alpha"
        });
        config.source.replace(...) // error!
    };
    modurite(configuredModule, {
        source: ["foo", "bar"]
    });
```

Modurite will warn you about this when type is different from one you have specified in module configuration. 
