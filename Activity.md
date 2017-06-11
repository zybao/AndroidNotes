# 创建Activity

创建Activity的子类(或使用其现有子类)。必须实现onCreate方法，系统将会在创建Activity时调用该方法，因此我们会在该方法内调用setContentView()以定义用户界面布局。

接着我们需要在清单文件中声明Activity使系统可以访问它。如下:
```xml
<manifest ... >
  <application ... >
      <activity android:name=".ExampleActivity" />
      ...
  </application ... >
  ...
</manifest >
```


# 生命周期