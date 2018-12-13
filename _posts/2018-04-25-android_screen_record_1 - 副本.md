---
layout: post
title: Android录屏之简单录屏
modified:
categories: 
tags: [Android, MediaCodec, 音视频]
share:
catalog: true
date: 2018-04-25T11:09:21+08:00
---

最近项目里面用到了录屏的功能，前后折腾了一周时间，终于将录屏功能彻底搞出来了，基本上了解了音视频的录制， 帧率的控制，利用mediacodec进行音视频的编码，音视频时间戳的同步问题。本文作为这个Android录屏系列文章的开篇主要大致讲一下录屏的大致流程，包含哪些功能。

#### 简介

对于录屏功能，通常会包含几个方面，视频的录制与编码，音频的录制与编码，音视频混合打包成mp4文件。
我们将每个步骤拆开：
获取视频帧     将视频帧丢到编码器里面编码   得到编码后的数据丢给混合器混合
获取音频数据   将音频数据丢到编码器里面编码   得到编码后的数据丢给混合器混合

其中需要注意一点的就是：
1、控制帧率的问题：由于编码好的视频帧丢弃会导致一些乱七八糟的问题，所以我们只能够控制编码器输入的视频帧，
2、音视频的同步问题，控制视频帧与音频帧的同步，防止发生一些视频播放完了，声音还在播放什么的一些问题



#### 实现

MediaProjection获取
{% highlight java %}

mMediaProjectionManager = (MediaProjectionManager) getSystemService(MEDIA_PROJECTION_SERVICE);
Intent captureIntent = mMediaProjectionManager.createScreenCaptureIntent();
startActivityForResult(captureIntent, REQUEST_CODE);

@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    MediaProjection mediaProjection = mMediaProjectionManager.getMediaProjection(resultCode, data);
    if (mediaProjection == null) {
        Log.e(TAG, "media projection is null");
        return;
    }
}

{% endhighlight %}

要获取MediaProjection，需要我们动态的得到录屏权限才能够进一步拿到mediaprojection对象。然后通过MediaProjection对象去创建一个VirtualDisplay，

{% highlight java %}

mVirtualDisplay = mMediaProjection.createVirtualDisplay("test-display",
                mWidth, mHeight, mDpi, DisplayManager.VIRTUAL_DISPLAY_FLAG_PUBLIC,
                mSurface, null, null);

{% endhighlight %}

这个过程很简单，但是需要注意一点: 参数mSurface是通过MediaCodec创建的，而不是我们自己new出来的(如果需要控制帧率的话，可以自己new一个出来，后面我们再说如何控制帧率的问题)
<!--more-->
既然mSurface是由MediaCodec创建的，那么我们就需要知道MediaCodec的初始化：

{% highlight java %}

MediaFormat format = MediaFormat.createVideoFormat(MIME_TYPE, mWidth, mHeight);
format.setInteger(MediaFormat.KEY_COLOR_FORMAT,
        MediaCodecInfo.CodecCapabilities.COLOR_FormatSurface);
format.setInteger(MediaFormat.KEY_BIT_RATE, mBitRate);
format.setInteger(MediaFormat.KEY_FRAME_RATE, FRAME_RATE);
format.setInteger(MediaFormat.KEY_I_FRAME_INTERVAL, IFRAME_INTERVAL);

mEncoder = MediaCodec.createEncoderByType(MIME_TYPE);
mEncoder.configure(format, null, null, MediaCodec.CONFIGURE_FLAG_ENCODE);
mSurface = mEncoder.createInputSurface();
mEncoder.start();

{% endhighlight %}

这里比较简单就是一堆参数的配置，然后创建一个编码器mEncoder，通过createInputSurface()方法从mEncoder中得到一个mSurface。
这样VirtualDisplay与MediaCodec就能关联起来：
VirtualDisplay输出的mSurface就是MediaCodec的输入

数据走向就是VirtualDisplay----->mSurface----->MediaCodec

VirtualDisplay会自动将屏幕数据渲染到surface上，MediaCodec会不断的从mSurface中获取数据进行编码

当MeidaCodec编码完以后我们需要处理掉编码后的数据。
处理的逻辑如下：
在子线程里面循环从MediaCodec的输出队列里拿数据并交给MediaMuxer进行混合。

{% highlight java %}

while (!mQuit.get()) {
    int outputBufferId = mEncoder.dequeueOutputBuffer(mBufferInfo, TIMEOUT_US);
    Log.i(TAG, "dequeue output buffer outputBufferAudioId=" + outputBufferId);

    if (outputBufferId == MediaCodec.INFO_OUTPUT_FORMAT_CHANGED) {
        resetOutputFormat();
    } else if (outputBufferId == MediaCodec.INFO_TRY_AGAIN_LATER) {
        Log.d(TAG, "retrieving buffers time out!");
        try {
            Thread.sleep(10);
        } catch (InterruptedException e) {
        }
    } else if (outputBufferId >= 0) {
        if (!mMuxerStarted) {
            throw new IllegalStateException("MediaMuxer dose not call addTrack(format) ");
        }

        // 根据outputBufferId获取ByteBuffer后,写入MediaMuxer
        encodeToVideoTrack(outputBufferId);
        mEncoder.releaseOutputBuffer(outputBufferId, false);
    }

}

{% endhighlight %}

在while循环中不断的去从MediaCodec的输出队列中拿数据，先取得队列的状态outputBufferId，
MediaCodec在一开始调用dequeueOutputBuffer()时会返回一次INFO_OUTPUT_FORMAT_CHANGED消息。我们只需在这里获取该MediaCodec的format，并注册到MediaMuxer里。接着判断当前audio track和video track是否都已就绪，如果是的话就启动Muxer。
如果状态为-1（INFO_TRY_AGAIN_LATER）表示没有数据过来，我们这边采用的是直接sleep(),当然也可以采用其他的方式。
如果说outputBufferId>0 那么表示队列里面有数据，我们可以直接通过outputBufferId从队列中获取编码后的数据encodedData。如下

{% highlight java %}

private void encodeToVideoTrack(int outputBufferId) {
    ByteBuffer encodedData = mEncoder.getOutputBuffers()[outputBufferId];
    if ((mBufferInfo.flags & MediaCodec.BUFFER_FLAG_CODEC_CONFIG) != 0) {
        // The codec config data was pulled out and fed to the muxer when we got
        // the INFO_OUTPUT_FORMAT_CHANGED status.
        // Ignore it.
        Log.d(TAG, "ignoring BUFFER_FLAG_CODEC_CONFIG");
        mBufferInfo.size = 0;
    }

    if (mBufferInfo.size == 0) {
        Log.d(TAG, "info.size == 0, drop it.");
        encodedData = null;
    }

    if (encodedData != null) {
        encodedData.position(mBufferInfo.offset);
        encodedData.limit(mBufferInfo.offset + mBufferInfo.size);

        // 获取流数据，从ByteBuffer 获取  byte[]，可以将byte[]传至后台
        byte[] bytes = new byte[encodedData.remaining()];
        encodedData.get(bytes, 0, bytes.length);
        mMuxer.writeSampleData(mVideoTrackIndex, encodedData, mBufferInfo);
        Log.i(TAG, "sent " + mBufferInfo.size + " bytes to muxer...");
    }
}

{% endhighlight %}
这里就是从输出队列OutputBuffers中获取数据然后丢到MeidaMuxer，最后启动MediaMuxer进行混合生成文件，如下
{% highlight java %}

private void resetOutputFormat() {
    if (mMuxerStarted) {
        throw new IllegalStateException("output format already changed!");
    }
    MediaFormat newFormat = mEncoder.getOutputFormat();
    newFormat.getByteBuffer("csd-0");    // SPS
    newFormat.getByteBuffer("csd-1");    // PPS
    Log.i(TAG, "output format changed.\n new format: " + newFormat.toString());
    mVideoTrackIndex = mMuxer.addTrack(newFormat);
    mMuxer.start();
    mMuxerStarted = true;
    Log.i(TAG, "started media muxer, videoIndex=" + mVideoTrackIndex);
}

{% endhighlight %}
到这里视频的录制就结束了，但还有很多问题，比如怎么同时把音频加入进去，怎么去控制帧率(对于直播或者手机差的情况就需要控制帧率)，音频的加入下一篇文章再说。