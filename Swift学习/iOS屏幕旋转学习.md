# iOS屏幕旋转学习

标签（空格分隔）： iOS Swift Orientation


----------
一般的应用，只会支持竖屏正方向一个方向，支持多个屏幕方向的应用还是比较少的。 不过，有些应用可能需求某一个页面是可以切换方向的，甚至是需要强制旋转到另一个方向（如视频播放页面）。下面是跟屏幕方向的一些总结。

> ## 怎么控制屏幕的方向？

在 iOS 的应用中，有多种方式可以控制界面的屏幕方向，有全局的，有针对 UIWindow 中界面的控制，也有针对单个界面。
## 1. 全局方向控制
在应用的 Info.plist 文件中，有一个 Supported interface orientations 的配置，可以配置整个应用的屏幕方向，如下图：

![][1]


  [1]: http://7xos21.com1.z0.glb.clouddn.com/wenzhang1.png
  
在应用启动时，会使用 Info.plist 中的 Supported interface orientations 项中的第一个值作为启动动画的屏幕方向。按照此处截图的取值，第一个取值为 Portrait，即竖屏方向，所以此应用在启动时，会使用竖屏反方向显示启动动画。

## 2. 单个页面的方向控制
单个界面的屏幕方向控制，要使用 UIViewController的2个方法：
<pre>
    override func supportedInterfaceOrientations() -> UIInterfaceOrientationMask {
        return UIInterfaceOrientationMask.Portrait
    }
    
    override func shouldAutorotate() -> Bool {
        return false
    }
</pre>

只有shouldAutorotate 返回 true 的时候，supportedInterfaceOrientations 方法才会被调用，以确定是否需要旋转界面。

**注意：** 上述2个方法只有当设备改变方向的时候才会被调用

> ### 那么问题来了，在app运行的过程中，怎么样去触发设备方向的改变呢？

**（1） 设备旋转带来的屏幕自适应转动**

    当手机的重力感应打开的时候，如果用户旋转手机，系统会抛发UIDeviceOrientationDidChangeNotification事件。您可以分别设置Application和UIViewcontroller支持的旋转方向。Application的设置会影响整个App,UIViewcontroller的设置仅仅会影响一个viewController。当UIKit收到UIDeviceOrientationDidChangeNotification事件的时候，会根据Application和UIViewcontroller的设置,如果双方都支持此方向,则会自动屏幕旋转到这个方向。更code的表达就是,会对两个设置求与,得到可以支持的方向。如果求与之后，没有任何可支持的方向，则会抛发UIApplicationInvalidInterfaceOrientationException异常。

 **2. 程序强制转向**

程序代码实现，强制要求某个页面转到指定的方向，如下述代码所示：

    let value = UIInterfaceOrientation.Portrait.rawValue
    UIDevice.currentDevice().setValue(value, forKey: "orientation")
    UIViewController.attemptRotationToDeviceOrientation()

上述代码会触发屏幕转向的通知，然后app会根据shouldAutorotate和supportedInterfaceOrientations方法来确定屏幕的方向。