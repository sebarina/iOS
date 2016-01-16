# iOS - UICollectionView学习

在一些购物类的APP中，我们经常会看到产品的瀑布流页面，这种瀑布流的展示可以通过UICollectionView来实现。本文就针对UICollectionView的用法做一个详细的介绍

## 简介
首先我们来看看官网上是怎么介绍UICollectionView的：

> The UICollectionView class manages an ordered collection of data items and presents them using customizable layouts. 

UICollectionView可以用来管理一批有序的数据，并且可以使用自定义的layout来展示它们。
Collection view有着tableView类似的功能，将数据通过列表的形式展示出来，但是collection view更强大的功能在于它支持custom的layout，如多列、平铺、圆形布局等等，我们甚至可以动态的改变collection view的布局。如，我们可以任意的制定一个数据项在UI上的长度和宽度，也可以指定两个数据项之间的间隔大小。

Collection view是以Cell为单位来展示数据项的，然后cell也可以归类为section。例如Photo App， 数据项是一个image，collection view通过一个cell来展示这个image

## Collection Views and Layout Objects
UICollectionView在初始化的时候需要指定一个layout，这个layout（UICollectionViewLayout类型的对象）用于布局所有的cell和 supplementary view. 这个layout只负责这些视图的显示位置和大小，并不管这些视图的创建。虽然我们初始化Collection view的时候指定了一个layout，但是这个layout是可以动态改变的，通过修改UICollectionView的collectionViewLayout属性来更新collection view的layout布局，一旦设置了这个值，collection view会马上更新视图展示。当然这个更新的过程是没有动画效果的，如果你想添加动画效果，可以调用collection view的方法： 
<pre>setCollectionViewLayout(_ layout: UICollectionViewLayout,
                        animated animated: Bool,
                        completion completion: ((Bool) -> Void)?)</pre>

collection view中的layout尺寸是通过 UICollectionViewDelegateFlowLayout来实现的：
### 1. cell尺寸的设置

<pre>func collectionView(_ collectionView: UICollectionView,
                      layout collectionViewLayout: UICollectionViewLayout,
      sizeForItemAtIndexPath indexPath: NSIndexPath) -> CGSize
</pre>

如果没有实现这个方法，则会使用flow layout的itemSize属性值作为cell的大小值。
另外，flow layout不会为了适应网格而裁剪cell的大小，如果一行展示不下，则会拉大cell之间的间隙，选择新的一行展示。

### 2. cell之间的间隔设置
这个间隔有2种：（**下面以纵向的layout为例来说明**）
- 横向cell之间的距离：
<pre>func collectionView(_ collectionView: UICollectionView,
                                  layout collectionViewLayout: UICollectionViewLayout,
minimumInteritemSpacingForSectionAtIndex section: Int) -> CGFloat
</pre>
- 纵向row之间的距离
<pre>func collectionView(_ collectionView: UICollectionView,
                             layout collectionViewLayout: UICollectionViewLayout,
minimumLineSpacingForSectionAtIndex section: Int) -> CGFloat
</pre>

注意： 这是minimum值的设置，在保证最小间隔的情况下去布局cell，如果一行排列不下，则换一行展示，从而上一行cell的间距也会被拉大。所以在程序中要固定这个间距就要动态的去计算cell的宽度（根据当前的collection view的宽度去计算）

### 3. supplementary view尺寸的设置
(1) header view
<pre>func collectionView(_ collectionView: UICollectionView,
                         layout collectionViewLayout: UICollectionViewLayout,
referenceSizeForHeaderInSection section: Int) -> CGSize</pre>

(2) footer view
<pre>func collectionView(_ collectionView: UICollectionView,
                         layout collectionViewLayout: UICollectionViewLayout,
referenceSizeForFooterInSection section: Int) -> CGSize</pre>

如果没有实现该方法，则会使用layout的 footerReferenceSize属性值和headerReferenceSize属性值来指定footer和header的尺寸大小，这两个属性的默认值为(0,0)，如果返回值为(0,0)则没有footer或者header

> **注意：** 在布局中，只有滚动方向的那个尺寸值才会被使用来决定view的尺寸。例如，对于纵向滚动的layout，layout只会使用返回size中的height值（view的宽度会使用collection view的宽度值），如果返回(100,0)，则认为没有footer活着header

### 4. section的margins设置
<pre>func collectionView(collectionView: UICollectionView, layout collectionViewLayout: UICollectionViewLayout, insetForSectionAtIndex section: Int) -> UIEdgeInsets {
    return UIEdgeInsets(top: 20, left: leftMargin, bottom: 30, right: leftMargin)
}
</pre>

如果没有实现这个方法，layout会使用它的sectionInset属性值来设置section的边距（这个属性值默认都是0）。

这个UIEdgeInsets值只对当前section的cell用，返回的UIEdgeInsets的4个值分别对应于：第一行cell和header之间的间距；左侧元素和collection view的左边框间的距离；最后一行元素和footer间的距离；右侧元素和collection view的右边框间的距离。

## Collection View Datasource and Delegate
UICollectionView和UITableView类似，由UICollectionViewDataSource协议来提供数据源，复杂cells和supplementary view的创建，由UICollectionViewDelegate（UICollectionViewDelegateFlowLayout）负责Cell操作事件等的回调。

### 1. cell的创建
调用<strong>dequeueReusableCellWithReuseIdentifier:forIndexPath: </strong>返回cell，然后对cell设置数据，最后返回给collection view；
注意： 在调用该方法前，需要注册这个cell的类： <mark>registerClass:forCellWithReuseIdentifier:</mark> 或者 <mark>registerNib:forCellWithReuseIdentifier: </mark>

### 2. supplementary view的创建
调用<strong>dequeueReusableSupplementaryViewOfKind:withReuseIdentifier:forIndexPath:  </strong>， 返回辅助的view对象，配置它的内容，返回给collection view.
同样的，在调用该方法之前，需要注册这个view的类： <mark>registerClass(_:forSupplementaryViewOfKind:withReuseIdentifier:)</mark> 或者 <mark>registerNib(_:forSupplementaryViewOfKind:withReuseIdentifier:)</mark>

> **注意：**针对不同类别的supplementary view都需要注册一次，

<strong>supplemetary view的类别有：</strong>

- UICollectionElementKindSectionHeader
- UICollectionElementKindSectionFooter

如果不需要返回这种类型的supplementary view，则不需要为这种类型dequeue reusable view， 如：
<pre>if kind == UICollectionElementKindSectionHeader {
        let headerView = collectionView.dequeueReusableSupplementaryViewOfKind(kind, withReuseIdentifier: headerIdentifier, forIndexPath: indexPath)
        ...
        return headerView
    } else {
        return UICollectionReusableView(frame: CGRectZero)
    }</pre>

## 参考资料
[UICollectionView苹果官方文档](https://developer.apple.com/library/ios/documentation/UIKit/Reference/UICollectionView_class/index.html#//apple_ref/doc/uid/TP40012177)


