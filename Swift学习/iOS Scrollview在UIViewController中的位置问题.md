# iOS Scrollview在UIViewController中的位置问题

iOS app的开发过程中，经常会使用到scrollview衍生出去的控件，如 UITableView， UICollectionView等等，然后Navigation controller又是贯穿于整个app中的，因此在很多controller中，是会展示Navigation Bar的，此时，我们的view应该是要从Navigation bar之下开始展示的，那view的起始点是0呢还是navigation bar的frame的maxY值呢？
本文就这个问题，做了一些总结：
## 1. automaticallyAdjustsScrollViewInsets属性
在iOS中，可以通过配置属性： automaticallyAdjustsScrollViewInsets值来决定是否根据Navigation bar或者 Tab bar来自动调整scroll view的content inset值。在iOS 7.x 及以下系统中，该值默认为false； 在8.x及以上系统中，该值为true；所以我们经常会遇见在8.x系统中显示正常在7.x系统中显示不对的问题。

> 解决方案： 在Controller中统一设置automaticallyAdjustsScrollViewInsets属性为false,程序中自己去计算位置。

