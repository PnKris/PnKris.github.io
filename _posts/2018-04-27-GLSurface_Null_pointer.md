---
layout: post
title: GLSurfaceView内部报空指针问题
modified:
categories: Android
tags: [Android, GLSurfaceView]
share:
catalog: true
date: 2018-04-27
---

直接使用GLSurfaceView的时候遇到了在GLSurfaceView内部报空指针异常。

布局内配置
{% highlight java %}
<android.opengl.GLSurfaceView
    android:id="@+id/gls_view"
    android:layout_width="match_parent"
    android:layout_height="match_parent"/>
{% endhighlight %}
1.异常
崩溃日志
{% highlight java %}
java.lang.NullPointerException
  at android.opengl.GLSurfaceView.surfaceCreated(GLSurfaceView.java:531)
  at android.view.SurfaceView.updateWindow(SurfaceView.java:601)
  at android.view.SurfaceView.access$000(SurfaceView.java:94)
  at android.view.SurfaceView$3.onPreDraw(SurfaceView.java:183)
{% endhighlight %}
(备注：GLSurfaceView属于framework层，有些厂商是定制的，与标准的sdk代码会有些不同。)

找到GLSurfaceView的崩溃的那一行

{% highlight java %}

public void surfaceCreated(SurfaceHolder holder) {
    mGLThread.surfaceCreated();
}

{% endhighlight %}

既然是空指针异常，显然mGLThread对象为空。

由此，找到mGLThread初始化的地方
{% highlight java %}
public void setRenderer(Renderer renderer) {
    checkRenderThreadState();
    if (mEGLConfigChooser == null) {
        mEGLConfigChooser = new SimpleEGLConfigChooser(true);
    }
    if (mEGLContextFactory == null) {
        mEGLContextFactory = new DefaultContextFactory();
    }
    if (mEGLWindowSurfaceFactory == null) {
        mEGLWindowSurfaceFactory = new DefaultWindowSurfaceFactory();
    }
    mRenderer = renderer;
    mGLThread = new GLThread(mThisWeakRef);
    mGLThread.start();
}
{% endhighlight %}
发现mGLThread是在渲染器设置的时候进行初始化

顺便再来看看GLSurfaceView的构造函数
{% highlight java %}

    /**
     * Standard View constructor. In order to render something, you
     * must call {@link #setRenderer} to register a renderer.
     *（这里已经提示我们，必须先设置一个渲染器）
     */
    public GLSurfaceView(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }

{% endhighlight %}
原来是画面加载的时候去调用渲染，但是布局里面没有设置渲染器所以就崩了。

<!--more-->

2.使用
网上常见的GLSurfaceView使用方法
{% highlight java %}
    ...
    GLSurfaceView mGLView = new GLSurfaceView(this); 
    mGLView.setRenderer(new DemoRenderer()); 
    //设置渲染器，其中DemoRenderer实现GLSurfaceView.Renderer

    setContentView(mGLView);
    ...
{% endhighlight %}
如果实在想在布局里使用的话，用DemoGLSurfaceView继承GLSurfaceView 
构造函数里面设置好适配器。

DemoGLSurfaceView.java
{% highlight java %}
    package test.com.asproject;
    
    import android.content.Context;
    import android.opengl.GLSurfaceView;
    import android.util.AttributeSet;
    import android.view.MotionEvent;
    
    import javax.microedition.khronos.egl.EGLConfig;
    import javax.microedition.khronos.opengles.GL10;

    /**
     * Created by Seselin on 2016/4/1.
     */
    public class DemoGLSurfaceView extends GLSurfaceView {
        DemoRenderer mRenderer;
        
        public DemoGLSurfaceView(Context context) {
            super(context);
            //为了可以激活log和错误检查，帮助调试3D应用，需要调用setDebugFlags()。
            this.setDebugFlags(DEBUG_CHECK_GL_ERROR | DEBUG_LOG_GL_CALLS);
            mRenderer = new DemoRenderer();
            this.setRenderer(mRenderer);
        }
    
        public DemoGLSurfaceView(Context context, AttributeSet attrs) {
            super(context, attrs);
            //为了可以激活log和错误检查，帮助调试3D应用，需要调用setDebugFlags()。
            this.setDebugFlags(DEBUG_CHECK_GL_ERROR | DEBUG_LOG_GL_CALLS);
            mRenderer = new DemoRenderer();
            this.setRenderer(mRenderer);
        }
    
        public boolean onTouchEvent(final MotionEvent event) {
            //由于DemoRenderer对象运行在另一个线程中，这里采用跨线程的机制进行处理。使用queueEvent方法
            //当然也可以使用其他像Synchronized来进行UI线程和渲染线程进行通信。
            this.queueEvent(new Runnable() {
    
                @Override
                public void run() {
                }
            });
    
            return true;
        }
    
        class DemoRenderer implements GLSurfaceView.Renderer {
    
            @Override
            public void onDrawFrame(GL10 gl) {
                //每帧都需要调用该方法进行绘制。绘制时通常先调用glClear来清空framebuffer。
                //然后调用OpenGL ES其他接口进行绘制
                gl.glClear(GL10.GL_COLOR_BUFFER_BIT | GL10.GL_DEPTH_BUFFER_BIT);
    
            }
    
            @Override
            public void onSurfaceChanged(GL10 gl, int w, int h) {
                //当surface的尺寸发生改变时，该方法被调用，。往往在这里设置ViewPort。或者Camara等。
                gl.glViewport(0, 0, w, h);
            }
    
            @Override
            public void onSurfaceCreated(GL10 gl, EGLConfig config) {
                // 该方法在渲染开始前调用，OpenGL ES的绘制上下文被重建时也会调用。
                //当Activity暂停时，绘制上下文会丢失，当Activity恢复时，绘制上下文会重建。
            }
        }
    }
{% endhighlight %}
布局内配置
{% highlight java %}
    <test.com.asproject.DemoGLSurfaceView
        android:id="@+id/gls_view"
        android:layout_width="match_parent"
        android:layout_height="match_parent">
    </test.com.asproject.DemoGLSurfaceView>

{% endhighlight %}
到此,GLSurfaceView已经可以使用了。

bug场景：
    
在项目中遇到过这种问题，render是在recorder库中设置到glSurfaceview的，但是recoder初始化的时候会初始化失败，所以我们采用了子线程进行初始化，如果失败会隔1000ms进行重试一次，这里注意到render实际上是在子线程中异步初始化的，而GLSurfaceView的初始化要在recoder初始化之前进行的，我们的GlSurfaceView初始化代码类似于下面：
{% highlight java %}
    GLSurfaceView glSurfaceView;
    private void initGlSurfaceViewIfNeeded() {
        if (glSurfaceView==null) {
            glSurfaceView = new new GLSurfaceView(getActivity());
            glSurfaceView.getHolder().addCallback(this);
            
            FrameLayout.LayoutParams params = new FrameLayout.LayoutParams(
                    ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT);
            mFlRecorder.addView(glSurfaceView, 0, params);
        }
    }
    
    private void initRecorder() {
        ThreadPoolManager.cacheExecute(new Runnable() {
            @Override
            public void run() {
            
                //initRecorder()存在失败的可能性，需要重试几次
                while (!(success = initRecorder(getActivity())) && retry < 5) {
                    try {
                        retry++;
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        });
    }
{% endhighlight %}
这样一来我们的代码就会有个问题, GLSurfaceView添加到FrameLayout上了以后就会执行内部的初始化方法，如果执行到
{% highlight java %}
    public void surfaceCreated(SurfaceHolder holder) {
        mGLThread.surfaceCreated();
    }

{% endhighlight %}
而我们在初始化recoder这个东西的时候，发生了意外，导致setRender()方法还没执行到，那么就会发生空指针异常。
解决方案：将GLSurfaceView 添加到布局上这个操作放到Recorder初始化成功之后，这样就可以保证GLSurfaceView中的mGLThread能够在使用之前实例化。

为什么不提前设置render？如果提前设置render的话，在recorder中再次设置render的时候就会报render已经设置的异常。

总结：GLSurfaceView在添加到布局之前一定要先设置render，如果中间涉及到异步的处理，一定小心要保证这个顺序不能颠倒。

