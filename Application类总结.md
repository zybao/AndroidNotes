# 类继承关系：
```java
public class Application extends ContextWrapper implements ComponentCallbacks2 
```

# 构造方法
Application()

# 重要共有方法

## void onConfigurationChanged (Configuration newConfig)

Called by the system when the device configuration changes while your component is running. Note that, unlike activities, other components are never restarted when a configuration changes: they must always deal with the results of the change, such as by re-retrieving resources.

At the time that this function has been called, your Resources object will have been updated to return resource values matching the new configuration. 