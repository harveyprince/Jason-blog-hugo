+++
tags = [
    "analyze",
    'source code',
    'AngularJS'
]
image = ""
math = false
date = "2016-12-26T20:30:26+08:00"
title = "Angular1 Source Code Injector"

+++

基于版本：1.6.0

## Injector
---
从整体的概念解读中，我们可以看出，AngularJS中很重要的一个依赖处理的概念就是Injector。
它很好地完成了所有依赖的管理，让我们从繁琐的对象创建工作中脱离出来，更关注功能实现。
从angularFiles.js可以看出Injector定义在`'src/auto/injector.js'`中。
<!--more-->

## Injector是怎么工作的
---
在这里不得不提一下js中一个方法——`toString()`，显而易见，这是一个将对象自身转化为字符串的函数，
在js里，`function`也可以这么做，举一个简单的例子

<embed width="100%" height="500" src="https://embed.plnkr.co/M9gqs6cV6AYEAPjfA3WN/" />

function通过toString可以将自己的定义完全转换成字符串，这样子就可以提取其中的参数名了。

这里正则表达式派上了用场，看看injector.js中的函数定义
```
    var ARROW_ARG = /^([^(]+?)=>/;
    var FN_ARGS = /^[^(]*\(\s*([^)]*)\)/m;
    var FN_ARG_SPLIT = /,/;
    var FN_ARG = /^\s*(_?)(\S+?)\1\s*$/;
    var STRIP_COMMENTS = /((\/\/.*$)|(\/\*[\s\S]*?\*\/))/mg;
    function stringifyFn(fn) {
      // Support: Chrome 50-51 only
      // Creating a new string by adding `' '` at the end, to hack around some bug in Chrome v50/51
      // (See https://github.com/angular/angular.js/issues/14487.)
      // TODO (gkalpak): Remove workaround when Chrome v52 is released
      return Function.prototype.toString.call(fn) + ' ';
    }
    
    function extractArgs(fn) {
      var fnText = stringifyFn(fn).replace(STRIP_COMMENTS, ''),
          args = fnText.match(ARROW_ARG) || fnText.match(FN_ARGS);
      return args;
    }

```
> 这里提到的Chrome v50/51 bug是针对arrow形式的函数`(arg0,arg1)=>{}`调用toString产生的问题的，新的chrome版本已经解决，不深究

我们来测试一下上面的函数

<embed width="100%" height="500" src="https://embed.plnkr.co/mTTVnTwX8298ciXZCjoS/" />

可以看到，所需的参数都得到了初步的提取，通过split就能拿到一个arguments的array

## $inject
---
其实我们可以直接通过给`function`增加`$inject`来声明其中的依赖，例如
```
    function SomeCtrl ($scope) {
    
    }
    //方式1
    SomeCtrl.$inject = ['$scope'];
    angular
        .module('app', [])
        .controller('SomeCtrl', SomeCtrl);
      
    //方式2
    angular
        .module('app', [])
        .controller('SomeCtrl', SomeCtrl);
        
    //方式3
    angular
        .module('app', [])
        .controller('SomeCtrl', ['$scope',SomeCtrl]);
          
```
相比通过转化为字符串提取出arguments，方式1可以节省Angular很多的工作，
我们来看看Angular是如何导出`$inject`的就明白是怎么回事了
```
    function annotate(fn, strictDi, name) {
      var $inject,
          argDecl,
          last;
    
      if (typeof fn === 'function') {
        if (!($inject = fn.$inject)) {
          $inject = [];
          if (fn.length) {
            if (strictDi) {
              if (!isString(name) || !name) {
                name = fn.name || anonFn(fn);
              }
              throw $injectorMinErr('strictdi',
                '{0} is not using explicit annotation and cannot be invoked in strict mode', name);
            }
            argDecl = extractArgs(fn);
            forEach(argDecl[1].split(FN_ARG_SPLIT), function(arg) {
              arg.replace(FN_ARG, function(all, underscore, name) {//此处为了去处空格等字符
                $inject.push(name);
              });
            });
          }
          fn.$inject = $inject;//缓存起来，防止重复进行耗时的操作
        }
      } else if (isArray(fn)) {
        last = fn.length - 1;
        assertArgFn(fn[last], 'fn');
        $inject = fn.slice(0, last);
      } else {
        assertArgFn(fn, 'fn', true);
      }
      return $inject;
    }

```
可以看出，非array并且不声明`$inject`的function在解析上会有一定的性能消耗，
并且Angular也意识到了这一点，尽量避免重复这项操作

## 再来看看Injector
---
在injector.js中，实现了一个createInjector的方法来封装injector
```
    function createInjector(modulesToLoad, strictDi) {
        strictDi = (strictDi === true);
        var INSTANTIATING = {},
          providerSuffix = 'Provider',
          path = [],
          loadedModules = new HashMap([], true),
          providerCache = {
            $provide: {
                provider: supportObject(provider),
                factory: supportObject(factory),
                service: supportObject(service),
                value: supportObject(value),
                constant: supportObject(constant),
                decorator: decorator
              }
          },
          providerInjector = (providerCache.$injector =
              createInternalInjector(providerCache, function(serviceName, caller) {
                if (angular.isString(caller)) {
                  path.push(caller);
                }
                throw $injectorMinErr('unpr', 'Unknown provider: {0}', path.join(' <- '));
              })),
          instanceCache = {},
          protoInstanceInjector =
              createInternalInjector(instanceCache, function(serviceName, caller) {
                var provider = providerInjector.get(serviceName + providerSuffix, caller);
                return instanceInjector.invoke(
                    provider.$get, provider, undefined, serviceName);
              }),
          instanceInjector = protoInstanceInjector;
        
        providerCache['$injector' + providerSuffix] = { $get: valueFn(protoInstanceInjector) };
        var runBlocks = loadModules(modulesToLoad);
        instanceInjector = protoInstanceInjector.get('$injector');
        instanceInjector.strictDi = strictDi;
        forEach(runBlocks, function(fn) { if (fn) instanceInjector.invoke(fn); });
        
        return instanceInjector;
        //省略函数定义
        
    }

```
我们看看`providerCache`，其中形如`provider: supportObject(provider)`，后一个provider是后边定义的`function provider(name, provider_) {...}`，
```
    function supportObject(delegate) {
        return function(key, value) {
          if (isObject(key)) {
            forEach(key, reverseParams(delegate));
          } else {
            return delegate(key, value);
          }
        };
      }
```
这里当key不为object时进行正常调用，为object时进行了一个处理，但是这一处理目前我还没遇到过适用场景。

不过看得出来，这里有一个很重要的概念`provider`。

Angular中需要对象协作来完成任务，这些对象是通过`injector service`初始化创建并协同工作的。
injector创建了两种类型的对象，`services`和`specialized objects`。
service是由编写该service的开发人员定义api的对象，
Specialized objects是符合特定Angular框架api的对象，这些对象是controllers, directives, filters 或者 animations。
injector需要知道如何创建这些对象，我们是通过注册一个recipe来使用injector创建对象的，目前有五种recipe类型。
其中最重要的是Provider recipe，其他的Value, Factory, Service 和 Constant仅仅是provider的语法糖。

## 简单认识一下这些recipe
---
### Value recipe
有时候我们需要一个很简单的提供基本数值的服务
```
    var myApp = angular.module('myApp', []);
    myApp.value('clientId', 'a12345654321x');
    //最终可以通过依赖注入的形式获取到这个值
    myApp.controller('DemoController', ['clientId', function DemoController(clientId) {
      this.clientId = clientId;
    }]);
```
关于value的函数定义
```
    function value(name, val) { return factory(name, valueFn(val), false); }
    //其中
    function valueFn(value) {return function valueRef() {return value;};}

```
### Factory recipe
value过于简单，有时候我们需要复杂一点的功能来为我们提供服务，factory可以做到

* 可以使用其他services（拥有依赖）
* service初始化
* 延迟／懒 初始化

> 注意一点，Angular中所有的service都是单例

```
    myApp.factory('clientId', function clientIdFactory() {
      return 'a12345654321x';
    });
    //进一步可以
    myApp.factory('apiToken', ['clientId', function apiTokenFactory(clientId) {
      var encrypt = function(data1, data2) {
        // NSA-proof encryption algorithm:
        return (data1 + ':' + data2).toUpperCase();
      };
    
      var secret = window.localStorage.getItem('myApp.secret');
      var apiToken = encrypt(clientId, secret);
    
      return apiToken;
    }]);
```
关于factory的函数定义
```
    function factory(name, factoryFn, enforce) {
        return provider(name, {
          $get: enforce !== false ? enforceReturnValue(name, factoryFn) : factoryFn
        });
      }

```
### Service recipe
Javascript开发者通常会使用现有的类型来进行面向对象编程，我们可以通过以下两种方式来实现面向对象
```
    function UnicornLauncher(apiToken) {
    
      this.launchedCount = 0;
      this.launch = function() {
        // Make a request to the remote API and include the apiToken
        ...
        this.launchedCount++;
      }
    }
    //方式一
    myApp.factory('unicornLauncher', ["apiToken", function(apiToken) {
      return new UnicornLauncher(apiToken);
    }]);
    //方式二
    myApp.service('unicornLauncher', ["apiToken", UnicornLauncher]);
```
关于service的函数定义
```
    function service(name, constructor) {
        return factory(name, ['$injector', function($injector) {
          return $injector.instantiate(constructor);
        }]);
      }
```
### Provider recipe
正如之前所提到过的，provider recipe是最核心的recipe，它能实现最全面的功能，但是对于大多数service来说功能上有点过了。
provider recipe实现了一个$get方法，这个方法是一个工厂方法，正如我们在使用factory recipe时一样，而事实上，当我们在定义
一个factory recipe的时候，一个拥有$get方法的空的provider类型被创建了。

仅仅当你想要在应用程序启动之前进行配置并对应用程序提供api时，才应该使用provider recipe。
```
    myApp.provider('unicornLauncher', function UnicornLauncherProvider() {
      var useTinfoilShielding = false;
    
      this.useTinfoilShielding = function(value) {
        useTinfoilShielding = !!value;
      };
    
      this.$get = ["apiToken", function unicornLauncherFactory(apiToken) {
    
        // let's assume that the UnicornLauncher constructor was also changed to
        // accept and use the useTinfoilShielding argument
        return new UnicornLauncher(apiToken, useTinfoilShielding);
      }];
    });
    //在配置阶段
    myApp.config(["unicornLauncherProvider", function(unicornLauncherProvider) {
      unicornLauncherProvider.useTinfoilShielding(true);
    }]);
```
关于provider的函数定义
```
    function provider(name, provider_) {
        assertNotHasOwnProperty(name, 'service');
        if (isFunction(provider_) || isArray(provider_)) {
          provider_ = providerInjector.instantiate(provider_);
        }
        if (!provider_.$get) {
          throw $injectorMinErr('pget', 'Provider \'{0}\' must define $get factory method.', name);
        }
        return (providerCache[name + providerSuffix] = provider_);
      }

```
### Constant recipe
Angular的生命周期包含配置阶段和运行阶段，在配置阶段的时候没有service是available的，
所以无法在配置阶段引入任何一个service，甚至是简单的value recipe也不行。
然而仍然有一些需要在这个时期使用的，比如url的前缀等等一系列不需要依赖的常量。
这个时候我们就可以使用constant，他和value的区别就是constant还可以在配置阶段使用。
```
    myApp.constant('planetName', 'Greasy Giant');
    //配置阶段
    myApp.config(['unicornLauncherProvider', 'planetName', function(unicornLauncherProvider, planetName) {
      unicornLauncherProvider.useTinfoilShielding(true);
      unicornLauncherProvider.stampText(planetName);
    }]);
    //运行阶段
    myApp.controller('DemoController', ["clientId", "planetName", function DemoController(clientId, planetName) {
      this.clientId = clientId;
      this.planetName = planetName;
    }]);
```
关于constant的函数定义
```
    function constant(name, value) {
        assertNotHasOwnProperty(name, 'constant');
        providerCache[name] = value;
        instanceCache[name] = value;
      }
```
可以看出，constant在调用之后立马加入了缓存中。
### Special Purpose Objects
前面我们提到过还有一些有别于services的Specialized objects，这些对象以插件的形式扩展框架，因此必须实现Angular框架声明的接口。
这些接口就是Controller, Directive, Filter 和 Animation。
injector在创建这些对象（除了Controller）的时候，使用了Factory的形式，即需要显式地返回一个对象。
而Controller使用了构造器的形式。

## 回到createInjector
---
通过分析可以知道providerCache的结构
```
    {
        $provide: {
            provider: func,
            factory: func,
            service: func,
            value: func,
            constant: func,
            decorator: func
        },
        $injector: obj,
        $injectorProvider: obj
        //还有用户定义的所有provider，都以name+'Provider'为key缓存起来了
        //还有用户定义的constant，都以name为key缓存
    }

```
createInjector最终返回了一个instanceInjector
```
    providerInjector = (providerCache.$injector =
              createInternalInjector(providerCache, function(serviceName, caller) {
                if (angular.isString(caller)) {
                  path.push(caller);
                }
                throw $injectorMinErr('unpr', 'Unknown provider: {0}', path.join(' <- '));
              })),
    instanceCache = {},
    protoInstanceInjector =
              createInternalInjector(instanceCache, function(serviceName, caller) {
                var provider = providerInjector.get(serviceName + providerSuffix, caller);
                return instanceInjector.invoke(
                    provider.$get, provider, undefined, serviceName);
              }),
    instanceInjector = protoInstanceInjector;
```
关于createInternalInjector的定义
```
    function createInternalInjector(cache, factory) {
        //省略函数定义
        return {
          invoke: invoke,
          instantiate: instantiate,
          get: getService,
          annotate: createInjector.$$annotate,
          has: function(name) {
            return providerCache.hasOwnProperty(name + providerSuffix) || cache.hasOwnProperty(name);
          }
        };
      }
    createInjector.$$annotate = annotate;
```
传入的cache和factory主要用在了getService里
```
    function getService(serviceName, caller) {
      if (cache.hasOwnProperty(serviceName)) {
        if (cache[serviceName] === INSTANTIATING) {
          throw $injectorMinErr('cdep', 'Circular dependency found: {0}',
                    serviceName + ' <- ' + path.join(' <- '));
        }
        return cache[serviceName];
      } else {
        try {
          path.unshift(serviceName);
          cache[serviceName] = INSTANTIATING;
          cache[serviceName] = factory(serviceName, caller);
          return cache[serviceName];
        } catch (err) {
          if (cache[serviceName] === INSTANTIATING) {
            delete cache[serviceName];
          }
          throw err;
        } finally {
          path.shift();
        }
      }
    }
```
从cache中获取service，并且检测依赖的回环和provider不存在的情况。

从constant recipe可以知道，instanceCache里目前只缓存了constant。
protoInstanceInjector在调用get时，先从instanceCache检测，不存在定义时
通过providerInjector来从providerCache里拿相应的provider。
在获取provider之后调用了invoke方法
```
    return instanceInjector.invoke(
                    provider.$get, provider, undefined, serviceName);
```
来看一下invoke的定义
```
    function injectionArgs(fn, locals, serviceName) {
      var args = [],
          $inject = createInjector.$$annotate(fn, strictDi, serviceName);

      for (var i = 0, length = $inject.length; i < length; i++) {
        var key = $inject[i];
        if (typeof key !== 'string') {
          throw $injectorMinErr('itkn',
                  'Incorrect injection token! Expected service name as string, got {0}', key);
        }
        //这里最终返回的args是$inject对应的provider对象的列表
        args.push(locals && locals.hasOwnProperty(key) ? locals[key] :
                                                         getService(key, serviceName));
      }
      return args;
    }
    function invoke(fn, self, locals, serviceName) {
      if (typeof locals === 'string') {
        serviceName = locals;
        locals = null;
      }

      var args = injectionArgs(fn, locals, serviceName);
      if (isArray(fn)) {
        fn = fn[fn.length - 1];
      }

      if (!isClass(fn)) {
        // http://jsperf.com/angularjs-invoke-apply-vs-switch
        // #5388
        return fn.apply(self, args);
      } else {
        args.unshift(null);
        //这里有一点绕，args中unshift一个null是用于占thisArg的位，由于new操作符，该参数无效
        //以下相当于以fn为构造函数进行new操作，并将不带null的args spread开作为参数传递
        //在instantiate中也有类似的方式
        return new (Function.prototype.bind.apply(fn, args))();
      }
    }
```
所以可以看出，invoke将会得到一个切实可用的service，并在获取的过程中进行了缓存以及将所有的需求的依赖进行注入。

整理一下流程就是，instanceInjector通过调用get，可以从instanceCache中获取已经初始化的service，如果instanceCache中没有，
就从providerCache中获取其provider定义，然后初始化再缓存至instanceCache中，并返回初始化好的service。

## 再回到createInjector
---
现在，我们已经弄清楚了injector的工作原理了，在返回instanceInjector之前，还调用了一句
```
    var runBlocks = loadModules(modulesToLoad);
    //省略
    forEach(runBlocks, function(fn) { if (fn) instanceInjector.invoke(fn); });
```
显而易见这是将所有需要加载的module进行了invoke，我们来看看loadModules的定义
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
可以看出，涉及到了很多angularModule方面的概念，其相关定义都在loader.js中，在module相关章节展开时再做解读，
包括modulesToLoad的格式，module的invoke。

