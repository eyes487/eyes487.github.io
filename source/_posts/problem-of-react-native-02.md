---
title: Android隐藏软键盘
categories: 前端
tag: ['React native','Android']
date: 2020-10-10
---

公司有需求需要使用PDA扫码，采用的方式是把扫码内容输出到输入框可以拿到扫码结果，这时就需要隐藏软键盘了。我所使用的框架是`ReactNative`，`Keyboard` 组件有一个方法 `dismiss()` 可以隐藏软键盘，但是它同时会使文本框失去焦点，这样就拿不到扫码内容，所以这个方法就不行了，就需要用原生的方法来隐藏软键盘。

## 1.在 android/app/src/main/java/com/app 文件下创建文件夹 newpda

### 在此文件夹下，创建文件  `BoardModule.java`

```java
package com.newpda;
import com.facebook.react.bridge.ReactApplicationContext;
import com.facebook.react.bridge.ReactContextBaseJavaModule;
import com.facebook.react.bridge.ReactMethod;

import android.content.Context;
import android.app.Activity;
import android.view.inputmethod.InputMethodManager;
import com.facebook.infer.annotation.Assertions;
import android.view.View;

/**
 * Description: Created by eyes487 on 2020/9/2. email：659721336@qq.com
 */
public class BoardModule extends ReactContextBaseJavaModule {
    private final ReactApplicationContext reactContext;  
    public BoardModule(ReactApplicationContext reactContext) {
        super(reactContext);
        this.reactContext = reactContext;   // 获取上下文
    }

    @Override
    public String getName() {
        return "BoardModule";
    }

    /**
     *    关闭Edittext软件盘，光标依然正常显示。 
     */
    @ReactMethod
    public void hideboard() {
        Activity currentActivity = getCurrentActivity();
        InputMethodManager mInputMethodManager = (InputMethodManager)
        Assertions.assertNotNull(this.reactContext.getSystemService(Context.INPUT_METHOD_SERVICE));

        if(mInputMethodManager == null) return;
        View view = currentActivity.getCurrentFocus();

        if(view == null) view = new View(currentActivity);
        mInputMethodManager.hideSoftInputFromWindow(view.getWindowToken(), 0);
    }
}
```

### 同时创建文件 ` CustomBoardPackage.java`

```java
package com.newpda;

import com.facebook.react.ReactPackage;
import com.facebook.react.bridge.NativeModule;
import com.facebook.react.bridge.ReactApplicationContext;
import com.facebook.react.uimanager.ViewManager;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

/**
 * Description:
 * Created by eyes487 on 2020/9/2.
 * email：659721336@qq.com
 */
public class CustomBoardPackage implements ReactPackage {
    @Override
    public List<NativeModule> createNativeModules(ReactApplicationContext reactContext) {
        List<NativeModule> modules=new ArrayList<>();
        modules.add(new BoardModule(reactContext));
        return modules;
    }

    @Override
    public List<ViewManager> createViewManagers(ReactApplicationContext reactContext) {
        return Collections.emptyList();
    }
}
```

### 创建完成之后，要在 MainApplication.java 中引用

android/app/src/main/java/com/app/MainApplication.java

```java
//....
+ import com.newpda.CustomBoardPackage;

protected List<ReactPackage> getPackages() {
    return Arrays.<ReactPackage>asList(
        //...
        new CustomBoardPackage()   // 刚刚添加的方法
    );
}
```

原生的方法就配置完成了，只需要在js端引用就行了

## 2.在 React代码中引用

### 创建一个文件，引用NativeModules，就不用每次都调用了,新建一个nativeHideBoard.js
```js
import {NativeModules} from 'react-native';

module.exports = NativeModules.BoardModule;
```

### 在需要的地方引用
```js
import BoardModule from "./nativeHideBoard";

//...

//在输入框获得焦点的时候，隐藏软键盘，输入框也不会失去焦点
_inputFocus=()=>{
    BoardModule.hideboard();
}
```

-------------如果以上内容有不对的地方，还请大家指正------------