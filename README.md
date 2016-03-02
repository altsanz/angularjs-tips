# angular-tips
Place where I put some of my angular tips, for my future me.

# Two-way data binding on directives

```html
  <my-directive value-binded="vm.variable"/>
```

```javascript

app.module('whatever').directive('my-directive', function() {
        return {
            restrict: 'E',
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

# Activate directive controller when two-way data binding is loaded

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


