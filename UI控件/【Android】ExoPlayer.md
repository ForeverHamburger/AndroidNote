ExoPlayer是运行在 YouTube App Android 版本上的视频播放器。不仅功能强大，而且使用简单，可定制性强。ExoPlayer也是Google官方推荐的Android媒体播放器，可以在Android官方文档的音频和视频目录中找到。

#### 一，优点和缺点

**优点：**

支持DASH和SmoothStreaming这两种数据格式的资源，而MediaPlayer对这两种数据格式都不支持。它还支持其它格式的数据资源，比如MP4, M4A, FMP4, WebM, MKV, MP3, Ogg, WAV, MPEG-TS, MPEG-PS, FLV and ADTS (AAC)等

支持高级的HLS特性，比如能正确的处理#EXT-X-DISCONTINUITY标签

无缝连接，合并和循环播放多媒体的能力

和应用一起更新播放器（ExoPlayer）,因为ExoPlayer是一个集成到应用APK里面的库，你可以决定你所想使用的ExoPlayer版本，并且可以随着应用的更新把ExoPlayer更新到一个最新的版本。

较少的关于设备的特殊问题，并且在不同的[Android](https://so.csdn.net/so/search?q=Android&spm=1001.2101.3001.7020)版本和设备上很少会有不同的表现。

在Android4.4(API level 19)以及更高的版本上支持Widevine通用加密

为了符合你的开发需求，播放器支持自定义和扩展。其实ExoPlayer为此专门做了设计，并且允许很多组件可以被自定义的实现类替换。

使用官方的扩展功能可以很快的集成一些第三方的库，比如IMA扩展功能通过使用互动媒体[广告](https://edu.csdn.net/cloud/pm_summit?utm_source=blogglc)SDK可以很容易地将视频内容货币化（变现）

**缺点：**

在某些设备上播放音频，ExoPlayer可能会比MediaPlayer消耗更多的电量。

#### 二，使用

添加依赖

```delphi
implementation 'com.google.android.exoplayer:exoplayer-core:2.15.1'  
implementation 'com.google.android.exoplayer:exoplayer-ui:2.15.1' 
```

创建播放器、绑定播放器容器、设置数据源、prepare

```kotlin
        mSimpleExoPlayer = new ExoPlayer.Builder(mContext).build();
        // 准备要播放的媒体资源
        MediaItem mediaItem = MediaItem.fromUri(mCurVideoPlayTask.getVideoUrl());
        mSimpleExoPlayer.setMediaItem(mediaItem);

        // 隐藏播放工具
        mCurVideoPlayTask.getSimpleExoPlayerView().setUseController(true);

        // 设置播放视频的宽高为Fit模式
        mCurVideoPlayTask.getSimpleExoPlayerView().setResizeMode(AspectRatioFrameLayout.RESIZE_MODE_FIT);
        // 绑定player和playerView
        mCurVideoPlayTask.getSimpleExoPlayerView().setPlayer(mSimpleExoPlayer);
        mSimpleExoPlayer.setPlayWhenReady(true);

        // 准备播放器
        mSimpleExoPlayer.prepare();
        mSimpleExoPlayer.play(); //1. 创建播放器
        player = SimpleExoPlayer.Builder(this).build()
        printCurPlaybackState("init")  //  此时处于STATE_IDLE = 1;
 
        //2. 播放器和播放器容器绑定
        playerView.player = player
 
        //3. 设置数据源
        //音频
        val mediaItem = MediaItem.fromUri(" https://storage.googleapis.com/exoplayer-test-media-0/play.mp3")
        player.setMediaItem(mediaItem)
    
    //4.当Player处于STATE_READY状态时，进行播放
        player.playWhenReady = true
 
    //5. 调用prepare开始加载准备数据，该方法时异步方法，不会阻塞ui线程
        player.prepare()
        printCurPlaybackState("prepare") //  此时处于 STATE_BUFFERING = 2;
```

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">
 
    <com.google.android.exoplayer2.ui.PlayerView
        android:id="@+id/player_view"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"/>
 
 
    <com.google.android.exoplayer2.ui.PlayerView
        android:id="@+id/player_view2"
        android:layout_width="wrap_content"
        android:layout_height="0dp"
        android:layout_weight="1"/>
 
</LinearLayout>
```



```java
public class MainActivity extends AppCompatActivity {
    private SimpleExoPlayer mPlayer;
    private SimpleExoPlayer mPlayer2;
    private PlayerView mPlayerView;
    private PlayerView mPlayerView2;
 
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
 
        mPlayerView = findViewById(R.id.player_view);
        mPlayerView2 = findViewById(R.id.player_view2);
 
        // 创建媒体播放器
        mPlayer = new SimpleExoPlayer.Builder(this).build();
        mPlayer2 = new SimpleExoPlayer.Builder(this).build();
        mPlayerView.setPlayer(mPlayer);
        mPlayerView2.setPlayer(mPlayer2);
 
 
        // 设置数据源
        String videoUrl = "https://v2.kwaicdn.com/upic/2023/04/14/13/BMjAyMzA0MTQxMzA3MDhfNDgwOTQ5MTg2XzEwMDU1NzkwNjgxMV8yXzM=_b_B4856a8b07ea4dbc4ad503f04681b5ec8.mp4?pkey=AAW-JkY6q-BoTEXl2FArO1ReMGt965IaFcPl0NGX7cYTVhLtgZVdld15RtYUAWwkkEmBjjKBPE2yDFb0kigIji2xGTYcxXU-rCTrXNQiN4N4_RpUja5SPyx99Eh46Cixcag&tag=1-1684724589-unknown-0-0eoiahcqin-0a0841bce7d498e5&clientCacheKey=3xt93xmd7emzv6k_b.mp4";
        String videoUrl2 = "https://v2.kwaicdn.com/upic/2023/05/20/16/BMjAyMzA1MjAxNjAyMjhfMTM2MDg5NzAyNl8xMDM1MjIwNDgzMjFfMl8z_b_B0d97abd06757be9906cd83e0571dcd7d.mp4?pkey=AAX4DgH_Q4KBjVVQpeITMyNxi34_0KOD7Dp80qxpwuNV4BfONaasSnAkBrFPEdGKbfj0m9Jd_VGG-isTQxiQGOlUQ6e2OKuPO2f72pFUVdxdEJkqCVpijL4zsJu3mMtoBfc&tag=1-1684724589-unknown-0-hilddj6arr-772b90a0f590e795&clientCacheKey=3xhd9ng5fbii652_b.mp4";
        MediaItem mediaItem = MediaItem.fromUri(videoUrl);
        MediaItem mediaItem2 = MediaItem.fromUri(videoUrl2);
        mPlayer.setMediaItem(mediaItem);
        mPlayer2.setMediaItem(mediaItem2);
        mPlayer.prepare();
        mPlayer2.prepare();
 
 
    }
 
//        以下是生命周期管理
    @Override
    protected void onStart() {
        super.onStart();
        mPlayerView.onResume();
    }
 
    @Override
    protected void onStop() {
        super.onStop();
        mPlayerView.onPause();
    }
 
    @Override
    protected void onDestroy() {
        super.onDestroy();
        mPlayer.release();
    }
 
//        注意：在使用ExoPlayer时，需要在Activity中保持屏幕常亮以避免视频播放过程中屏幕自动关闭。
    @Override
    protected void onResume() {
        super.onResume();
        getWindow().addFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON);
    }
 
    @Override
    protected void onPause() {
        super.onPause();
        getWindow().clearFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON);
    }
}
```

## 二、自定义播放暂停

#### 1、首先如何隐藏默认的开始暂停和快进？

​    隐藏它们很简单，只需把use_controller设置成false即可。

```xml
<com.google.android.exoplayer2.ui.StyledPlayerView
        android:id="@+id/videoView"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        app:use_controller="false"/>
```

#### 2、自定义

 ExoPlayer还提供了很多控制视频播放的API，如暂停、继续、快进、快退、调整音量、调整亮度等。下面将介绍如何实现视频的暂停和继续播放。

```java
private boolean isPlaying = false;
 
private void togglePlay() {
    if (isPlaying) {
        player.pause();
    } else {
        player.play();
    }
    isPlaying = !isPlaying;
}
```

利用isPlaying布尔变量记录当前视频的播放状态，使用player.pause()方法暂停视频，使用player.play()方法继续播放视频，并且通过isPlaying变量更新当前播放状态。

在SimpleExoPlayerView中添加点击事件：

```java
playerView.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View view) {
        togglePlay();
    }
});
```

这样就可以实现通过点击视频区域来暂停和继续视频的播放。

## 三、控制视频画面旋转和比例调整

ExoPlayer支持旋转视频画面和调整视频比例的功能。下面将介绍如何使用ExoPlayer来实现这些功能。

使用代码旋转视频画面，可以调用SimpleExoPlayerView.setUseController(false)方法隐藏内置的控制器，然后利用下面的代码实现视频旋转：

```java
playerView.setKeepScreenOn(true);
playerView.setRotation(90);
playerView.setResizeMode(AspectRatioFrameLayout.RESIZE_MODE_FIT);
```

通过playerView.setKeepScreenOn(true)方法保持屏幕常亮，playerView.setRotation(90)方法实现视频的旋转，playerView.setResizeMode(AspectRatioFrameLayout.RESIZE_MODE_FIT)方法调整视频的比例。

## 四、全屏放大和缩小

实现视频画面的缩放，可以通过以下代码实现：

```java
private boolean isFullscreen = false;
 
private void toggleFullscreen() {
    if (isFullscreen) {
        setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_PORTRAIT);
        getWindow().clearFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN);
        playerView.setResizeMode(AspectRatioFrameLayout.RESIZE_MODE_FIT);
        getSupportActionBar().show();
    } else {
        setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE);
        getWindow().addFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN);
        playerView.setResizeMode(AspectRatioFrameLayout.RESIZE_MODE_FILL);
        getSupportActionBar().hide();
    }
    isFullscreen = !isFullscreen;
}
```

利用isFullscreen布尔变量记录当前是否全屏状态，使用setRequestedOrientation()方法实现屏幕的旋转，使用getWindow().addFlags()和getWindow().clearFlags()方法实现全屏状态的切换，并且使用playerView.setResizeMode()方法调整视频的比例。在全屏状态下隐藏ActionBar，退出全屏状态后显示ActionBar。

#### 双击视频放大缩小

在SimpleExoPlayerView中添加双击事件：

```java
playerView.setOnTouchListener(new View.OnTouchListener() {
    private GestureDetector gestureDetector = new GestureDetector(MainActivity.this,
            new GestureDetector.SimpleOnGestureListener() {
                @Override
                public boolean onDoubleTap(MotionEvent e) {
                    toggleFullscreen();
                    return super.onDoubleTap(e);
                }
            });
 
    @Override
    public boolean onTouch(View view, MotionEvent motionEvent) {
        gestureDetector.onTouchEvent(motionEvent);
        return true;
    }
});
```

这样就可以实现在双击视频区域时切换视频的全屏状态。

#### 按钮放大缩小

```java
        mBtnZoom.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                toggleFullscreen();
            }
       });
```

## 五、完整的实现代码

如果看不懂前面的操作，可以直接用下面的代码，有点击按钮和双击视频全屏放大缩小效果。

```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">
 
    <com.google.android.exoplayer2.ui.PlayerView
        android:id="@+id/player_view"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />
 
    <Button
        android:id="@+id/btn_zoom"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_alignParentBottom="true"
        android:layout_alignParentRight="true"
        android:layout_margin="16dp"
        android:backgroundTint="@color/white"
        android:text="Full Screen"
        android:textColor="@color/black"
        android:visibility="visible"
        android:onClick="onFullscreenButtonClick"/>
 
 
</RelativeLayout>
```



```java
public class MainActivity extends AppCompatActivity {
    private SimpleExoPlayer mPlayer;
    private PlayerView mPlayerView;
    private Button mBtnZoom;
    private boolean isFullscreen = false;
 
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
 
        mPlayerView = findViewById(R.id.player_view);
        mBtnZoom = findViewById(R.id.btn_zoom);
 
//         创建媒体播放器
        mPlayer = new SimpleExoPlayer.Builder(this).build();
        mPlayerView.setPlayer(mPlayer);
 
//        准备视频源
        String videoUrl = "https://v2.kwaicdn.com/upic/2023/04/14/13/BMjAyMzA0MTQxMzA3MDhfNDgwOTQ5MTg2XzEwMDU1NzkwNjgxMV8yXzM=_b_B4856a8b07ea4dbc4ad503f04681b5ec8.mp4?pkey=AAW-JkY6q-BoTEXl2FArO1ReMGt965IaFcPl0NGX7cYTVhLtgZVdld15RtYUAWwkkEmBjjKBPE2yDFb0kigIji2xGTYcxXU-rCTrXNQiN4N4_RpUja5SPyx99Eh46Cixcag&tag=1-1684724589-unknown-0-0eoiahcqin-0a0841bce7d498e5&clientCacheKey=3xt93xmd7emzv6k_b.mp4";
        String videoUrl2 = "https://v2.kwaicdn.com/upic/2023/05/20/16/BMjAyMzA1MjAxNjAyMjhfMTM2MDg5NzAyNl8xMDM1MjIwNDgzMjFfMl8z_b_B0d97abd06757be9906cd83e0571dcd7d.mp4?pkey=AAX4DgH_Q4KBjVVQpeITMyNxi34_0KOD7Dp80qxpwuNV4BfONaasSnAkBrFPEdGKbfj0m9Jd_VGG-isTQxiQGOlUQ6e2OKuPO2f72pFUVdxdEJkqCVpijL4zsJu3mMtoBfc&tag=1-1684724589-unknown-0-hilddj6arr-772b90a0f590e795&clientCacheKey=3xhd9ng5fbii652_b.mp4";
        MediaItem mediaItem = MediaItem.fromUri(videoUrl);
        MediaItem mediaItem2 = MediaItem.fromUri(videoUrl2);
        mPlayer.setMediaItem(mediaItem);
        mPlayer.prepare();
 
 
//        按钮放大缩小
        mBtnZoom.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                toggleFullscreen();
            }
        });
 
 
//        双击视频放大缩小
        mPlayerView.setOnTouchListener(new View.OnTouchListener() {
            private GestureDetector gestureDetector = new GestureDetector(MainActivity.this,
                    new GestureDetector.SimpleOnGestureListener() {
                        @Override
                        public boolean onDoubleTap(MotionEvent e) {
                            toggleFullscreen();
                            return super.onDoubleTap(e);
                        }
                    });
 
            @Override
            public boolean onTouch(View view, MotionEvent motionEvent) {
                gestureDetector.onTouchEvent(motionEvent);
                return true;
            }
        });
 
 
    }
 
 
 
//    视频放大缩小方法
    private void toggleFullscreen() {
        if (isFullscreen) {
            setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_PORTRAIT);
            getWindow().clearFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN);
            mPlayerView.setResizeMode(AspectRatioFrameLayout.RESIZE_MODE_FIT);
            getSupportActionBar().show();
        } else {
            setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE);
            getWindow().addFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN);
            mPlayerView.setResizeMode(AspectRatioFrameLayout.RESIZE_MODE_FILL);
            getSupportActionBar().hide();
        }
        isFullscreen = !isFullscreen;
    }
 
 
//    以下是生命周期
    @Override
    protected void onStart() {
        super.onStart();
        mPlayerView.onResume();
    }
 
    @Override
    protected void onStop() {
        super.onStop();
        mPlayerView.onPause();
    }
 
    @Override
    protected void onDestroy() {
        super.onDestroy();
        mPlayer.release();
    }
 
 
//    注意：在使用ExoP注意：在使用ExoPlayer时，需要在Activity中保持屏幕常亮以避免视频播放过程中屏幕自动关闭。layer时，需要在Activity中保持屏幕常亮以避免视频播放过程中屏幕自动关闭。
    @Override
    protected void onResume() {
        super.onResume();
        getWindow().addFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON);
    }
 
    @Override
    protected void onPause() {
        super.onPause();
        getWindow().clearFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON);
    }
}
```

`MediaPlayer` 是 Android 中用于播放音频和视频的类。它提供了一系列方法来控制媒体播放、暂停、停止等操作，可以从本地文件、网络流媒体或者资源文件中读取并播放音视频数据。

### 1. **MediaPlayer 基本用法**

使用 `MediaPlayer` 时，首先需要实例化它，然后设置数据源（音频或视频文件），最后进行播放操作。以下是一个典型的音频播放示例：

```java
// 创建 MediaPlayer 实例
MediaPlayer mediaPlayer = new MediaPlayer();

try {
    // 设置数据源，可以是文件路径、URL、资源文件等
    mediaPlayer.setDataSource("http://www.example.com/audio.mp3");
    
    // 准备播放
    mediaPlayer.prepare();  // prepare() 是同步的，prepareAsync() 是异步的
    // 或者 mediaPlayer.prepareAsync(); 来异步准备
    
    // 开始播放
    mediaPlayer.start();
} catch (IOException e) {
    e.printStackTrace();
}

// 在播放完成后释放资源
mediaPlayer.setOnCompletionListener(new MediaPlayer.OnCompletionListener() {
    @Override
    public void onCompletion(MediaPlayer mp) {
        // 播放完成后进行资源释放
        mediaPlayer.release();
    }
});
```

### 2. **常见方法**

- `setDataSource(String path)`：设置播放的数据源，可以是文件路径、网络 URL 或者资源文件。
- `prepare()`：同步准备播放（会阻塞当前线程），建议使用 `prepareAsync()` 进行异步准备。
- `prepareAsync()`：异步准备播放，通常在 UI 线程上使用。
- `start()`：开始播放。
- `pause()`：暂停播放。
- `stop()`：停止播放。
- `release()`：释放资源，调用后不能再使用该 `MediaPlayer` 实例。
- `seekTo(int msec)`：跳转到指定的播放位置（以毫秒为单位）。

### 3. **生命周期与回调**

`MediaPlayer` 的生命周期包括准备、播放、暂停和释放等，通常需要监听相关的回调：

- `OnCompletionListener`：播放完成后的回调。
- `OnErrorListener`：播放过程中遇到错误时的回调。
- `OnPreparedListener`：准备完成后回调，适用于 `prepareAsync()`。
- `OnInfoListener`：用于接收播放过程中的各种信息。

例如：

```java
mediaPlayer.setOnErrorListener(new MediaPlayer.OnErrorListener() {
    @Override
    public boolean onError(MediaPlayer mp, int what, int extra) {
        // 处理错误
        return false;
    }
});
```

### 4. **使用资源文件**

如果你想播放资源文件，可以使用 `getResources().openRawResourceFd()` 来获取文件的描述符，例如：

```java
AssetFileDescriptor afd = getResources().openRawResourceFd(R.raw.audio_file);
MediaPlayer mediaPlayer = new MediaPlayer();
mediaPlayer.setDataSource(afd.getFileDescriptor(), afd.getStartOffset(), afd.getLength());
mediaPlayer.prepare();
mediaPlayer.start();
```

### 5. **管理音量**

`MediaPlayer` 本身不直接控制设备音量，而是依赖于 `AudioManager` 来调整。你可以在播放时设置音量：

```java
AudioManager audioManager = (AudioManager) getSystemService(Context.AUDIO_SERVICE);
audioManager.setStreamVolume(AudioManager.STREAM_MUSIC, 5, 0); // 设置音量
```

### 6. **播放视频**

`MediaPlayer` 也支持视频播放，通常配合 `SurfaceView` 或 `TextureView` 使用：

```java
MediaPlayer mediaPlayer = new MediaPlayer();
SurfaceView surfaceView = findViewById(R.id.surfaceView);
SurfaceHolder holder = surfaceView.getHolder();
mediaPlayer.setDisplay(holder);
mediaPlayer.setDataSource("http://www.example.com/video.mp4");
mediaPlayer.prepare();
mediaPlayer.start();
```

### 7. **异步任务与状态管理**

`MediaPlayer` 的 `prepareAsync()` 方法常用于异步准备操作，尤其是在加载远程音频文件时。它会在准备完成时触发 `OnPreparedListener`，而且不会阻塞主线程，适合进行 UI 更新。

```java
java复制代码mediaPlayer.setOnPreparedListener(new MediaPlayer.OnPreparedListener() {
    @Override
    public void onPrepared(MediaPlayer mp) {
        // 准备完成，可以开始播放
        mediaPlayer.start();
    }
});
mediaPlayer.prepareAsync();
```