+++
math = false
tags = [
    "analyze",
    'source code',
    'AngularJS'
]
date = "2017-03-06T19:10:05+08:00"
title = "Angular1 Source Code Loader"
image = ""

+++

基于版本：1.6.0

## Loader
---
Injector的章节我们说到了关于module的部分，关于模块的载入等都集中在了loader.js。
我们来看看`'src/loader.js'`。
<!--more-->

## loader是如何工作的
---
在loader中，整体被一个函数包裹
```
    function setupModuleLoader(window) {
    
      var $injectorMinErr = minErr('$injector');
      var ngMinErr = minErr('ng');
    
      function ensure(obj, name, factory) {
        return obj[name] || (obj[name] = factory());
      }
    
      var angular = ensure(window, 'angular', Object);
    
      // We need to expose `angular.$$minErr` to modules such as `ngResource` that reference it during bootstrap
      angular.$$minErr = angular.$$minErr || minErr;
      
      return ensure(angular, 'module', function() {
        var modules = {};
        return function module(name, requires, configFn){...};
      });
      
    }

```
可以看出，`ensure`函数用来防止模块定义冲突，如果存在该模块定义则直接返回，否则执行`factory`函数，获得并返回模块对象。
这里`ensure`了`angular`的定义，如果不存在则赋予一个空的object对象。

并且紧接着`ensure`了`angular`中`module`的定义。
最终`angular.module`就相当于`function module(name, requires, configFn){...}`
用了闭包使得`modules`成为一个private变量。

来看看我们平常是如何声明并使用`module`的。
`angular.module('MyApp', [...requires])`
对应下来name为MyApp，[...]为requires，即模块所依赖的模块。
来看看`function module`的具体实现。
```
    if (requires && modules.hasOwnProperty(name)) {
        modules[name] = null;
    }
```
如果已存在该模块定义，则清空重新进行定义
然后通过`ensure`来创建`modules`中`name`的定义。

函数最终`return moduleInstance;`
来看看关于moduleInstance的定义
```
    var moduleInstance = {
        // Private state
        _invokeQueue: invokeQueue,
        _configBlocks: configBlocks,
        _runBlocks: runBlocks,
        
        requires: requires,
        name: name,
        provider: invokeLaterAndSetModuleName('$provide', 'provider'),
        factory: invokeLaterAndSetModuleName('$provide', 'factory'),
        service: invokeLaterAndSetModuleName('$provide', 'service'),
        value: invokeLater('$provide', 'value'),
        constant: invokeLater('$provide', 'constant', 'unshift'),
        decorator: invokeLaterAndSetModuleName('$provide', 'decorator', configBlocks),
        animation: invokeLaterAndSetModuleName('$animateProvider', 'register'),
        filter: invokeLaterAndSetModuleName('$filterProvider', 'register'),
        controller: invokeLaterAndSetModuleName('$controllerProvider', 'register'),
        directive: invokeLaterAndSetModuleName('$compileProvider', 'directive'),
        component: invokeLaterAndSetModuleName('$compileProvider', 'component'),
        config: config,
        run: function(block) {
            runBlocks.push(block);
            return this;
        }
        
    }
```
可以看到，我们常用的用来定义app功能的方法都带上了`invokeLater*`的字样。来看看关于`invokeLater`和`invokeLaterAndSetModuleName`的定义
```
    function invokeLater(provider, method, insertMethod, queue) {
      if (!queue) queue = invokeQueue;
      return function() {
        queue[insertMethod || 'push']([provider, method, arguments]);
        return moduleInstance;
      };
    }
            
    function invokeLaterAndSetModuleName(provider, method, queue) {
      if (!queue) queue = invokeQueue;
      return function(recipeName, factoryFunction) {
        if (factoryFunction && isFunction(factoryFunction)) factoryFunction.$$moduleName = name;
        queue.push([provider, method, arguments]);
        return moduleInstance;
      };
    }

```
可以看到，向queue中添加了形如`[provider,method,arguments]`这样的以数组形式组织的元组。
所以当我们在真正调用`angular.module`之后的`.controller .filter .directive`这样的方法之后，我们所定义的内容都被加入了`queue`中等待`invoke`。
每次调用也都返回了`moduleInstance`，所以可以进行链式调用。

解读完这部分，就可以回到我们上一章的`Injector`，继续看module是如何invoke的了。

## 回到Injector
---
上一章我们留下了module相关的部分。
```
    var runBlocks = loadModules(modulesToLoad);
    //省略
    forEach(runBlocks, function(fn) { if (fn) instanceInjector.invoke(fn); });
```
关于loadModules的定义
```
    function loadModules(modulesToLoad) {
        assertArg(isUndefined(modulesToLoad) || isArray(modulesToLoad), 'modulesToLoad', 'not an array');
        var runBlocks = [], moduleFn;
        forEach(modulesToLoad, function(module) {
          //如果模块已经加载，直接返回
          if (loadedModules.get(module)) return;
          //没有加载时，设置为已加载，并继续执行
          loadedModules.put(module, true);
          //执行invoke队列的函数声明
          function runInvokeQueue(queue) {
            var i, ii;
            for (i = 0, ii = queue.length; i < ii; i++) {
              var invokeArgs = queue[i],
                  provider = providerInjector.get(invokeArgs[0]);
    
              provider[invokeArgs[1]].apply(provider, invokeArgs[2]);
            }
          }
    
          try {
            if (isString(module)) {
              moduleFn = angularModule(module);
              //这里递归加载所有依赖的模块
              runBlocks = runBlocks.concat(loadModules(moduleFn.requires)).concat(moduleFn._runBlocks);
              //执行invoke队列
              runInvokeQueue(moduleFn._invokeQueue);
              runInvokeQueue(moduleFn._configBlocks);
            } else if (isFunction(module)) {
                runBlocks.push(providerInjector.invoke(module));
            } else if (isArray(module)) {
                runBlocks.push(providerInjector.invoke(module));
            } else {
              assertArgFn(module, 'module');
            }
          } catch (e) {
            if (isArray(module)) {
              module = module[module.length - 1];
            }
            if (e.message && e.stack && e.stack.indexOf(e.message) === -1) {
              // Safari & FF's stack traces don't contain error.message content
              // unlike those of Chrome and IE
              // So if stack doesn't contain message, we create a new string that contains both.
              // Since error.stack is read-only in Safari, I'm overriding e and not e.stack here.
              // eslint-disable-next-line no-ex-assign
              e = e.message + '\n' + e.stack;
            }
            throw $injectorMinErr('modulerr', 'Failed to instantiate module {0} due to:\n{1}',
                      module, e.stack || e.message || e);
          }
        });
        return runBlocks;
      }

```
通过断点观察执行情况
可以看出，函数通过`queue`中的每一个元组拿到`provider`，并使用`provider`的相关方法来调用所存储的参数。
![loader1](/blog/img/post/AngularjsSourceCodeLoader/loader1.png)
我们还可以观察到，一些并未在`provider`中定义的Provider也进行了调用，比如`controller`
![loader2](/blog/img/post/AngularjsSourceCodeLoader/loader2.png)
在项目中进行检索，可以看到，在`AngularPublic`中进行了大量的provider注册
```
  //setupModuleLoader在这里进行了调用
  angularModule = setupModuleLoader(window);

  //进行了核心ng模块的声明
  angularModule('ng', ['ngLocale'], ['$provide',
    function ngModule($provide) {
      // $$sanitizeUriProvider needs to be before $compileProvider as it is used by it.
      $provide.provider({
        $$sanitizeUri: $$SanitizeUriProvider
      });
      //directive是$CompileProvider下的一个方法
      $provide.provider('$compile', $CompileProvider).
        //provider方法返回了$CompileProvider，所以可以调用directive
        directive({
            a: htmlAnchorDirective,
            input: inputDirective,
            textarea: inputDirective,
            form: formDirective,
            script: scriptDirective,
            select: selectDirective,
            option: optionDirective,
            ngBind: ngBindDirective,
            ngBindHtml: ngBindHtmlDirective,
            ngBindTemplate: ngBindTemplateDirective,
            ngClass: ngClassDirective,
            ngClassEven: ngClassEvenDirective,
            ngClassOdd: ngClassOddDirective,
            ngCloak: ngCloakDirective,
            ngController: ngControllerDirective,
            ngForm: ngFormDirective,
            ngHide: ngHideDirective,
            ngIf: ngIfDirective,
            ngInclude: ngIncludeDirective,
            ngInit: ngInitDirective,
            ngNonBindable: ngNonBindableDirective,
            ngPluralize: ngPluralizeDirective,
            ngRepeat: ngRepeatDirective,
            ngShow: ngShowDirective,
            ngStyle: ngStyleDirective,
            ngSwitch: ngSwitchDirective,
            ngSwitchWhen: ngSwitchWhenDirective,
            ngSwitchDefault: ngSwitchDefaultDirective,
            ngOptions: ngOptionsDirective,
            ngTransclude: ngTranscludeDirective,
            ngModel: ngModelDirective,
            ngList: ngListDirective,
            ngChange: ngChangeDirective,
            pattern: patternDirective,
            ngPattern: patternDirective,
            required: requiredDirective,
            ngRequired: requiredDirective,
            minlength: minlengthDirective,
            ngMinlength: minlengthDirective,
            maxlength: maxlengthDirective,
            ngMaxlength: maxlengthDirective,
            ngValue: ngValueDirective,
            ngModelOptions: ngModelOptionsDirective
        }).
        directive({
          ngInclude: ngIncludeFillContentDirective
        }).
        directive(ngAttributeAliasDirectives).
        directive(ngEventDirectives);
      $provide.provider({
        $anchorScroll: $AnchorScrollProvider,
        $animate: $AnimateProvider,
        $animateCss: $CoreAnimateCssProvider,
        $$animateJs: $$CoreAnimateJsProvider,
        $$animateQueue: $$CoreAnimateQueueProvider,
        $$AnimateRunner: $$AnimateRunnerFactoryProvider,
        $$animateAsyncRun: $$AnimateAsyncRunFactoryProvider,
        $browser: $BrowserProvider,
        $cacheFactory: $CacheFactoryProvider,
        $controller: $ControllerProvider,
        $document: $DocumentProvider,
        $$isDocumentHidden: $$IsDocumentHiddenProvider,
        $exceptionHandler: $ExceptionHandlerProvider,
        $filter: $FilterProvider,
        $$forceReflow: $$ForceReflowProvider,
        $interpolate: $InterpolateProvider,
        $interval: $IntervalProvider,
        $http: $HttpProvider,
        $httpParamSerializer: $HttpParamSerializerProvider,
        $httpParamSerializerJQLike: $HttpParamSerializerJQLikeProvider,
        $httpBackend: $HttpBackendProvider,
        $xhrFactory: $xhrFactoryProvider,
        $jsonpCallbacks: $jsonpCallbacksProvider,
        $location: $LocationProvider,
        $log: $LogProvider,
        $parse: $ParseProvider,
        $rootScope: $RootScopeProvider,
        $q: $QProvider,
        $$q: $$QProvider,
        $sce: $SceProvider,
        $sceDelegate: $SceDelegateProvider,
        $sniffer: $SnifferProvider,
        $templateCache: $TemplateCacheProvider,
        $templateRequest: $TemplateRequestProvider,
        $$testability: $$TestabilityProvider,
        $timeout: $TimeoutProvider,
        $window: $WindowProvider,
        $$rAF: $$RAFProvider,
        $$jqLite: $$jqLiteProvider,
        $$HashMap: $$HashMapProvider,
        $$cookieReader: $$CookieReaderProvider
      });
    }
  ]);
```
按我们之前的理解，`provider`方法是传入一个name和一个factory方法进行定义，不过我们之前有一个不理解用途的方法`supportObject`,
从此处的`provider({...})`便能猜想出用途了。

即传入一个对象，可以进行批量的`provider`注册。

至此，我们基本上已经理解了所有的依赖注入问题。
```
    var runBlocks = loadModules(modulesToLoad);
    //省略
    forEach(runBlocks, function(fn) { if (fn) instanceInjector.invoke(fn); });
```
这里将最终的runBlocks进行forEach然后invoke，关于runBlocks
```
        if (isString(module)) {
          moduleFn = angularModule(module);
          //这里递归加载所有依赖的模块
          //runBlocks是一个局部变量，每一次loadModules，都是一个新的runBlocks，所以requires会被放在前面
          runBlocks = runBlocks.concat(loadModules(moduleFn.requires)).concat(moduleFn._runBlocks);
          //执行invoke队列
          runInvokeQueue(moduleFn._invokeQueue);
          runInvokeQueue(moduleFn._configBlocks);
        } else if (isFunction(module)) {
            runBlocks.push(providerInjector.invoke(module));
        } else if (isArray(module)) {
            runBlocks.push(providerInjector.invoke(module));
        } else {
          assertArgFn(module, 'module');
        }
```
这里module会是Function或者Array。
来看看modulesToLoad是怎样的传入形式，在injector中，是通过
```
    function createInjector(modulesToLoad, strictDi)
```
进行传入的，进行该函数的调用检索，在`'Angular.js'`中
```
    function bootstrap(element, modules, config) {
        if (!isObject(config)) config = {};
        var defaultConfig = {...};
        config = extend(defaultConfig, config);
        var doBootstrap = function() {
        element = jqLite(element);
        
        if (element.injector()) {...}
        
        modules = modules || [];
        modules.unshift(['$provide', function($provide) {
          $provide.value('$rootElement', element);
        }]);
        
        if (config.debugInfoEnabled) {...}
        
        modules.unshift('ng');
        //这里调用了createInjector
        var injector = createInjector(modules, config.strictDi);
        injector.invoke(['$rootScope', '$rootElement', '$compile', '$injector',
           function bootstrapApply(scope, element, compile, injector) {
            scope.$apply(function() {
              element.data('$injector', injector);
              compile(element)(scope);
            });
          }]
        );
        return injector;
    }
```
来看看bootstrap的调用例子
```
    var app = angular.module('demo', [])
      .controller('WelcomeController', function($scope) {
          $scope.greeting = 'Welcome!';
      });
      angular.bootstrap(document, ['demo']);
```
目前只看到`module`传入`Array<String>`的例子