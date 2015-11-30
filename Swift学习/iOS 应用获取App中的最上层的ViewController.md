# iOS 应用获取App中的最上层的ViewController

标签（空格分隔）： iOS Swift ViewController

---
直接上代码了：

<pre>
- (UIViewController*)topViewController {
  return [self topViewControllerWithRootViewController:[UIApplication sharedApplication].keyWindow.rootViewController];
}
 
- (UIViewController*)topViewControllerWithRootViewController:(UIViewController*)rootViewController {
  if ([rootViewController isKindOfClass:[UITabBarController class]]) {
    UITabBarController* tabBarController = (UITabBarController*)rootViewController;
    return [self topViewControllerWithRootViewController:tabBarController.selectedViewController];
  } else if ([rootViewController isKindOfClass:[UINavigationController class]]) {
    UINavigationController* navigationController = (UINavigationController*)rootViewController;
    return [self topViewControllerWithRootViewController:navigationController.visibleViewController];
  } else if (rootViewController.presentedViewController) {
    UIViewController* presentedViewController = rootViewController.presentedViewController;
    return [self topViewControllerWithRootViewController:presentedViewController];
  } else {
    return rootViewController;
  }
}
</pre>

示例：
