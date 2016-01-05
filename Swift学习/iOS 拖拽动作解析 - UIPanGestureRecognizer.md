# iOS 拖拽动作解析 - UIPanGestureRecognizer

UIPanGestureRecognizer是拖拽动作，拖拽动作是一个连续的动作，当手指开始移动并被确认为拖动的动作时，就开始了(UIGestureRecognizerStateBegan)； 手指按下拖动的过程中，动作出于Changed状态(UIGestureRecognizerStateChanged)； 当手机都抬起了时，动作就结束了 (UIGestureRecognizerStateEnded)， 下面是官方文档的描述：
>It begins (UIGestureRecognizerStateBegan) when the minimum number of fingers allowed (minimumNumberOfTouches) has moved enough to be considered a pan. It changes (UIGestureRecognizerStateChanged) when a finger moves while at least the minimum number of fingers are pressed down. It ends (UIGestureRecognizerStateEnded) when all fingers are lifted.

## 拖拽动作的配置
1. maximumNumberOfTouches
最多多少个手指点击，才能被认为是有效的拖拽动作
<pre>var maximumNumberOfTouches: Int </pre>

2. minimumNumberOfTouches 
最少多少个手指点击，才能被认为是有效的拖拽动作
<pre>var minimumNumberOfTouches: Int</pre>

## Tracking the Location and Velocity of the Gesture

**1.location**
translationInView: 该方法将拖拽的动作轨迹转换成在指定view中的位置点（x：横向的移动距离，y：纵向的移动距离）
官方文档说明：
> The x and y values report the total translation over time. They are not delta values from the last time that the translation was reported. Apply the translation value to the state of the view when the gesture is first recognized—***do not concatenate the value each time the handler is called.***
<pre>func translationInView(_ view: UIView?) -> CGPoint</pre>

**2.velocity**
velocityInView: 这个方法返回了拖拽动作的速度，将拖拽动作的速度转换成在制定View中的位置点（x： 横向的拖动速度， y：纵向的拖动速度），该速度是以每秒移动的像素点来描述的。
官方说明文档：
> The velocity of the pan gesture, which is expressed in points per second. The velocity is broken into horizontal and vertical components.
<pre>func velocityInView(_ view: UIView?) -> CGPoint</pre>


