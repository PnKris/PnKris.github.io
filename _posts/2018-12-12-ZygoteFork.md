---
layout: post
title: Zygote Fork 过程
categories: [Android, framwork源码]
tags: [Zygote]
catalog: true
date: 2018-12-12
---
#### Zygote Fork 过程

这个流程算是一个承上启下的流程,难度不大,但是可以让我们了解到,从其他APP打开一个新的app的过程中,如何进行zygote的fork的,知道一个新的APP打开的入口在什么地方.

先来看一下调用过程的时序图:

[![fork Zygote进程.jpg](https://i.loli.net/2018/12/04/5c065290daf43.jpg)](https://i.loli.net/2018/12/04/5c065290daf43.jpg)

上图是从Process到调用ActivityThread.main()方法的流程，最后的run()方法就是调用ActivityThread.main()的

有几个遗留的知识点，我们在后面的分析中再看：
1. Process.start()是从什么地方调用的
2. ZygoteInit.main()方法是在什么地方调用

阅读此部分内容需要知道：
ZygoteInit.main()方法执行之后，会去调用zygoteServer.runSelectLoop()这个方法，然后在里面有个循环.

ZygoteInit.main()方法代码如下：

    public static void main(String argv[]) {
        ZygoteServer zygoteServer = new ZygoteServer();

        ...

        final Runnable caller;
        try {
            .....

            zygoteServer.registerServerSocket(socketName);
            .....

            // The select loop returns early in the child process after a fork and
            // loops forever in the zygote.
            caller = zygoteServer.runSelectLoop(abiList);
        } catch (Throwable ex) {
           ....
        } finally {
            ....
        }

        // We're in the child process and have exited the select loop. Proceed to execute the
        // command.
        if (caller != null) {
            caller.run();
        }
    }

创建一个ZygoteServer对象，然后调用zygoteServer.registerServerSocket(socketName);创建一个socket，然后调用zygoteServer.runSelectLoop()来进行处理socket请求信息，并返回一个Runnable caller,最后执行call.run()方法。


> 创建socket

- registerServerSocket()

> 监听socket

- runSelectLoop()
- while(true)循环里面监听socket
- 接收socket客户端发起的请求
- 然后调用processOneCommand()来处理请求
- 最后返回一个Runnable

看一下runSelectLoop()方法：

    Runnable runSelectLoop(String abiList) {
        ......

        while (true) {
            ......
            ......
                        final Runnable command = connection.processOneCommand(this);

                        if (mIsForkChild) {
                            ......

                            return command;
                        } else {
                            ......
                        }
                    } catch (Exception e) {
                        ......
                    }
                }
            }
        }
    }

上面就是调用processOneCommand()处理socket发过来的请求信息.

> 这里我们做个总结: 

ZygoteServer.main()在某个地方被调用了,具体哪个地方我们先不管,反正在我们用的时候这个东西已经被执行了.
然后,这个里面创建了一个socket,并且监听了,等待socket请求过来,然后处理请求,  这里先说明一下,这个请求就是客户端发起Fork Zygote进程请求,后面我们会看到.

接下来我们回到上面看那张时序图:

> 1. 发起请求

有个地方调用了Process.start()--->ZygoteProcess.start()---->ZygoteProcess.startViaZygote()---->zygoteSendArgsAndGetResult()发起forkZygote()请求
 也就是时序图的2,3,4,5,6,7部分

> 2. 响应请求

zygoteServer.runSelectLoop()中的socket收到这个请求了,调用processOneCommand()来处理请求. 参考时序图4部分

> 3. 处理请求 

- 读取请求参数
- 调用native方法fork出一个进程
- 调用handleChildProc()处理子进程
- 调用zygoteInit()初始化子进程
- 调用RuntimeInit.applicationInit(),利用反射查找ActivityThread,并调用main()方法,这些比较简单可以参考时序图看代码.
