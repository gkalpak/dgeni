# Dgeni - Documentation Generator [![Build Status](https://travis-ci.org/angular/dgeni.svg?branch=master)](https://travis-ci.org/angular/dgeni)

![](assets/dgeni-logo-600x400.png)

The node.js documentation generation utility by angular.js and other projects.

Dgeni is pronounced like the girl name Jenny ([/ˈdʒɛni/](https://en.wikipedia.org/wiki/Help:IPA_for_English)),
i.e the `d` is silent and the `g` is soft.

## Getting started

Try out the Dgeni [example project](https://github.com/petebacondarwin/dgeni-example). Or maybe you're looking for an example [using AngularJS](https://github.com/petebacondarwin/dgeni-angular).

Watch Pete's ng-europe talk on Dgeni :

[![ScreenShot](http://img.youtube.com/vi/PQNROxXajyQ/0.jpg)](http://youtu.be/PQNROxXajyQ)


## Documenting AngularJS Apps

There are two projects out there that build upon dgeni to help create documentation for AngularJS apps:

* dgeni-alive: https://github.com/wingedfox/dgeni-alive
* sia: https://github.com/boundstate/sia

Do check them out and thanks to [Ilya](https://github.com/wingedfox) and [Bound State Software](https://github.com/boundstate)
for putting these projects together.

## Installation

You'll need node.js and a bunch of npm modules installed to use Dgeni.  Get node.js from here:
http://nodejs.org/.

In the project you want to document you install Dgeni by running:

```
npm install dgeni --save-dev
```

This will install Dgeni and any modules that Dgeni depends upon.


## Running Dgeni

Dgeni on its own doesn't do much.  You must configure it with **Packages** that contain **Services**
and **Processors**. It is the **Processors** that actually convert your source files to
documentation files.

To run the processors we create a new instance of `Dgeni`, providing to it an array of **Packages**
to load.  Then simply call the `generate()` method on this instance.  The `generate()` method runs the
processors asynchronously and returns a **Promise** that gets fulfilled with the generated documents.

```js
var Dgeni = require('dgeni');

var packages = [require('./myPackage')];

var dgeni = new Dgeni(packages);

dgeni.generate().then(function(docs) {
  console.log(docs.length, 'docs generated');
});
```

### Running from the Command Line

Dgeni is normally used from a build tool such as Gulp or Grunt but it does also come with a
command line tool.

If you install Dgeni globally then you can run it from anywhere:

```bash
npm install -g dgeni
dgeni some/package.js
```

If Dgeni is only installed locally then you either have to specify the path explicitly:

```bash
npm install dgeni
node_modules/.bin/dgeni some/package.js
```

or you can run the tool in an npm script:

```js
{
  ...
  scripts: {
    docs: 'dgeni some/package.js'
  }
  ...
}
```


The usage is:


```bash
dgeni path/to/mainPackage [path/to/other/packages ...] [--log level]
```

You must provide the path to one or more Dgeni Packages to load. You can, optionally, set
the logging level.


## Packages

**Services**, **Processors**, configuration values and templates are be bundled into a `Package`.  Packages
can depend upon other Packages.  In this way you can build up your custom configuration on
top of an existing configuration.

### Defining a Package

Dgeni provides a `Package` constructor to create new Packages.  A Package instance has methods to register **Services** and
**Processors**, and to configure the properties of **Processors**:

```js
var Package = require('dgeni').Package;
var myPackage = new Package('myPackage', ['packageDepencency1', 'packageDependency2']);

myPackage.processor(require('./processors/processor1'));
myPackage.processor(require('./processors/processor2'));

myPackage.factory(require('./services/service1'));
myPackage.factory(require('./services/service2'));

myPackage.config(function(processor1, service2) {
  service2.someProperty = 'some value';
  processor1.specialService = service2;
});
```


## Services

Dgeni makes significant use of **Dependency Injection (DI)** to instantiate objects.  Objects that
will be instantiated by the DI system must be provided by a **factory function**, which is registered
in a **Package**, either as a **Processor**, by `myPackage.processor(factoryFn)`, or as a **Service**,
by `myPackage.factory(factoryFn)`.

### Defining a Service

The parameters to a factory function are dependencies on other services that the DI system must find
or instantiate and provide to the factory function.

**car.js**:
```js
module.exports = function car(engine, wheels) {
  return {
    drive: function() {
      engine.start();
      wheels.turn();
    }
  };
};
```

Here we have defined a `car` service, which depends upon two other services, `engine` and `wheels`
defined elsewhere.  Note that this `car` service doesn't care how and where these dependencies are
defined. It relies on the DI system to provide them when needed.

The `car` service returned by the factory function is an object containing one method, `drive()`,
which in turn calls methods on `engine` and `wheels`.


### Registering a Service

You then register the Service with a Package:

**myPackage.js**:
```jsv
var Package = require('dgeni').Package;

module.exports = new Package('myPackage')
  .factory(require('./car'));
```

This car Service is then available to any other Service, Processor or configuration block:

```js
var Package = require('dgeni').Package;

module.exports = new Package('greenTaxiPackage', ['myPackage'])

  .factory(function taxi(car) {
    return {
      orderTaxi: function(place) { car.driveTo(place); }
    };
  })

  .config(function(car) {
    car.fuel = 'hybrid';
  });
```


## Processors

**Processors** are **Services** that contain a `$process(docs)` method.  The processors are run
one after another in a pipeline. Each Processor takes the collection documents from the previous
Processor and manipulates it, maybe inserting new documents or adding meta data to documents that are
there already.

Processors can expose properties that tell Dgeni where in the pipeline they should be run and
how to validate the configuration of the Processor.

* `$enabled` - if set to `false` then this Processor will not be included in the pipeline
* `$runAfter` - an array of strings, where each string is the name of a Processor that must appear
**earlier** in the pipeline than this Processor
* `$runBefore` - an array of strings, where each string is the name of a Processor that must appear
**later** in the pipeline than this one
* `$validate` - a [http://validatejs.org/](http://validatejs.org/) constraint object that Dgeni uses
to validate the properties of this Processor.


**Note that the validation feature has been moved to its own Dgeni Package `processorValidation`.
Currently dgeni automatically adds this new package to a new instance of dgeni so that is still available
for backward compatibility. In a future release this package will be moved to `dgeni-packages`.**

### Defining a Processor

You define Processors just like you would a Service:

**myDocProcessor.js**:
```js
module.exports = function myDocProcessor(dependency1, dependency2) {
  return {
    $process: function (docs) {
        //... do stuff with the docs ...
    },
    $runAfter: ['otherProcessor1'],
    $runBefore: ['otherProcessor2', 'otherProcessor3'],
    $validate: {
      myProperty: { presence: true }
    },
    myProperty: 'some config value'
  };
};
```


### Registering a Processor

You then register the Processor with a Package:
**myPackage.js**:
```jsv
var Package = require('dgeni').Package;

module.exports = new Package('myPackage')
  .processor(require('./myDocProcessor'));
```

### Asynchronous Processing

The `$process(docs)` method can be synchronous or asynchronous:

* If synchronous then it should return `undefined` or a new array of documents.
If it returns a new array of docs then this array will replace the previous `docs` array.
* If asynchronous then it must return a **Promise**, which should resolve to `undefined`
or a new collection of documents. By returning a **Promise**, the processor tells Dgeni
that it is asynchronous and Dgeni will wait for the promise to resolve before calling the
next processor.


Here is an example of an asynchronous **Processor**
```js
var qfs = require('q-io/fs');
module.exports = function readFileProcessor() {
  return {
    filePath: 'some/file.js',
    $process(docs) {
      return qfs.readFile(this.filePath).then(function(response) {
        docs.push(response.data);
      });
    }
  };
```

### Standard Dgeni Packages

The [dgeni-packages repository](https://github.com/angular/dgeni-packages) contains many Processors -
from basic essentials to complex, angular.js specific.  These processors are grouped into Packages:

* `base` -  contains the basic file reading and writing Processors as well as an abstract
rendering Processor.

* `jsdoc` - depends upon `base` and adds Processors and Services to support parsing and
extracting jsdoc style tags from comments in code.

* `typescript` - depends upon `base` and adds Processors and Services to support parsing and
extracting jsdoc style tags from comments in TypeScript (*.ts) code.

* `nunjucks` - provides a [nunjucks](http://mozilla.github.io/nunjucks/) based rendering
engine.

* `ngdoc` - depends upon `jsdoc` and `nunjucks` and adds additional processing for the
AngularJS extensions to jsdoc.

* `examples` - depends upon `jsdoc` and provides Processors for extracting examples from jsdoc
comments and converting them to files that can be run.

* `dgeni` - support for documenting dgeni Packages.


### Pseudo Marker Processors

You can define processors that don't do anything but act as markers for stages of the
processing.  You can use these markers in `$runBefore` and `$runAfter` properties to ensure
that your Processor is run at the right time.

The **Packages** in dgeni-packages define some of these marker processors. Here is a list
of these in the order that Dgeni will add them to the processing pipeline:


* reading-files *(defined in base)*
* files-read *(defined in base)*
* parsing-tags *(defined in jsdoc)*
* tags-parsed *(defined in jsdoc)*
* extracting-tags *(defined in jsdoc)*
* tags-extracted *(defined in jsdoc)*
* processing-docs *(defined in base)*
* docs-processed *(defined in base)*
* adding-extra-docs *(defined in base)*
* extra-docs-added *(defined in base)*
* computing-ids *(defined in base)*
* ids-computed *(defined in base)*
* computing-paths *(defined in base)*
* paths-computed *(defined in base)*
* rendering-docs *(defined in base)*
* docs-rendered *(defined in base)*
* writing-files *(defined in base)*
* files-written *(defined in base)*


## Configuration Blocks

You can configure the **Services** and **Processors** defined in a **Package** or its dependencies
by registering **Configuration Blocks** with the **Package**.  These are functions that can be
injected with **Services** and **Processors** by the DI system, giving you the opportunity to
set properties on them.


### Registering a Configuration Block

You register a **Configuration Block** by calling `config(configFn)` on a Package.

```js
myPackage.config(function(readFilesProcessor) {
  readFilesProcessor.sourceFiles = ['src/**/*.js'];
});
```

## Dgeni Events

In Dgeni you can trigger and handle **events** to allow packages to take part in the processing
lifecycle of the documentation generation.


### Triggering Events

You trigger an event simply by calling `triggerEvent(eventName, ...)` on a `Dgeni` instance.

The `eventName` is a string that identifies the event to be triggered, which is used to wire up
event handlers. Additional arguments are passed through to the handlers.

Each handler that is registered for the event is called in series. The return value
from the call is a promise to the event being handled. This allows event handlers to be async.
If any handler returns a rejected promise the event triggering is cancelled and the rejected
promise is returned.

For example:

```js
var eventPromise = dgeni.triggerEvent('someEventName', someArg, otherArg);
```

### Handling Events

You register an event handler in a `Package`, by calling `handleEvent(eventName, handlerFactory)` on
the package instance. The handlerFactory will be used by the DI system to get the handler, which allows
you to inject services to be available to the handler.

The handler factory should return the handler function. This function will receive all the arguments passed
to the `triggerHandler` method. As a minimum this will include the `eventName`.

For example:

```js
myPackage.eventHandler('generationStart', function validateProcessors(log, dgeni) {
  return function validateProcessorsImpl(eventName) {
    ...
  };
});

```

### Built-in Events

Dgeni itself triggers the following events during documentation generation:

* `generationStart`: triggered after the injector has been configured and before the processors begin
  their work.
* `generationEnd`: triggered after the processors have all completed their work successfully.
* `processorStart`: triggered just before the call to `$process`, for each processor.
* `processorEnd`: triggered just after `$process` has completed successfully, for each processor.
