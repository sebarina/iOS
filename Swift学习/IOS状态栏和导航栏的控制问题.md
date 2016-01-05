#IOS状态栏和导航栏的控制问题

>IOS的项目多数会遇到控制状态栏和导航栏的问题，比如隐藏状态栏、控制状态栏的背景颜色、文字颜色等，导航栏也有同样需求。本文总结一下操作方法。

##1. iOS界面的组成
IOS的界面分为状态栏和导航栏，状态栏是指显示电池、时间的最顶部的一个窄条，高度为20个点；而导航栏是紧接着状态栏的44个点高度的横条，一般用于显示app标题，返回按钮等操作按钮。
在ios7之前，状态栏和导航栏是分开的，而从ios7开始状态栏和导航栏交织在一起了，状态栏变为透明，导航栏的高度变为44+20=64：
![](https://mmbiz.qlogo.cn/mmbiz/xkoH02ic8yabs1N8mKiaDdfVM9782Lh1kB6zKL3rZxoc31ib44GGjT5o9r5Wc1GfSR5Pvib2p6Qo0IkPR32jyFE8Sg/0?wx_fmt=jpeg)

##2. 状态栏控制

- 全局控制： 在info.plist文件中配置属性"controller-based status bar appearance" 为NO， 这样，在view controller中的配置都无效； 当然也可以通过代码来实现全局的配置：
<pre>//设置顶部状态栏
if (IS_IOS_7) {
    UIApplication.sharedApplication().setStatusBarStyle(UIStatusBarStyle.Default, animated: false)          
} else {
    UIApplication.sharedApplication().setStatusBarStyle(UIStatusBarStyle.LightContent, animated: false)          
}</pre>

- 局部控制：在ViewController中设置状态的样式
<pre>func preferredStatusBarStyle() -> UIStatusBarStyle {
    return UIStatusBarStyle.LightContent
}</pre>
为了确保样式设置成功，在合适的地方要调用setNeedsStatusBarAppearanceUpdate 方法（如在viewWillAppear方法中调用）

- 状态栏的隐藏展示控制：
<pre>// 实现方法： 
func prefersStatusBarHidden() -> Bool {
    return true
}
//调用方法引发更新：
setNeedsStatusBarAppearanceUpdate()</pre>

##3. 导航栏控制
### 导航栏背景颜色设置：
<pre>
/// 设置导航栏的背景颜色, Default: White color
var navBarColor : UIColor = ColorConstant.navBarColor {
    didSet {
        navigationController?.navigationBar.barTintColor = navBarColor
        navigationController?.navigationBar.backgroundColor = navBarColor
    }
}
</pre>
### 导航栏的shadow配置：
<pre>/// 设置导航栏 shadow hint是否隐藏, 默认为 false
var hideNavBarHint : Bool = false {
    didSet {
        if hideNavBarHint {
            navigationController?.navigationBar.setBackgroundImage(UIImage(), forBarMetrics: UIBarMetrics.Default)
            navigationController?.navigationBar.shadowImage = UIImage()
        } else {
            navigationController?.navigationBar.setBackgroundImage(nil, forBarMetrics: UIBarMetrics.Default)
            navigationController?.navigationBar.shadowImage = nil
        }
    }    
}
</pre>

### 导航栏控制按钮的设置：
- 自定义 UIBarButtonItem
- 设置navigationItem的leftBarButtonItem和rightBarButtonItem 
<pre>self.navigationItem.leftBarButtonItem = backItem
self.navigationItem.rightBarButtonItems = rightBarItems
</pre>

### 导航栏标题栏的设置：
- 标题为文字，文字颜色设置
<pre>
self.navigationItem.titleView = nil
self.navigationItem.title = self.title
if navigationController != nil {
    if navigationController!.navigationBar.titleTextAttributes == nil {
        navigationController!.navigationBar.titleTextAttributes = [NSForegroundColorAttributeName: navTitleColor,NSFontAttributeName:UIFont.systemFontOfSize(18)]
    } else {
        navigationController!.navigationBar.titleTextAttributes![NSForegroundColorAttributeName] = navTitleColor
    }
}
</pre>
- 自定义标题设置: 通过navigationItem的titleView来实现
<pre>
    self.navigationItem.titleView = customTitleView
</pre>

### 不同View Controller不同的配置实现：
- 为导航栏定义属性
- 在viewWillAppear方法中根据定义的属性更新导航栏的展示
