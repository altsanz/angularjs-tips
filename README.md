# angularjs-tips
Place where I put some of my angular tips, for my future me.

- [Directives](#directives)
- [Testing](#testing)
- [Internals](#angularjs-internals)

## Directives

### Event listener: event.target vs event.currentTarget
- **target** is whatever you **actually clicked on**. It can vary, as this can be within an element that the event was bound to.
- **currentTarget** is the element you **actually bound the event to**. This will never change.

Source: _http://joequery.me/code/event-target-vs-event-currenttarget-30-seconds/_

### Symbols in scope object

1. "@"   (  Text binding / one-way binding )
2. "="   ( Direct model binding / two-way binding )
3. "&"   ( Behaviour binding / Method binding  )

```js
scope: {
        name: "@",
        color: "=",
        reverse: "&"
    }
```

```html
<div my-directive
  name="{{name}}"
  color="color"
  reverse="reverseName">
</div>
```

### Using passed methods with params through scope on directives

DONT DO: 
```html
<div my-directive
  method="methodName(param1, param2)">
</div>
```

As it forces you to use it the following way, which doesn't survive minification and is not DRY:
```js
controller: function(scope) {
        scope.methodName({ param1: 'value1', param2: 'value2' });
}
```

PLS DO:
```html
<div my-directive
  method="methodName">
</div>
```

And use it this way:
```js
controller: function(scope) {
        scope.methodName()('value1', 'value2');
}
```


### Two-way data binding on directives

```html
  <my-directive value-binded="vm.variable"/>
```

```javascript

app.module('whatever').directive('my-directive', function() {
        return {
            restrict: 'EA',
            scope: {
                valueBinded: '=valueBinded' // THIS IS THE IMPORTANT PART
            },
            controller: ['$scope', function($scope) {
              // binded value is accesible through scope
              console.log($scope.valueBinded) // Logs 
            }],
            template: '<input type="text" ng-model="valueBinded"/>';
        };
    });
    
```

### Activate directive controller when two-way data binding is loaded

Sometimes we need to wait until data is available so we can operate inside directive. Just add a $watch that run once, and when valid data is available run the activate() method.

Useful when we can show the template but running the controller would raise errors when running some methods working with value binded.

```javascript

app.module('whatever').directive('my-directive', function() {
        return {
            restrict: 'E',
            scope: {
                valueBinded: '=valueBinded' 
            },
            controller: ['$scope', function($scope) {
              // binded value is accesible through scope
              var unbindActivateListener = $scope.$watch("valueBinded", function(newVal, oldVal) {
                    if (typeof newVal !== "undefined") {  // Check if valueBinded contains valid information
                        unbindActivateListener();
                        activate();
                    }
                });
                
              function activate() {
                console.log($scope.valueBinded) // Logs 
              }
              
            }],
            template: '<input type="text" ng-model="valueBinded"/>';
        };
    });
    
```

### Link vs Controller, inside a directive

Link for only simple DOM manipulation and event handling.
Controller when logic or templates are expected to be _not that simple_.

## Testing

### Mocking a factory and injecting it in a controllerAs 

```javascript

describe('Controller: myCtrl as vm', function() {
  var myCtrl, scope;  // Declare here what you will use across the $provide and the tests

  // Initialize the controller and scope
  beforeEach(function() {
    // Load the controller's module
    module('myCtrlModule');

    // Provide any mocks needed
    module(function($provide) {
      // Provide all the dependencies injected on controller
      /*
      $provide.value('dependency1', {});
      */

      // Provide myFactory mock
      $provide.factory('myFactory', function($q) {  // Add $q as we will mock a promise response
        return {
          factoryMethod: jasmine.createSpy('factoryMethod').andCallFake(function(num) {
            return $q.when({
              "result": "this must be the result of your promise"
            });
          })
        };
      });
      
      // Add the rest of dependencies
      $provide.value('dependencyN', {});
    });


    inject(function($controller, _dependency1_, _myFactory_, _dependencyN_) {
      scope = {};
      
      // ADD dependencies to the controller
      myCtrl = $controller('myCtrl as vm', {
        $scope: scope,
        dependency1: _dependency1_,
        myFactory: _myFactory_,
        dependencyN: _dependencyN_
      });
    });

  });

  it('should exist', function() {
    expect(!!myCtrl).toBe(true);
  });

});

```

### Getting controller from a directive that is using controllerAs and BindToController syntax

When you have a directive created with a controller using controllerAs and BindToController syntax, in order to test controller methods you should better keep reading.

**FIRST: Never do element.controller('directiveName')**, as controller won't have binded values.

You have a directive created like this:
```javascript
angular.module('whatever', []).directive('myDirective', function () {
  return {
    restrict: 'E',
    templateUrl: 'components/myDirective/myDirective.tpl.html',
    scope: true,
    bindToController: {
      demoValueOne: '=',
      demoValueTwo: '='
    },
    controllerAs: 'vm',
    controller: 'MyDirectiveCtrl'
  }
})

.controller('MyDirectiveCtrl', function(myService, anotherService) {
    var self = this;
    
    self.method1 = method1;
    
    function method1(example1, example2) {
        [...]
    }
})
```

and then unit test must look like this:
```javascript
describe('my-directive', function() {

    var element, scope, ctrl;

    beforeEach(function() {
      angular.mock.module('my-module');

      angular.mock.inject(function($compile, $rootScope, $controller) {
        scope = $rootScope;
        element = angular.element('<my-directive demo-value-one="one" demo-value-two="two"></my-directive>');
        $compile(element)(scope);
        scope.$digest();
 
        ctrl = $controller(
                'MyDirectiveCtrl', {
                        myService: function() {},
                        anotherService: function(){}
                }, {
                        demoValueOne: 'one',
                        demoValueTwo: 'two'
                });
        
      });
    });

    it('should have a controller with values binded on it', function() {
      expect(ctrl).toBeDefined();
      expect(ctrl.demoValueOne).toBetruthy();
      expect(ctrl.demoValueTwo).toBetruthy();
    });

  });
```

### Expecting backend not to have been requested

When using httpBackend mock and expect it to not have been called, simply use the following line:

```js
expect(httpBackend.flush).toThrow();
```

As no httpBackend call is used, there's nothing to flush, so it raises an error that we detect with Jasmine! 

### Unit testing config blocks

Let's say you have a module that registers and http interceptor. And you wanna be sure that it is being registered during config block of that module.

How you assert that $http.interceptors array has your brand new interceptor?

In other words, how I make $httpProvider to be available inside 'it' unit test block?

On beforeEach create new module with a name that doesn't collide with other modules you want to test.

```js
beforeEach(function() {
    angular.module('componentWithConfig', [])
})
```

Then create a config block for this new module:

```js
beforeEach(function() {
    angular.module('componentWithConfig', [])
}).config(function() {});
```

Then inject $httpProvider inside this custom config block and save reference, as we do for regular unit testing:

```js
    
describe('Component logger', function() {    
    var httpProvider;
    beforeEach(function() {
        angular.module('loggerConfig', [])
        .config(function($httpProvider) { // Note that _$httpProvider_ notation doesnt work here
            httpProvider = $httpProvider; // Save reference
        })
    })
    
    
    it('Check logger interceptor has been registered on config block', function() {
        // Now $httpProvider is available. We got you, bastard.
        expect(httpProvider.interceptor.whatever).toBeTruthy();
    });
})
```

### Unit testing components with child-components

So you have a component or directive with some components on it's HTML. Imagine that child-component is another container so it interacts with services, and GUESS WHAT? On unit testing, those services are called, as child-components are rendered also.

but, you are unit testing. **Unit** testing. YOU **UNIT**, DUDE. Best approach would be to mock'em all. Mock that child-component. 

How you do that?

Directives are like factories, so do like you always mock bricks. You have a magic-counter directive, then you do the following on beforeEach:

```js

beforeEach(module('yourapp/test', function($provide){
  $provide.factory('magicCounterDirective', function(){ return {}; });
}));

// Or using the shorthand sytax
beforeEach(module('yourapp/test', { magicCounterDirective: {} ));
```

Props to [trodriges](https://stackoverflow.com/a/20951085/1306660).


## AngularJS Internals

### When is a digest cycle triggered?

There are basically three possible cases when the state of an application can change and these are the only moments where $digest cycles are needed. The case are:

- **User Interaction through events** - The user clicks UI controls like buttons and in turn triggers something in our application that changes state.

- **XMLHttpRequests** - Also known as AJAX. Something in our app requests some data from a server and update model data accordingly.

- **Timeouts** - Asynchronous operations cause through timers that can possibly change the state of our application

Source: _https://blog.thoughtram.io/angularjs/2015/01/14/exploring-angular-1.3-speed-up-with-applyAsync.html_

### $onDestroy VS $scope.on('$destroy', fn())?

There are 2 ways to define behaviour when destroying a scope, useful for removing listeners, using tear down methods. and so on.
They might look similar but they are NOT.

First one:

```js
function() {
    [...]
    this.$onDestroy = function() {}
}
```

Is only called when scope where controller or component lives in is destroyed. ONLY in this case.
So if from inside we do a $scope.$destroy()... $onDestroy won't be called.

Second way:

```js
function() {
    [...]
    $scope.$on('$destroy', function() { [...] })
}
```
    
This is called when inner scope is destroyed. Why is this? Because before really destroying scope, a $destroy event is broadcasted, from this child $scope to the rest of child scopes. So ctrl.$onDestroy(), which lives in a higher level won't notice.

### $digest vs $apply

**scope.$digest()** will fire watchers on the current scope, and on all of its children, too.
**scope.$apply** will evaluate passed function and run $rootScope.$digest().

The first one is faster, as it needs to evaluate watchers for current scope and its children. The second one is slower, as it needs to evaluate watchers for$rootScope and all it's child scopes.

Source _https://stackoverflow.com/questions/18697745/apply-vs-digest-in-directive-testing_

