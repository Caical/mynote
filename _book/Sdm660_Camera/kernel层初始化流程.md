## 一、Camera Kernel 驱动

*Kernel*驱动中高通把*Camera*系统分为*Camera*和*Sensor*两部分：

+ *Camera*部分是通用的代码逻辑，该部分由*msm_cam* 设备作为 *video*设备 与 *userspace* 进行交互，*Qcom*自已的*MIPI*，*ISP*，*CPP* 等硬件设备都属于*Camera*部分。

+ *Sensor* 可以理解为外部设备，是不同产商生产的*Camera sensor*模组。开发者分只需要配置不同的*Sensor*模组，将其注册到*msm_cam*设备上，创建好对应的*video* 设备，其他具体的接口逻辑均由*Camera*部分来实现。


在实际工作时，*camera video*设备主要是提供一个*v4l2*接口，*Camera* 驱动在接收到*event*消息后，会把该*event*消息及其参数以*Post* *event*形式发出到*hal* 层中，*hal*层接收到*camera* 驱动*post* 上来的*event*，来调用对应的逻辑，如果要操作*sensor* ，刚调用对应的 *video*设备就可以了。

## 二、工作流程

1. *Camera* 部分
   通过解析*compatible = "qcom,msm-cam"*;来初始化并注册好*media_device* 、*v4l2_device*、*video_device* 设备，同时生成*/dev/media0*节点。

2. *Sensor* 部分
   通过解析*compatible = "qcom,camera"*;来初始化调用probe解析*camera sensor*的*dts*节点信息，保存在全局*g_sctrl* 数组中。
   然后，上层在初始化时，依次对每个*sensor*下发 *ioctl* 参数，触发其作初始化*probe* ，上电*check_sensor_id* 及 创建对应的 /dev/*videoX* 节点 及 /dev/*mediaX* 的节点