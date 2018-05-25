---
title: ExoPlayer开发指南 (译)
layout: post
date: 2018-05-24 15:38:42
comments: true
categories: 译文
tags: [Android译文,多媒体]
---
## 开发者指南

视频播放和音乐播是安卓设备上流行的用户行为 。 Android 框架提供 `MediaPlayer` 以最少的编码来解决播放媒体的问题 ， 也提供了低版本媒体 API ， 比如 `MediaCodec` ， `AudioTrack` 和 `MediaDrm` 用于自主定制媒体播放的方案 。

**ExoPlayer** 是一个构建于 Android 低版本 API 之上的开源应用项目 。 本指南描述了 ：
* 使用 **ExoPlayer** 库，并提供了具体场景的 [main demo app](https://github.com/google/ExoPlayer/tree/release-v2/demos/main) 。
* 描述 **ExoPlayer** 使用的利弊 。
* 使用 **ExoPlayer** 播放 `DASH` ， `SmoothStreaming` 和 `HLS` 自适应流 ， 以及更多的格式视频 ， 如 `MP4` ， `M4A` ， `FMP4` ， `WebM` ， `MKV` ， `MP3` ， `WAV` ， `Ogg` ，`MPEG-TS` ， `MPEG-PS` ， `FLV ` 和 `ADTS (ACC)` 等等 。
* 描述 **ExoPlayer** 包含的事件、消息、定制及 DRM 的支持 。

### 优缺点
***

相对于安卓内置的 **MediaPlayer** , **ExoPlayer** 有以下优点 ：
* 支持 `DASH` (动态自适应流媒体) 和 `SmoothStreaming` 格式 ， 具体支持的格式可查看 [Supported formats](https://github.com/google/ExoPlayer/tree/release-v2/demos/mainhttps://google.github.io/ExoPlayer%20/supported-formats.html) 。
* 支持 `HLS` 高级特性 ，比如处理 `#EXT-X-DISCONTINUIT` 标签 。
* 无缝地合并、连接和循环媒体 。
* 更新播放器版本 ，**ExoPlayer** 作为依赖库用于构建应用，你可以轻易地升级版本 。
* 良好的设备兼容，在不同的版本设备上很少产生问题 。
* 支持 4.4 及以上使用常见加密方式 。
* 支持开发者自定义扩展播放器来适应自己的项目 ，为此 **ExoPlayer** 预留了很多可替代的组件 。
* 支持使用官方扩展组件来整合其他开源库 ， 比如 [IMA extension](https://github.com/google/ExoPlayer/tree/release-v2/extensions/ima) 使得开发者更简便地接入 [Interactive Media Ads SDK](https://developers.google.com/interactive-media-ads) 来实现盈利 。

值得留意的是 ，**ExoPlayer** 也存在部分缺点：
* 对于仅实现音频播放的设备 ， `ExoPlayer` 比其他媒体播放器可能会消耗更多的电量 。 详情可参考 [Battery consumption page](https://google.github.io/ExoPlayer/battery-consumption.html)

### 库概述
***

库的核心是 *ExoPlayer* 接口 。 *ExoPlayer* 暴露了传统高级媒体播放器的功能 ， 如缓冲、播放、暂停及进度控制 。 预先设想实现了如何播放、如何存储及在哪存储 ，如何渲染一些媒体等 ， 而不是实现直接实现加载和渲染 media 。这些都是 **ExoPlayer** 在播放器被创建或准备回播时 ，注入组件委托者来实现 。 这些组件分别是：
* **MediaSource** 定义了如何播放、加载 media ， 以及在哪里加载可读的 media 。 *MediaSource* 对象会在回播开始时调用 `ExoPlaer.prepare` 后被注入 。
* **Render** 能渲染 MediaSource 中各个独立组件的渲染器 。 *Render* 对象会在播放器创建时注入 。
* **TrackSelector** 从 MediaSource 中选择轨道以供 *Render* 渲染器消费 。 *TrackSelector* 对象会在播放器创建时注入 。
* **LoadControl** 控制 MediaSource 缓冲更多的 media 及 控制缓冲量 。 *LoadControl* 对象会在播放器创建时注入 。

库提供一些默认实现组件以供常见用例使用 ，后续将有更详细的描述 。 一个 *ExoPlayer* 对象能使用多个默认组件 ，但也可以使用一些非标准行为的自定义组件 。 比如通过注入自定义 *LoadControl* 对象来改变播放器的缓冲策略 ， 或者注入自定义 *Render* 对象来使用 Android 不支持的视频编解码器。

注入实现播放器功能组件的理念贯穿整个库 。 上述默认实现的组件都是委托工作到进一步注入的组件中 ，这使得许多子组件可以通过自定义实现来替换其功能 。例如 ， 默认的 `MediaSource` 实现中，其构造器需要注入一个或多个 `DataSource` 工厂对象。 通过提供自定义工厂能从非标注输入源中或不同的网络堆栈中加载数据 。

### 开始
***

对于简单用例，开发者可通过以下步骤使用 **ExoPlayer** ：
* 添加 ExoPlayer 依赖到项目 。
* 创建一个 *SimpleExoPlayer* 对象 。
* 绑定播放器到界面 (用于视频输入及用户输入) 。
* 把 *MediaSource* 给播放器加载准备播放 。
* 播放工作完成后释放播放器 。

下面将详细描述这些步骤 。 对于完成实例 ， 请参考[main demo app](https://github.com/google/ExoPlayer/tree/release-v2/demos/main) 中的 `PlayerActivity` 。

#### 添加 ExoPlayer 作为项目依赖

确保项目文件 `build.gradle` 根节点中包含 `JCenter` 和 `Google` 仓库

```
repositories {
    jcenter()
    google()
}
```

接着在 app 模块中的 `build.gredle` 添加 ExoPlayer 库依赖 ：

```
implementation 'com.google.android.exoplayer:exoplayer:2.X.X'
```

`2.X.X` 是你所需要的库版本 。 另外 ， 你也可以只依赖项目所需要的库 。 比如 ， 如果项目只需要播放 `DASH` 格式的内容 ，将会添加 `Code` , `DASH` 和 `UI` 库 ：

```
implementation 'com.google.android.exoplayer:exoplayer-core:2.X.X'
implementation 'com.google.android.exoplayer:exoplayer-dash:2.X.X'
implementation 'com.google.android.exoplayer:exoplayer-ui:2.X.X'
```

下面列出的是可用的模块 。 添加一个完整的 ExoPlayer 库相当于添加下面所有的独立库 :
* **exoplayer-core** : 核心功能 (必须)
* **exoplayer-dash** : 用于支持 DASH 格式的内容
* **exoplayer-hls** : 用于支持 HLS 格式的内容
* **exoplayer-smoothstreaming** : 用于支持 SmoothStreaming 格式的内容
* **exoplayer-ui** : ExoPlayer 能使用的 UI 组件和资源

除了库模块外 ， ExoPlayer 有多个依赖外部库实现的扩展模块用于提供额外的功能 。 这些内容不再本指南的范围内，具体细节可以查看 [extensions directory](https://github.com/google/ExoPlayer/tree/release-v2/extensions/) 及 README 文件 。

#### 创建播放器

你可以使用 `ExoPlayerFactory` 来创建一个 *ExoPlayer* 对象 。 工厂类提供了一系列方法来创建可定制化的 *ExoPlayer* 对象 。 对于绝大部分用例 , `ExoPlayerFactory.newSimpleInstance` 会被调用 。 这些方法会返回 `SimpleExoPlayer` 对象 ， 该继承于 `ExoPlayer` 并添加了额外的高级播放器高级功能。 下面的代码是展示如何创建一个 `SimpleExoPlayer` 对象 。

```
// 1. Create a default TrackSelector
Handler mainHandler = new Handler();
BandwidthMeter bandwidthMeter = new DefaultBandwidthMeter();
TrackSelection.Factory videoTrackSelectionFactory =
    new AdaptiveTrackSelection.Factory(bandwidthMeter);
TrackSelector trackSelector =
    new DefaultTrackSelector(videoTrackSelectionFactory);

// 2. Create the player
SimpleExoPlayer player =
    ExoPlayerFactory.newSimpleInstance(context, trackSelector);
```

#### 绑定一个播放器到视图
ExoPlayer 库提供了一个 `PlayerView` 类 ， 封装了 `PlayerControlView` 控制层视图和 `Surface` 视频呈现层 。 `PlayerView` 能包含写入程序 layout 布局中 。 下面代码是展示绑定一个播放器到视图上 ：

```
// Bind the player to the view.
playerView.setPlayer(player);
```

如果你需要进一步控制播放器的控制层及视频呈现层 ， 可以直接调用 `SimpleExoPlayer` 的 `setVideoSurfaceView` ， `setVideoTextureView` ， `setVideoSurfaceHolder ` 或 `setVideoSurface ` 来分别设置 `SurfaceView` ， `TextureView` ， `SurfaceHolder` 或者 `Surface` 。 可以把 `PlayerControlView` 当做一个独立的组件使用 ， 或者实现自己的播放控制来直接与用户进行互动 。 在播放的时候， `setTextOutput` 和 `setId3OutPut` 能够被用于接收标题和 ID3 元数据 。

#### 准备播放器

在 ExoPlayer 中每一块媒体都由 `MediaSource` 提供。 为了播放每一块媒体 ， 你必须先创建一个对应的 `MediaSource` 对象并传递给 `ExoPlayer.prepare` 。 ExoPlayer 库提供了一系列 `MediaSource` 实现类 ， 如 `DASH (DashMediaSource)` ， `SmoothStreaming (SsMediaSource)` ， `HLS (HlsMediaSource)` 和 常见的媒体文件 (ExtractorMediaSource) 。 这些实现的细节会在后续的指南中描述 。 下面的代码展示了播放器利用 `MediaSource` 准备播放一个 MP4 文件 ：

```
// Measures bandwidth during playback. Can be null if not required.
DefaultBandwidthMeter bandwidthMeter = new DefaultBandwidthMeter();

// Produces DataSource instances through which media data is loaded.
DataSource.Factory dataSourceFactory = new DefaultDataSourceFactory(context,
    Util.getUserAgent(context, "yourApplicationName"), bandwidthMeter);

// This is the MediaSource representing the media to be played.
MediaSource videoSource = new ExtractorMediaSource.Factory(dataSourceFactory)
    .createMediaSource(mp4VideoUri);

// Prepare the player with the source.
player.prepare(videoSource);
```

#### 控制播放器

一旦播放器准备就绪 ， 就可以通过调用方法来控制播放行为 。 比如 `setPlayWhenReady` 可控制是否暂停/播放 ； `seekTo` 可在媒体进度中寻求播放点 ； `setRepeatMode` 可可控制媒体是否循环和如何循环 ； `setPlaybackParameters` 可调整播放时的速度和音调 。

如果播放器绑定了 `PlayerView` 或者 `PlayerControlView` ，则用户与视图的互动会调用播放器的对应方法。

#### 监听播放器事件

状态的改变或者播放异常这些事件会被上报到已注册的 `Player.EventListener` 对象 。注册一个监听对象用于接收事件是比较简单的 ：

```
// Add a listener to receive events from the player.
player.addListener(eventListener);
```

如果你只对其中部分事件感兴趣 ， 可继承 `Player.DefaultEventListener` 覆盖部分方法而不是实现 `Player.EventListener` 。

当你使用 `SimpleExoPlayer` 时， 还可以添加额外的监听 。 `addVideoListener` 方法允许你接收视频渲染相关的事件 , 这可能有助于你调整视图 。 `addAnalyticsListener` 方法允许你接收更细节的事件 ， 能有效帮助你分析播放行为 。

#### 释放播放器

当不在使用播放器时 ， 释放播放器是非常重要的 。 释放播放器能释放有限的资源 ， 如可供其他应用使用的视频解码器 ， 能通过调用 `ExoPlayer.release` 实现。

### 媒体资源 (MediaSource)
***

在 ExoPlayer 中，每一块媒体都由 `MediaSource` 提供 。 ExoPlayer 库提供了一系列 `MediaSource` 实现类 ， 如 `DASH (DashMediaSource)` , `SmoothStreaming (SsMediaSource)` ， `HLS (HlsMediaSource)` 和 常见的媒体文件 (ExtractorMediaSource) 。 可在 [main demo app](https://github.com/google/ExoPlayer/tree/release-v2/demos/main) 的 `PlayerActivity` 中查看如何实例化这四种对象 。

除了上述四种 `MediaSource` 的实现外 ， ExoPlayer 库也提供了 `MergingMediaSource` ， `LoopingMediaSource` 和 `ConcatenatingMediaSource` 。 这些 `MediaSource` 的实现能处理复杂的组合播放 。 下面将会描述一些常见的例子 。 注意这些例子虽然使用了相同类型的媒体资源 ， 但实际上是支持多种媒体资源组合播放的 。

#### 加载字幕

`MergingMediaSource` 能够将给定的视频文件和独立的字幕文件合并成单一的播放资源 。

```
// Build the video MediaSource.
MediaSource videoSource =
    new ExtractorMediaSource.Factory(...).createMediaSource(videoUri);

// Build the subtitle MediaSource.
Format subtitleFormat = Format.createTextSampleFormat(
    id, // An identifier for the track. May be null.
    MimeTypes.APPLICATION_SUBRIP, // The mime type. Must be set correctly.
    selectionFlags, // Selection flags for the track.
    language); // The subtitle language. May be null.
MediaSource subtitleSource =
    new SingleSampleMediaSource.Factory(...)
        .createMediaSource(subtitleUri, subtitleFormat, C.TIME_UNSET);

// Plays the video with the sideloaded subtitle.
MergingMediaSource mergedSource =
    new MergingMediaSource(videoSource, subtitleSource);
```

#### 循环一个视频

> 如果不确定循环视频循环周期 ， 使用 `ExoPlayer.setRepeatMode` 替代 `LoopingMediaSource` 会更好 。

一个视频能够被无缝循环播放特定次数 。 下面展示如何播放视频两次 ：

```
MediaSource source =
    new ExtractorMediaSource.Factory(...).createMediaSource(videoUri);
// Plays the video twice.
LoopingMediaSource loopingSource = new LoopingMediaSource(source, 2);
```

#### 按照顺序播放视频

`ConcatenatingMediaSource` 能将多个独立的 `MediaSource` 按照一定的顺序进行播放 。 下面用例展示顺序播放两个视频 。 视频间的切换时无缝衔接的 。 顺序播放的视频不要求是相同格式的 (比如一个视频可以是 480p H264 ， 另一个可以是 720p VP9) 。 播放源甚至可以是不同类型的 (比如可以一个视频衔接一个音频) 。 同时 ，也允许在单一连接中多次使用独立的 `MediaSource` ， `MediaSource` 也可以在播放前或播放时动态添加 ， 删除 和移动 。

```
MediaSource firstSource =
    new ExtractorMediaSource.Factory(...).createMediaSource(firstVideoUri);
MediaSource secondSource =
    new ExtractorMediaSource.Factory(...).createMediaSource(secondVideoUri);
// Plays the first video, then the second video.
ConcatenatingMediaSource concatenatedSource =
    new ConcatenatingMediaSource(firstSource, secondSource);
```

#### 高级用法

开发者可以根据特定的用例自行组合 `MediaSource` 。 给定视频 A 和 视频 B ， 下面用例展示 `LoopingMediaSource`  和 `ConcatenatingMediaSource` 如何按照 (A , A , B) 的顺序进行播放 ：

```
MediaSource firstSource =
    new ExtractorMediaSource.Factory(...).createMediaSource(firstVideoUri);
MediaSource secondSource =
    new ExtractorMediaSource.Factory(...).createMediaSource(secondVideoUri);
// Plays the first video twice.
LoopingMediaSource firstSourceTwice = new LoopingMediaSource(firstSource, 2);
// Plays the first video twice, then the second video.
ConcatenatingMediaSource concatenatedSource =
    new ConcatenatingMediaSource(firstSourceTwice, secondSource);
```

接下来的用例实现的功能和上述例子一致 ， 实际上有多种实现的方式来达到一样的效果 。

```
MediaSource firstSource =
    new ExtractorMediaSource.Builder(firstVideoUri, ...).build();
MediaSource secondSource =
    new ExtractorMediaSource.Builder(secondVideoUri, ...).build();
// Plays the first video twice, then the second video.
ConcatenatingMediaSource concatenatedSource =
    new ConcatenatingMediaSource(firstSource, firstSource, secondSource);
```

### 发送消息给组件
***

开发者发送消息给 ExoPlayer 组件 ， 通过使用 `createMessage` 创建消息并调用 `PlayerMessage.send` 发送 。 默认情况下 ， 消息应该尽可能在播放线程中传递 ， 但也可对消息进行定制 ， 如设置回调线程 (使用 PlayerMessage.setHandler) 或指定传递播放位置 (使用 PlayerMessage.setPosition) 。 通过 ExoPlayer 发送消息能确保播放器当有其他操作被执行时 ， 该操作能被执行 。

ExoPlayer 绝大部分渲染器都允许在播放时接收消息更改配置信息 。 举个例子 ， 音频渲染器接收消息设置音量和视频渲染器接收消息设置 surface 。 这些消息都应该在播放线程中传递已确保线程安全 。

### 定制
***

ExoPlayer 相对于安卓内置的 `MediaPlayer` 的一个明显优势就是支持开发者定制扩展播放器以更好的满足用户需求 。 ExoPlayer 库的设计都贯穿这一理念 ， 为了让开发者能方便地替代默认实现， 库设计了一系列接口和基础抽象类 。 下面有一些用例可供自定义构建组件 ：
* **Renderer** ， 实现自定义 `Render` 渲染器来处理库不支持媒体类型 。
* **TrackSelector** ， 实现自定义 `TrackSelector` 轨道选择器允许开发者改变 `MediaSource` 暴露轨道的方式 ， 这些轨道会被每一个可用的渲染器所渲染 。
* **LoadControl** , 实现自定义 `LoadControl` 允许开发者改变播放器缓冲策略 。
* **Extractor** , 如果你如果支持一种当前库不支持的容器格式 ， 则可自定义 `Extractor` 和 `ExtractorMediaSource` 组合播放该类型的媒体 。
* **MediaSource** ， 如果你想获取媒体样品进行自定义渲染或更改 `MediaSrouce` 的合成行为，可实现自定义 `MediaSource`
* **DataSource** , ExoPlayer 上游包中已包含多个 `DataSource` 的实现 。 你可以实现自己的 `DataSource` 从其他途径加载数据 ， 比如对于自定义协议 ， 使用自定义 HTTP 栈或自定义缓存来加载 。

当构建自定义组件式时 ， 推荐采纳以下建议 ：
* 自定义的组件需要上报事件给应用 ， 建议你使用相同的模型 (构造时需提供了事件监听及 `Handler`) 作为 ExoPlayer 组件模板 。
* 我们建议使用相同的模型 (允许在播放时更改配置 ， 如 [Sending messages to components](https://google.github.io/ExoPlayer/guide.html#sending-messages-to-components) 中描述) 作为 ExoPlayer 组件模板 。 为了实现这一行为 ， 你应该实现 `ExoPlayerComponent` 并在其 `handleMessage` 方法中接收配置变化 。 你的应用应该通过 ExoPlayer's `sendMessages` 和 `blockingSendMessages` 方法来传递配置更改的信息 。

### 高级主题
***

#### 数字版权管理
ExoPlayer 从 Android 4.4 (API级别19) 支持数字版权管理 (DRM) 媒体播放。 更多细节请查看 [DRM page for more details](https://google.github.io/ExoPlayer/drm.html) 。

#### 电量消耗
关于使用 ExoPlayer 的电量消耗信息可在 [Battery consumption page](https://google.github.io/ExoPlayer/battery-consumption.html) 中查阅 。

#### 最小化 使用ExoPlayer库
最小化使用 ExoPlayer 库可在 [Shrinking ExoPlayer page](https://google.github.io/ExoPlayer/shrinking.html) 中查阅 。