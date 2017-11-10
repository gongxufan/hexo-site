---
layout: post
title: "【基于Ijkplayer】安卓机顶盒电视播放器开发"
date: 2017-10-30 17:27
tags: [java,android]
category: 随笔
description: 开发一个安卓机顶盒的电视直播app。

---
## 前言
Ijkplayer是B站开源的一款多媒体播放引擎，其基于ffmpeg开并支持很多的在线媒体播放格式。本文实现了在安卓TV上播放各大电视台的直播，其格式是.m3u8。当然了只要是编译的Ijkplayer支持的格式这款播放器自然可以播放。

首先上界面：
![img](/upload/images/android/tv-player/1.jpeg)
![img](/upload/images/android/tv-player/2.jpeg)
播放器界面非常简单，因为是用电视遥控器操作，所以布局和操作力求简单易用。布局分为播放区域和左侧菜单，按遥控器的确定按钮调出菜单，可以选择界面确认后进入播放界面。下面开始讲如何实现：
## 一，创建布局
播放器主界面布局文件activity_play_rel.xml
```xml
<?xml version="1.0" encoding="utf-8"?>  
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"  
    android:layout_width="match_parent"  
    android:layout_height="match_parent"  
    android:orientation="vertical">  
  
    <LinearLayout  
        android:layout_width="fill_parent"  
        android:layout_height="fill_parent"  
        android:orientation="vertical">  
  
        <RelativeLayout  
            android:id="@+id/fl_videoview"  
            android:layout_width="match_parent"  
            android:layout_height="match_parent"  
            android:background="@color/colorBlack">  
  
            <com.gxf.liveplay.ijkplayer.media.IjkVideoView  
                android:id="@+id/videoview"  
                android:layout_width="match_parent"  
                android:layout_height="match_parent"  
                android:layout_centerInParent="true"  
                android:background="@color/colorBlack"></com.gxf.liveplay.ijkplayer.media.IjkVideoView>  
  
            <RelativeLayout  
                android:id="@+id/rl_loading"  
                android:layout_width="match_parent"  
                android:layout_height="match_parent"  
                android:background="#de262a3b">  
  
                <TextView  
                    android:id="@+id/tv_load_msg"  
                    android:layout_width="wrap_content"  
                    android:layout_height="wrap_content"  
                    android:layout_below="@+id/pb_loading"  
                    android:layout_centerInParent="true"  
                    android:layout_marginTop="6dp"  
                    android:textColor="#ffffff"  
                    android:textSize="16sp" />  
  
                <ProgressBar  
                    android:id="@+id/pb_loading"  
                    android:layout_width="60dp"  
                    android:layout_height="60dp"  
                    android:layout_centerInParent="true"  
                    android:layout_marginTop="60dp"  
                    android:indeterminate="false"  
                    android:indeterminateDrawable="@drawable/video_loading"  
                    android:padding="5dp" />  
            </RelativeLayout>  
  
            <!-- <LinearLayout  
                 android:layout_width="match_parent"  
                 android:layout_height="wrap_content"  
                 android:background="@color/player_panel_background_color">  
  
                 <TextView  
                     android:id="@+id/tv_title"  
                     android:layout_width="wrap_content"  
                     android:layout_height="60dp"  
                     android:textSize="24dp"  
                     android:text=""  
                     android:layout_centerVertical="true"  
                     android:layout_marginTop="18dp"  
                     android:textColor="@color/white"/>  
  
                 <TextView  
                     android:id="@+id/tv_time"  
                     android:layout_width="wrap_content"  
                     android:layout_height="60dp"  
                     android:textSize="20dp"  
                     android:layout_toRightOf="@id/tv_title"  
                     android:layout_alignParentRight="true"  
                     android:layout_centerVertical="true"  
                     android:layout_marginLeft="60dp"  
                     android:layout_marginTop="20dp"  
                     android:textColor="@color/white"/>  
             </LinearLayout>-->  
        </RelativeLayout>  
    </LinearLayout>  
  
    <include layout="@layout/layout_menu" />  
</RelativeLayout>  
```
整体分为两部分：上边为播放器实现的界面布局，下边为左侧菜单布局。这里没有采用滑动调出左侧菜单的效果，只是采用相对布局通过设定菜单的宽带直接覆盖在播放器布局之上。IjkVidoView下边的是播放器的控制器布局，可以扩展实现进度音量等控制，本文尚未实现。

接下来看左侧菜单布局文件layout_menu.xml
```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"  
    android:layout_width="360dp"  
    android:layout_height="match_parent"  
    android:id="@+id/leftMenuCtrl"  
    android:layout_gravity="left"  
    android:visibility="gone"  
    android:background="@drawable/dark_no_shadow"  
    android:orientation="vertical">  
  
    <LinearLayout  
        android:layout_width="wrap_content"  
        android:layout_height="0dp"  
        android:layout_weight="1"  
        android:layout_gravity="center"  
        android:orientation="horizontal">  
  
        <ImageButton  
            android:id="@+id/turnLeft"  
            android:layout_width="30px"  
            android:layout_height="40px"  
            android:layout_gravity="center"  
            android:adjustViewBounds="false"  
            android:background="@drawable/tv_switch_bg_left"  
            android:onClick="turnLeft" />  
  
        <TextView  
            android:id="@+id/tvGroupTitle"  
            android:layout_width="300px"  
            android:layout_height="100px"  
            android:layout_gravity="center"  
            android:gravity="center"  
            android:text="我的频道"  
            android:textColor="@color/white"  
            android:textSize="24sp"  
            android:textStyle="bold"  />  
  
        <ImageButton  
            android:id="@+id/turnRight"  
            android:layout_width="30px"  
            android:layout_height="40px"  
            android:layout_gravity="center"  
            android:adjustViewBounds="false"  
            android:background="@drawable/tv_switch_bg_right"  
            android:onClick="turnRight" />  
  
    </LinearLayout>  
  
    <ListView  
        android:id="@+id/listView"  
        android:layout_width="match_parent"  
        android:layout_height="0dp"  
        android:layout_weight="9"  
        android:layout_gravity="center"  
        android:listSelector="@color/colorPrimaryDark"/>  
</LinearLayout>  
```
上边的LinearLayout用于切换节目组，下边是ListView显示每个分组下的节目列表。这次布局就实现了，这是相当简单的。
 ## 二，实现播放
LiveActivityRel实现了播放功能，启动该Activity的时候传入媒体资源的地址mVideoUrl，然后启动Ijkplayer的播放功能
```java
public void initVideo() {  
        // init player`  
        IjkMediaPlayer.loadLibrariesOnce(null);  
        IjkMediaPlayer.native_profileBegin("libijkplayer.so");  
        mVideoView.setVideoURI(Uri.parse(mVideoUrl));  
        mVideoView.setOnPreparedListener(new IMediaPlayer.OnPreparedListener() {  
            @Override  
            public void onPrepared(IMediaPlayer mp) {  
                mp.setVolume(100, 100);  
                mVideoView.start();  
            }  
        });  
  
        mVideoView.setOnInfoListener(new IMediaPlayer.OnInfoListener() {  
            @Override  
            public boolean onInfo(IMediaPlayer mp, int what, int extra) {  
                switch (what) {  
                    case IjkMediaPlayer.MEDIA_INFO_BUFFERING_START:  
                        mLoadingLayout.setVisibility(View.VISIBLE);  
                        break;  
                    case IjkMediaPlayer.MEDIA_INFO_VIDEO_RENDERING_START:  
                    case IjkMediaPlayer.MEDIA_INFO_BUFFERING_END:  
                        mLoadingLayout.setVisibility(View.GONE);  
                        break;  
                }  
                return false;  
            }  
        });  
  
        mVideoView.setOnCompletionListener(new IMediaPlayer.OnCompletionListener() {  
            @Override  
            public void onCompletion(IMediaPlayer mp) {  
                mLoadingLayout.setVisibility(View.VISIBLE);  
                mVideoView.stopPlayback();  
                mVideoView.release(true);  
                mVideoView.setVideoURI(Uri.parse(mVideoUrl));  
            }  
        });  
  
        mVideoView.setOnErrorListener(new IMediaPlayer.OnErrorListener() {  
            @Override  
            public boolean onError(IMediaPlayer mp, int what, int extra) {  
                if (++mRetryTimes > CONNECTION_TIMES) {  
                    new AlertDialog.Builder(LiveActivityRel.this)  
                            .setMessage("该频道媒体资源出现错误，节目暂时不能播放...")  
                            .setPositiveButton(R.string.VideoView_error_button,  
                                    new DialogInterface.OnClickListener() {  
                                        public void onClick(DialogInterface dialog, int whichButton) {  
                                            dialog.dismiss();  
                                            //LiveActivityRel.this.finish();  
                                        }  
                                    })  
                            .setCancelable(false)  
                            .show();  
                } else {  
                    mVideoView.stopPlayback();  
                    mVideoView.release(true);  
                    mVideoView.setVideoURI(Uri.parse(mVideoUrl));  
                }  
                return false;  
            }  
        });  
  
    }  
```
这里也是很简答的，首先进行库的加载和初始化，绑定各种事件，然后在setOnPreparedListener中启动播放。其余的功能主要是涉及到UI和数据解析。节目列表是一个JSON结构，在原来的项目是有API进行登录和节目列表获取的。为了方便本文直接把节目列表写死，同时启动的Activity设置为MainActivityOffline。
```java
package com.gxf.liveplay;  
  
import android.app.Activity;  
import android.content.Context;  
import android.content.Intent;  
import android.content.SharedPreferences;  
import android.os.Bundle;  
import android.widget.Toast;  
  
/** 
 * splash 
 * 启动登录检测 
 */  
public class MainActivityOffline extends Activity {  
    private final int SPLASH_DISPLAY_LENGHT = 1000; // 两秒后进入系统  
    private boolean isUpdateChecked = false;  
    public void checkLogin(){  
        SharedPreferences userSettings = getSharedPreferences("auth", Context.MODE_PRIVATE);  
        final String lastVedioUrl = userSettings.getString("lastVedioUrl", "none");  
        final Toast toast = Toast.makeText(MainActivityOffline.this, "系统初始化失败...", Toast.LENGTH_SHORT);  
        // 开启一个子线程，进行网络操作，等待有返回结果，使用handler通知UI  
        new Thread(new Runnable() {  
            @Override  
            public void run() {  
                try {  
                    PlayListCache.initPlayInfo(null,null);  
                    String url = lastVedioUrl;  
                    if (url.equals("none"))  
                        url = PlayListCache.playListMap.get(PlayListCache.playListMap.keySet().iterator().next());  
                    LiveActivityRel.activityStart(MainActivityOffline.this, url);  
                    MainActivityOffline.this.finish();  
                } catch (Exception e) {  
                    toast.show();  
                    e.printStackTrace();  
                }  
            }  
        }).start();  
    }  
    @Override  
    protected void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.activity_main);  
        checkLogin();  
    }  
}  
```
启动App在一个线程里去获取节目列表并缓存起来放在PlayListCache中，然后调用LiveActivityRel并传入一个默认的播放地址(首次启动)或者上次播放的地址。至于登录和HttpUtils中的代码(主要用于节目列表和登录控制)就不在赘述了，因为不属于本文的重点。
## 三，项目的代码
[https://github.com/gongxufan/TvPlayer](https://github.com/gongxufan/TvPlayer)
## THE END
PS：本项目中的播放源地址可能会失效，如需测试需要更改可用的播放源。

具体代码在com.gxf.liveplay.HttpUtils#getOfflinePlayList，数据格式如下：
```json
[  
  {  
    "group": "省内频道",  
    "list": [  
      {  
        "湖南都市": "http://220.248.175.231:6610/001/2/ch00000090990000001049/index.m3u8?virtualDomain=001.live_hls.zte.com"  
      },  
      {  
        "湖南经视": "http://220.248.175.230:6610/001/2/ch00000090990000001052/index.m3u8?virtualDomain=001.live_hls.zte.com"  
      }  
    ]  
  },  
  {  
    "group": "其他频道",  
    "list": [  
      {  
        "安徽卫视": "http://220.248.175.230:6610/001/2/ch00000090990000001024/index.m3u8?virtualDomain=001.live_hls.zte.com"  
      },  
      {  
        "北京卫视": "http://220.248.175.230:6610/001/2/ch00000090990000001025/index.m3u8?virtualDomain=001.live_hls.zte.com"  
      }  
    ]  
  },  
  {  
    "group": "高清频道",  
    "list": [  
      {  
        "CCTV1综合HD": "http://220.248.175.230:6610/001/2/ch00000090990000001075/index.m3u8?virtualDomain=001.live_hls.zte.com"  
      },  
      {  
        "湖南经视HD": "http://220.248.175.230:6610/001/2/ch00000090990000001080/index.m3u8?virtualDomain=001.live_hls.zte.com"  
      }  
    ]  
  },  
  {  
    "group": "央视频道",  
    "list": [  
      {  
        "CCTV1综合": "http://220.248.175.230:6610/001/2/ch00000090990000001075/index.m3u8?virtualDomain=001.live_hls.zte.com"  
      },  
      {  
        "CCTV证券": "http://220.248.175.230:6610/001/2/ch00000090990000001133/index.m3u8?virtualDomain=001.live_hls.zte.com"  
      }  
    ]  
  }  
]  
```