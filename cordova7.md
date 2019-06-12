

cordova.js 文件分析(7)

### cordova/argscheck 从factory 变成 exports

该module 目前没有在初始化的过程中从factory 变成exports

```js
var utils = require('cordova/utils');
```

引用utils工具

```js
var moduleExports = module.exports;
```

声明变量指定到module.exports

```js
var typeMap = {
    'A': 'Array',
    'D': 'Date',
    'N': 'Number',
    'S': 'String',
    'F': 'Function',
    'O': 'Object'
};

```

声明类型匹配变量

```js
function extractParamName (callee, argIndex) {
    return (/.*?\((.*?)\)/).exec(callee)[1].split(', ')[argIndex];
}
```

正则表达式

> exec() 方法是一个正则表达式方法。
>
> exec() 方法用于检索字符串中的正则表达式的匹配。
>
> 该函数返回一个数组，其中存放匹配的结果。如果未找到匹配，则返回值为 null。



> 前后的"/"代表正则的起始和结束位置
>
> .*?   表示匹配任意字符到下一个符合条件的字符 举例 正则表达式a.*?xxx   可以匹配 abxxx  axxxxx  abbbbbxxx
>
> \\( 匹配左边括号
>
> \\) 匹配右边括号
>
> ( )  标记一个子表达式的开始和结束位置。子表达式可以获取供以后使用

因此这个表达式的含义是匹配括号格式 * (*) 

```js
function checkArgs (spec, functionName, args, opt_callee) {
    if (!moduleExports.enableChecks) {
        return;
    }
    var errMsg = null;
    var typeName;
    for (var i = 0; i < spec.length; ++i) {
        var c = spec.charAt(i);
        var cUpper = c.toUpperCase();
        var arg = args[i];
        // Asterix means allow anything.
        if (c === '*') {
            continue;
        }
        typeName = utils.typeName(arg);
        if ((arg === null || arg === undefined) && c === cUpper) {
            continue;
        }
        if (typeName !== typeMap[cUpper]) {
            errMsg = 'Expected ' + typeMap[cUpper];
            break;
        }
    }
    if (errMsg) {
        errMsg += ', but got ' + typeName + '.';
        errMsg = 'Wrong type for parameter "' + extractParamName(opt_callee || args.callee, i) + '" of ' + functionName + ': ' + errMsg;
        // Don't log when running unit tests.
        if (typeof jasmine === 'undefined') {
            console.error(errMsg);
        }
        throw TypeError(errMsg);
    }
}


```

1. 是否开启参数检查
2. 巡查检查args参数类型和spec指定的类型是否匹配.不匹配就抛出异常

```js
function getValue (value, defaultValue) {
    return value === undefined ? defaultValue : value;
}
```

指定默认值,在没有值的情况下返回默认值

```js
moduleExports.checkArgs = checkArgs;
moduleExports.getValue = getValue;
moduleExports.enableChecks = true;
```

参数的检查是默认开启的,这里我们也可以关闭



### cordova/base64  从factory 变成 exports

这个module 在调用exec命令的时候使用了.是用来将ArrayBuffer 进行编码的.  ArrayBuffer 在不同的设备可能没办法直接使用

```js
var base64 = exports;
```

指定module

```js
base64.fromArrayBuffer = function (arrayBuffer) {
    var array = new Uint8Array(arrayBuffer);
    return uint8ToBase64(array);
};
```

该函数就是讲ArrayBuffer转换成base64

**ArrayBuffer** 对象用来表示通用的、固定长度的原始二进制数据缓冲区.`ArrayBuffer` 不能直接操作，而是要通过[类型数组对象](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/TypedArray)或 [`DataView`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/DataView) 对象来操作，它们会将缓冲区中的数据表示为特定的格式，并通过这些格式来读写缓冲区的内容。

可以理解将二进制转换成base64



```js
base64.toArrayBuffer = function (str) {
    var decodedStr = typeof atob !== 'undefined' ? atob(str) : Buffer.from(str, 'base64').toString('binary'); // eslint-disable-line no-undef
    var arrayBuffer = new ArrayBuffer(decodedStr.length);
    var array = new Uint8Array(arrayBuffer);
    for (var i = 0, len = decodedStr.length; i < len; i++) {
        array[i] = decodedStr.charCodeAt(i);
    }
    return arrayBuffer;
};

```

将base64 转变成ArrayBuffer(二进制)

> **WindowOrWorkerGlobalScope.atob()** 对经过 base-64 编码的字符串进行解码。
>
> 你可以使用 [`window.btoa()`](https://developer.mozilla.org/zh-CN/docs/Web/API/WindowBase64/btoa) 方法来编码一个可能在传输过程中出现问题的数据.

要是有atob命令就直接使用将其转变成二进制文件,否则就使用Buffer的from命令



```js
var b64_6bit = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/';
var b64_12bit;

var b64_12bitTable = function () {
    b64_12bit = [];
    for (var i = 0; i < 64; i++) {
        for (var j = 0; j < 64; j++) {
            b64_12bit[i * 64 + j] = b64_6bit[i] + b64_6bit[j];
        }
    }
    b64_12bitTable = function () { return b64_12bit; };
    return b64_12bit;
};

```

base64 表格

```js
function uint8ToBase64 (rawData) {
    var numBytes = rawData.byteLength;
    var output = '';
    var segment;
    var table = b64_12bitTable();
    for (var i = 0; i < numBytes - 2; i += 3) {
        segment = (rawData[i] << 16) + (rawData[i + 1] << 8) + rawData[i + 2];
        output += table[segment >> 12];
        output += table[segment & 0xfff];
    }
    if (numBytes - i === 2) {
        segment = (rawData[i] << 16) + (rawData[i + 1] << 8);
        output += table[segment >> 12];
        output += b64_6bit[(segment & 0xfff) >> 6];
        output += '=';
    } else if (numBytes - i === 1) {
        segment = (rawData[i] << 16);
        output += table[segment >> 12];
        output += '==';
    }
    return output;
}

```

八进制转变成64

这里不想讲解base64 原理 [原理参考](https://www.cnblogs.com/diligenceday/p/6002382.html)

### cordova/exec/proxy 从factory 变成 exports

```js
var CommandProxyMap = {};
```

声明变量

```js
module.exports = {

    // example: cordova.commandProxy.add("Accelerometer",{getCurrentAcceleration: function(successCallback, errorCallback, options) {...},...);
    add: function (id, proxyObj) {
        console.log('adding proxy for ' + id);
        CommandProxyMap[id] = proxyObj;
        return proxyObj;
    },

    // cordova.commandProxy.remove("Accelerometer");
    remove: function (id) {
        var proxy = CommandProxyMap[id];
        delete CommandProxyMap[id];
        CommandProxyMap[id] = null;
        return proxy;
    },

    get: function (service, action) {
        return (CommandProxyMap[service] ? CommandProxyMap[service][action] : null);
    }
};
```

很类似map,有增删改查

### cordova/plugin/ios/console 从从factory 变成 exports

这里其实就是将window的console 改变成下面的module

```js
var logger = require('cordova/plugin/ios/logger');
```

依赖module

```js
var console = module.exports;
```

声明module ,该变量在cordova/init的时候加载到window.console上(其实就是修改默认的window.console)

```js
var WinConsole = window.console;
```

获取原生的WinConsole

```js
var UseLogger = false;
```

默认不开启用户logger

```js
var Timers = {};
```

timer 对象

```js
function noop() {}
```

声明一个函数,具体作用未知

```js
console.useLogger = function (value) {
    if (arguments.length) UseLogger = !!value;

    if (UseLogger) {
        if (logger.useConsole()) {
            throw new Error("console and logger are too intertwingly");
        }
    }

    return UseLogger;
};
```

定义useLogger

```js
console.log = function() {
    if (logger.useConsole()) return;
    logger.log.apply(logger, [].slice.call(arguments));
};

//------------------------------------------------------------------------------
console.error = function() {
    if (logger.useConsole()) return;
    logger.error.apply(logger, [].slice.call(arguments));
};

//------------------------------------------------------------------------------
console.warn = function() {
    if (logger.useConsole()) return;
    logger.warn.apply(logger, [].slice.call(arguments));
};

//------------------------------------------------------------------------------
console.info = function() {
    if (logger.useConsole()) return;
    logger.info.apply(logger, [].slice.call(arguments));
};

//------------------------------------------------------------------------------
console.debug = function() {
    if (logger.useConsole()) return;
    logger.debug.apply(logger, [].slice.call(arguments));
};

//------------------------------------------------------------------------------
console.assert = function(expression) {
    if (expression) return;

    var message = logger.format.apply(logger.format, [].slice.call(arguments, 1));
    console.log("ASSERT: " + message);
};

//------------------------------------------------------------------------------
console.clear = function() {};

//------------------------------------------------------------------------------
console.dir = function(object) {
    console.log("%o", object);
};

//------------------------------------------------------------------------------
console.dirxml = function(node) {
    console.log(node.innerHTML);
};
//------------------------------------------------------------------------------
console.trace = noop;

//------------------------------------------------------------------------------
console.group = console.log;

//------------------------------------------------------------------------------
console.groupCollapsed = console.log;

//------------------------------------------------------------------------------
console.groupEnd = noop;

```

重写相关函数

```js
//------------------------------------------------------------------------------
console.time = function(name) {
    Timers[name] = new Date().valueOf();
};

//------------------------------------------------------------------------------
console.timeEnd = function(name) {
    var timeStart = Timers[name];
    if (!timeStart) {
        console.warn("unknown timer: " + name);
        return;
    }

    var timeElapsed = new Date().valueOf() - timeStart;
    console.log(name + ": " + timeElapsed + "ms");
};

```

增加两个事件相关函数  Timers 用来记录事件开始的时间的

```js
console.timeStamp = noop;

//------------------------------------------------------------------------------
console.profile = noop;

//------------------------------------------------------------------------------
console.profileEnd = noop;

//------------------------------------------------------------------------------
console.count = noop;

//------------------------------------------------------------------------------
console.exception = console.log;

//------------------------------------------------------------------------------
console.table = function(data, columns) {
    console.log("%o", data);
};

```

未知作用

```
function wrappedOrigCall(orgFunc, newFunc) {
    return function() {
        var args = [].slice.call(arguments);
        try { orgFunc.apply(WinConsole, args); } catch (e) {}
        try { newFunc.apply(console,    args); } catch (e) {}
    };
}

```

选用新的console 还是老的console 打印

```js
for (var key in console) {
    if (typeof WinConsole[key] == "function") {
        console[key] = wrappedOrigCall(WinConsole[key], console[key]);
    }
}
```

这里是重新更改console[key]的定义,在新定义的console 和原始WinConsole

### cordova/plugin/ios/logger 从factory 变成 exports

这里略过

### cordova/urlutil 从factory 变成 exports

```js
exports.makeAbsolute = function makeAbsolute (url) {
    var anchorEl = document.createElement('a');
    anchorEl.href = url;
    return anchorEl.href;
};

```

获取绝对路径