# cordova.js总结

### cordova声明周期

cordova 一共有一下声明周期

1. onDOMContentLoaded
2. onNativeReady
3. onCordovaReady
4. onPluginsReady
5. onDeviceReady



> onDOMContentLoaded  当dom加载完毕之后调用 (当document调用DOMContentLoaded或者ocument.readyState是complete或者interactive时候调用.)
>
> onNativeReady 代表native依赖module加载完毕
>
> onPluginsReady 代表我们依赖的插件加载完毕
>
> onCordovaReady 当deviceready 和onPluginsReady 状态都加载完毕,执行该函数
>
> onDeviceReady  当onDOMContentLoaded 和onCordovaReady 都加载完毕执行该函数

声明周期如下图



![image-20190612183423939](image-20190612183423939.png)







