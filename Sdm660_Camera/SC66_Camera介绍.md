## 高通Camera架构

### 设计原理：

+ Camera的所有功能被划分为不同的模块，让模块自己来决定自己的事情（高内聚，低耦合）；
+ 模块有统一的接口和格式；
+ 模块中有端口，通过端口把模块连接起来，又把模块挂在总线上；
+ 每一个端口的连接就是一个流，这些流通过pipeline来管理；
+ 每次启动一个camera就创建一个会话，由这个会话来管理此camera的一切事物。对于每一个会话，模块是共享的，它可以是camera的硬件资源也可以是其它资源（如一些软件算法等资源） ；

###  Camera模块结构：

1. 端口，端口分为src、sink、inter，如果这个模块只有source端口，那么它就是一个src模块；如果只有sink端口就是sink模块，如果都有就是中间模块（ISP模块即属于inter模块）。没有端口的模块是不能连接到流中的，但他可以完成一些其他的功能，比如接收引擎的设置，报告事件到bus等。
2. 模块线程，每个模块可以有一个线程来处理模块的事情。一个线程对应一个队列，线程就是从队列中取出数据处理，然后应答回去；
3. 总线回调，当一个模块向总线注册时，总线向其提供一个回调函数，当模块有事件发生时，调用这个函数向bus发消息，然后总线把这个消息提交给管道，管道把这个消息顺着流发下去；
4. 模块的get、set以及process函数，可以参看aecawb_thread_handler()函数awb 和aec 相关的处理。

### 代码结构：（下列目录未列出完整目录，仅讨论关键文件）

高通camera把sensor端底层设置、ISP效果参数、chomatix等进行了单独的剥离，放在**daemon**进程中进行。

daemon进程：

>daemon 进程又名守护进程，因为没有控制终端，所以说他们运行在后台。守护进程一般在系统启动时开始运行，在系统关闭时消亡。进程不因为用户或者其他变化而受到影响。

*camera deamon*代码放置在*vendor/qcom/proprietary/mm-camera*目录下，而此目录下的***mm-camera2***就是最新的camera架构位置。

可以看到*media-controller、server-imaging、server-tuning*及其它几个目录，重点主要在于***media-controller***目录。

```shell
media-controller
    |- mct	——包含了camera引擎、pipiline、bus、module、stream及event等定义及封装。
    |- modules	——进程需加载的六大模块
        |- sensors —— sensor 的驱动模块     							   —— src模块
        |- iface —— ISP interface模块      						   —— inter模块
        |- isp —— 主要是ISP的处理，内部包含众多的模块   				  —— inter模块
        |- stats —— 统计算法模块，如3A，ASD,AFD,IS,GRRO等数据统计的处理     —— sink模块
        |- pproc —— post process处理，如flip，rotate等     			—— inter模块
        |- imglib —— 主要是图片的一些后端处理，如HDR，人脸识别等   			 —— sink模块
```

以上各模块内部又包含了众多的模块，具体需要看代码分析。

*sensors*目录下：

```
sensors
	├─configs		不同型号的camera配置文件以及对应的chromatix配置文件
	├─sensor		sensor程序
	├─actuator		actuator程序
	├─flash			flash程序
	├─eeprom		eeprom程序
	└─chromatix		chromatix代码
```

*sensor、actuator、flash、eeprom*目录下的lib文件夹里包含不同型号硬件的驱动程序

*chromatix*下的*code*根据软件调试的结果导出，在进入目录前，我们会看到 0309 和 0310 这两个目录，其主要是根据平台来决定，具体的定位文件 在*/vendor/qcom/proprietary/mm-camera/Android.mk*下。

以*chromatix_imx230*为例（仅截取部分），*chromatix*中每个目录级对应一个库，最后都会映射到*configs*下的*chromatix*配置文件下。

```
├─3A
│  ├─4k_preview
│  ├─hdr_snapshot
│  ├─1080p_preview
│  ├─hfr_60
│  ├─video_16M
├─cpp
│  ├─cpp_hfr_60
│  ├─cpp_video_4k
│  ├─cpp_snapshot_hdr
│  ├─cpp_video
├─common
├─postproc
└─isp
    ├─snapshot
    ├─hfr_60
    ├─video_4k
    ├─preview
    └─hfr_120
```
