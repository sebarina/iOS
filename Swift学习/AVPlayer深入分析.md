# AVPlayer深入分析
本文主要分析一下使用AVPlayer来构建自己的视频播放页面和控制器，包括视频快进、暂停、播放、退出等功能

##AVPlayer概述
AVPlayer可被Controller或者View用来播放单个或多个视频，AVPlayer不仅支持本地视频资源的播放，还支持远程视频资源的播放。我们可以通过AVPlayer监控视频的状态信息，如视频是否准备好了播放、视频缓冲信息、视频播放进度等等。

##AVPlayer初始化
首先来看一下AVPlayer的工作流程图：
![AVPlayer](http://7xos21.com1.z0.glb.clouddn.com/AVPlayer.png)

先将在线视频链接存放在videoUrl中，然后初始化playerItem，playerItem是管理资源的对象（A player item manages the presentation state of an asset with which it is associated. A player item contains player item tracks—instances of AVPlayerItemTrack—that correspond to the tracks in the asset）。上代码：
<pre>
let playItem = AVPlayerItem(URL: contentUrl)
player = AVPlayer(playerItem: playItem)
</pre>

##UI展示
AVPlayer本身并不能显示视频，而且它也不像MPMoviePlayerController有一个view属性。如果AVPlayer要显示必须创建一个播放器层AVPlayerLayer用于展示，播放器层继承于CALayer，有了AVPlayerLayer之添加到控制器视图的layer中即可。具体实现如下代码所示：
<pre>
    let playerLayer = AVPlayerLayer(player: player)
    playerLayer.frame = CGRectMake(0,0,frame.width,frame.height)
    layer.addSublayer(playerLayer)
</pre>

##AVPlayer播放控制
###（1）视频的加载播放状态
视频的加载播放状态可以通过监控status属性来监听，在视频播放之前，播放器会尝试去加载视频资源，会将加载的状态通过status属性通知出来。
<pre>player?.addObserver(self, forKeyPath: "status", options: NSKeyValueObservingOptions.New, context: nil)</pre>
AVPlayer的status取值有：
- AVPlayerStatus.ReadyToPlay: 视频已经准备好可以播放了，此后
- AVPlayerStatus.Unknown: 未知状态，还没有开始加载视频，所以不知道能不能播放
- AVPlayerStatus.Failed: 加载失败了，不能播放视频
###（2）视频的缓冲状态
视频的缓冲状态是通过监控播放器的player item的loadedTimeRanges属性来监控的：
<pre>player?.addObserver(self, forKeyPath: "currentItem.loadedTimeRanges", options: NSKeyValueObservingOptions.New, context: nil)</pre>
这个属性值会告诉播放器视频的哪一段是缓冲好了：下面的代码会根据缓冲的大小来自动的继续播放卡住的视频。
<pre>
    if let timeRanges = change?[NSKeyValueChangeNewKey] as? [AnyObject] {
        if timeRanges.count > 0 {
            let start = CMTimeGetSeconds(timeRanges[0].CMTimeRangeValue.start)
            let end = CMTimeGetSeconds(CMTimeAdd(timeRanges[0].CMTimeRangeValue.start, timeRanges[0].CMTimeRangeValue.duration))
            if start > 0 && end > CMTimeGetSeconds(player!.currentTime()) + 10 {
                if player!.rate == 0.0 && isPlaying {
                    player!.play()                                
                }
            }
        }
    }
</pre>
###（3）视频播放暂停状态监控
视频是否正在播放还是在暂停状态可以通过监控播放器的rate属性来判断:
- rate = 0.0: 视频播放暂停
- rate > 0: 视频正在播放
<pre>player?.addObserver(self, forKeyPath: "rate", options: NSKeyValueObservingOptions.New, context: nil)</pre>
###（4）视频播放完毕通知
当视频播放完成了的时候，播放器的current item会发出通知 AVPlayerItemDidPlayToEndTimeNotification， 我们可以通过注册监听这个消息来处理当视频播放完成了后的操作。
<pre>NSNotificationCenter.defaultCenter().addObserver(self, selector: "movieFinished", name: AVPlayerItemDidPlayToEndTimeNotification, object: moviePlayerView!.player!.currentItem)</pre>

**注意：**
注册了属性值监控时，一定要配对removeObserver的操作，如在对象销毁时，调用对应的removeObserver操作：
<pre>
    deinit {
        player?.removeTimeObserver(self)
        player?.removeObserver(self, forKeyPath: "status")
        player?.removeObserver(self, forKeyPath: "rate")
    }
</pre>
##AVPlayer时间控制
###（1） 更新播放时间
播放器的addPeriodicTimeObserverForInterval:(CMTime)interval queue:(dispatch_queue_t)queue usingBlock:(void (^)(CMTime time))block;此方法就是关键，interval参数为响应的间隔时间，这里设为每秒都响应，queue是队列，传NULL代表在主线程执行。可以更新一个UI，比如进度条的当前时间等。
<pre>
    player?.addPeriodicTimeObserverForInterval(CMTimeMake(500, 1000), queue: dispatch_get_main_queue(), usingBlock: {
        (time : CMTime) in
        if self.readyToPlay {
            let endTime = CMTimeConvertScale (self.player!.currentItem?.asset.duration ?? kCMTimeZero, self.player!.currentTime().timescale, CMTimeRoundingMethod.RoundHalfAwayFromZero)
            if (CMTimeCompare(endTime, kCMTimeZero) != 0) {
                let normalizedTime = Double(self.player!.currentTime().value) / Double(endTime.value)
                self.progressBar?.value = Float(normalizedTime)
            }
            self.currentTimeLabel?.text = self.getStringFromCMTime(self.player?.currentTime() ?? kCMTimeZero)
        }
    })
</pre>

**注意：** addPeriodicTimeObserverForInterval后，在不使用Timer的时候一定要remove掉timer的observer：
<pre>player?.removeTimeObserver(self)</pre>
###（2） 拖动视频
使用播放器的seekToTime(_ time: CMTime)方法，先根据UISlider的滑动值换算出拖动到视频的哪个时间点，然后调用seekToTime方法。
<pre>
    let seekTime = CMTimeMakeWithSeconds(Double(sender.value) * Double(player!.currentItem!.asset.duration.value)/Double(player!.currentItem!.asset.duration.timescale), player!.currentTime().timescale)
    player?.seekToTime(seekTime)
    currentTimeLabel?.text = getStringFromCMTime(seekTime)
</pre>

##［参考资料］
1. [Playback](https://developer.apple.com/library/ios/documentation/AudioVideo/Conceptual/AVFoundationPG/Articles/02_Playback.html)
2. [AVPlayer Class Reference
](https://developer.apple.com/library/ios/documentation/AVFoundation/Reference/AVPlayer_Class/index.html)
