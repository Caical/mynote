# 操作流程：

## 1. 内核驱动（主要是在设备树上添加传感器各部分的node）：

在设备树上添加对应node；

Path：*kernel/msm-3.18/Documentation/devicetree/bindings/*

>在arch目录下存在不同架构的平台，需要根据目标板平台进行选择（该板为arm64）

需要添加的包括

+ 传感器的GPIO；
+ 相关时钟 ；
+ 电源处理：

并在对应相机的设备树下也进行添加：

Path：*kernel/msm-3.18/arch/arm64/boot/dts/qcom/yourplatform-camera-sensor-mtp.dtsi*

## 2. 用户空间驱动

在***Sensor Driver***中参数主要包括：

+ ***camera ID, slaver address, sensor ID***
+ **输出信息格式（bayer/yuv，输出数据的格式）**

1. **寄存器信息**

> sensor的不同输出模式对应不同的寄存器设置，根据这个会有RES0_REG_ARRAY，RES1_REG_ARRAY等，一般对应不同的分辨率输出，对应的需要在res_settings_array，out_info_array等中添加多个res与之对应。主要包括init寄存器数组、启动/停止寄存器数组、解析寄存器数组。

2. **csi的lane数目**,必须在sensor最大范围之内，sensor寄存器的设置必须与lane数目相同.

+ **传感器电源开关上电顺序**

具体步骤：

1. 添加传感器驱动程序和*chromatix*代码
2. 添加内核驱动时添加对应电源节点，再按照开关顺序配置电源
3. 配置以下两个.xml文件：
   + *vendor/qcom/proprietary/mm-camera/mm-camera2/media-controller/modules/sensors/configs/yourplatform_camera.xml*
   + *vendor/qcom/proprietary/mm-camera/mm-camera2/media-controller/modules/sensors/configs/your_chromatix.xml*
4. 添加编译文件 *vendor/qcom/proprietary/common/config/device-vendor.mk*，在文件中输入要包含在构建中的新库。



+ **添加传感器驱动**

在上述sensor/lib里根据传感器型号添加对应传感器驱动，包括（以s5k3p3为例）

 *s5k3p3_lib.c*, *s5k3p3_lib.h*, *s5k3p3_pdaf_flip_mirror.h*, *s5k3p3_pdaf.h*, and *Android.mk*。

一般不需要修改传感器驱动，由相机制造商预先配置好参数。

+ **配置chromatix**

1. 在如下路径添加chromatix code 
   Path：*vendor/qcom/proprietary/mm-camera/mm-camera2/media-controller/modules/sensors/chromatix/0310/chromatix_yoursensor*

2. 在sensors目录的config文件夹里配置对应*chromatix.xml*与*camera.xml*文件。

>在chromatix.xml文件里ISP/CPP/3A需要根据上面设置的不同分辨率进行配置。（分辨率设置在s5k3p3_lib.h里)。
>
>camera.xml配置对应相机信息

3. 在sensors目录的*configs/Android.mk*里修改成对应xml文件。

以上传感器默认为Bayer的，如使用yuv传感器则修改传感器输出信息格式，且无需配置chromatix，其余步骤不变。

## 3. AF actuator驱动

+ **更新设备树**
+ **添加马达用户空间驱动**

1. 添加驱动在actuator目录的libs文件夹下

```
actuator
	├─module
	└─libs
    	├─bu64244gwz
    	├─lc898217xc
    	├─lc898212xd
    	├─dw9763b
```

2.添加AF算法调优文件

不需要单独chromatix文件。它包含在传感器3A文件中。

3.更新camera.xml文件

## 4. EEPRROM与LED_Flash驱动

+ **更新设备树**

+ 添加用户空间驱动