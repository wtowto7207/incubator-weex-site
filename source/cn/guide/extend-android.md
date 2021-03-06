---
title: Android 扩展
type: guide
group: 扩展
order: 6.2
version: 2.1
---

# Android 扩展

Weex 提供了扩展机制，可以根据自己的业务进行定制自己的功能。
主要分为两类扩展：

- Module 扩展 非 UI 的特定功能。例如 sendHttp、openURL 等。
- Component 扩展 实现特别功能的 Native 控件。例如：RichTextview，RefreshListview 等。
- Adapter 扩展 Weex 对一些基础功能实现了统一的接口，可实现这些接口来定制自己的业务。例如：图片下载等。

## Module 扩展

1. Module 扩展必须继承 WXModule 类。
2. 扩展方法必须加上`@JSMethod (uiThread = false or true)` 注解。Weex 会根据注解来判断当前方法是否要运行在 UI 线程，和当前方法是否是扩展方法。
3. Weex是根据反射来进行调用 Module 扩展方法，所以Module中的扩展方法必须是 public 类型。
4. 同样因为是通过反射调用，Module 不能被混淆。请在混淆文件中添加代码：`-keep public class * extends com.taobao.weex.common.WXModule{*;}`
5. Module 扩展的方法可以使用 int, double, float, String, Map, List 类型的参数
6. 完成 Module 后一定要在初始化时注册 `WXSDKEngine.registerModule("myModule", MyModule.class);` 否则会报类似错误：`ReportException :undefined:9: TypeError: Object #<Object> has no method 'printLog'`

示例如下：

```java
public class MyModule extends WXModule {

  //run ui thread
  @JSMethod (uiThread = true)
  public void printLog(String msg) {
    Toast.makeText(mWXSDKInstance.getContext(),msg,Toast.LENGTH_SHORT).show();
  }

  //run JS thread
  @JSMethod (uiThread = false)
  public void fireEventSyncCall(){
   //implement your module logic here
  }
}
```
Register the module

```java
WXSDKEngine.registerModule("MyModule", MyModule.class);
```
JS 调用如下：

```html
<template>
  <div>
    <text onclick="click">testMyModule</text>
  </div>
</template>

<script>
  module.exports = {
    methods: {
      click: function() {
        weex.requireModule('MyModule').printLog("I am a weex Module");
      }
    }
  }
</script>
```
## Component 扩展

1. Component 扩展类必须继承 WXComponent.
2. Component 对应的设置属性的方法必须添加注解 @WXComponentProp(name=value(value is attr or style of dsl))
3. Weex sdk 通过反射调用对应的方法，所以 Component 对应的属性方法必须是 public，并且不能被混淆。请在混淆文件中添加代码  `-keep public class * extends com.taobao.weex.ui.component.WXComponent{*;}`
4. Component 扩展的方法可以使用 int, double, float, String, Map, List 类型的参数
5. 完成 Component 后一定要在初始化时注册 `WXSDKEngine.registerComponent("richText", RichText.class);`

示例如下:

```java
public class RichText extends WXComponent<TextView> {

    public RichText(WXSDKInstance instance, WXDomObject dom, WXVContainer parent) {
        super(instance, dom, parent);
    }

    @Override
    protected TextView initComponentHostView(@NonNull Context context) {
        TextView textView = new TextView(context);
        textView.setTextSize(20);
        textView.setTextColor(Color.BLACK);
        return textView;
    }

    @WXComponentProp(name = "tel")
    public void setTel(String telNumber) {
        getHostView().setText("tel: " + telNumber);
    }
}
```
注册你的组件：

```java
WXSDKEngine.registerComponent("richText", RichText.class);
```

JS 调用如下：

```html
<template>
  <div>
    <richText tel="12305" style="width:200;height:100">12305</richText>
  </div>
</template>
```
#### 组件方法支持
从WeexSDK 0.9.5开始，你可以定义组件方法

- 在组件中如下声明一个组件方法

 ```java
 @JSMethod
 public void focus(){
  //method implementation
 }
 ```

- 注册组之后，你可以在weex 文件中调用

  ```html
	<template>
    <mycomponent ref='mycomponent'></mycomponent>
	</template>
	<script>
    module.exports = {
      created: function() {
        this.$refs.mycomponent.focus();
      }
    }
	</script>
	```

注:工程要添加依赖 `compile 'com.squareup.picasso:picasso:2.5.2'`

## Adapter扩展

图片下载：

需要时集成接口 IWXImgLoaderAdapter，实现 setImage 方法。

示例如下：

```java
public class ImageAdapter implements IWXImgLoaderAdapter {

  public ImageAdapter() {
  }

  @Override
  public void setImage(final String url, final ImageView view,
                       WXImageQuality quality, WXImageStrategy strategy) {

    WXSDKManager.getInstance().postOnUiThread(new Runnable() {

      @Override
      public void run() {
        if(view==null||view.getLayoutParams()==null){
          return;
        }
        if (TextUtils.isEmpty(url)) {
          view.setImageBitmap(null);
          return;
        }
        String temp = url;
        if (url.startsWith("//")) {
          temp = "http:" + url;
        }
        if (view.getLayoutParams().width <= 0 || view.getLayoutParams().height <= 0) {
          return;
        }
        Picasso.with(WXEnvironment.getApplication())
            .load(temp)
            .into(view);
      }
    },0);
  }
}
```
#### SDK混淆规则
若要在APP中使用混淆，请在相应的配置文件中添加如下规则：

```java
-keep class com.taobao.weex.WXDebugTool{*;}
-keep class com.taobao.weex.devtools.common.LogUtil{*;}
-keepclassmembers class ** {
  @com.taobao.weex.ui.component.WXComponentProp public *;
}
-keep class com.taobao.weex.bridge.**{*;}
-keep class com.taobao.weex.dom.**{*;}
-keep class com.taobao.weex.adapter.**{*;}
-keep class com.taobao.weex.common.**{*;}
-keep class * implements com.taobao.weex.IWXObject{*;}
-keep class com.taobao.weex.ui.**{*;}
-keep class com.taobao.weex.ui.component.**{*;}
-keep class com.taobao.weex.utils.**{
    public <fields>;
    public <methods>;
    }
-keep class com.taobao.weex.view.**{*;}
-keep class com.taobao.weex.module.**{*;}
-keep public class * extends com.taobao.weex.common.WXModule{*;}
-keep public class * extends com.taobao.weex.ui.component.WXComponent{*;}
-keep class * implements com.taobao.weex.ui.IExternalComponentGetter{*;}
```
