# cordova.js 文件分析

为啥上来先分析这个类呢?其实是因为我们创建的index.html文件中只对该文件做了引用,因此猜测,该文件应该是cordova的入口文件了.

# 文件解析



### 整体语法

```js
;(function() {

})()
```

这个是js自执行语法 .大神解释———给自执行的代码块的中的变量保护,防止变量污染.



###  声明变量

```js
var PLATFORM_VERSION_BUILD_LABEL = '5.0.1';
var require;
var define;

```

这里声明三个变量

### 自执行程序

```js
(**function** () {

​    **var** modules = {};

​    *// Stack of moduleIds currently being built.*

​    **var** requireStack = [];

​    *// Map of module ID -> index into requireStack of modules currently being built.*

​    **var** inProgressModules = {};

​    **var** SEPARATOR = '.';



​    **function** build (module) {

​        **var** factory = module.factory;

​        **var** localRequire = **function** (id) {

​            **var** resultantId = id;

​            *// Its a relative path, so lop off the last portion and add the id (minus "./")*

​            **if** (id.charAt(0) === '.') {

​                resultantId = module.id.slice(0, module.id.lastIndexOf(SEPARATOR)) + SEPARATOR + id.slice(2);

​            }

​            **return** require(resultantId);

​        };

​        module.exports = {};

​        **delete** module.factory;

​        factory(localRequire, module.exports, module);

​        **return** module.exports;

​    }



​    require = **function** (id) {

​        **if** (!modules[id]) {

​            **throw** 'module ' + id + ' not found';

​        } **else** **if** (id **in** inProgressModules) {

​            **var** cycle = requireStack.slice(inProgressModules[id]).join('->') + '->' + id;

​            **throw** 'Cycle in require graph: ' + cycle;

​        }

​        **if** (modules[id].factory) {

​            **try** {

​                inProgressModules[id] = requireStack.length;

​                requireStack.push(id);

​                **return** build(modules[id]);

​            } **finally** {

​                **delete** inProgressModules[id];

​                requireStack.pop();

​            }

​        }

​        **return** modules[id].exports;

​    };



​    define = **function** (id, factory) {

​        **if** (modules[id]) {

​            **throw** 'module ' + id + ' already defined';

​        }



​        modules[id] = {

​            id: id,

​            factory: factory

​        };

​    };



​    define.remove = **function** (id) {

​        **delete** modules[id];

​    };



​    define.moduleMap = modules;

})();
```



