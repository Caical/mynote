[TOC]

## 1. Camera Kernel 驱动

*Kernel*驱动中高通把*Camera*系统分为*Camera*和*Sensor*两部分：

+ *Camera*部分是通用的代码逻辑，该部分由*msm_cam* 设备作为 *video*设备 与 *userspace* 进行交互，*Qcom*自已的*MIPI*，*ISP*，*CPP* 等硬件设备都属于*Camera*部分。

+ Sensor 可以理解为外部设备，是不同产商生产的Camera sensor模组。开发者分只需要配置不同的Sensor模组，将其注册到msm_cam设备上，创建好对应的video 设备，其他具体的接口逻辑均由Camera部分来实现。


在实际工作时，*camera video*设备主要是提供一个*v4l2*接口，*Camera* 驱动在接收到*event*消息后，会把该*event*消息及其参数以*Post* *event*形式发出到*hal* 层中，*hal*层接收到*camera* 驱动*post* 上来的*event*，来调用对应的逻辑，如果要操作*sensor* ，刚调用对应的 *video*设备就可以了。


接下来，我们依次来看看 *msm_cam*、*sensor*、*v4l2* 这几部分的代码逻辑：

## 2. msm-cam 驱动

msm-cam是在dts 中定义的，在*\kernel\msm-4.4\arch\arm\boot\dts\qcom\sdm660-camera.dtsi中*：

```c
	qcom,msm-cam@ca00000 {
		compatible = "qcom,msm-cam";
		reg = <0xca00000 0x4000>;
		reg-names = "msm-cam";
		status = "ok";
		bus-vectors = "suspend", "svs", "nominal", "turbo";
		qcom,bus-votes = <0 150000000 320000000 320000000>;
		qcom,gpu-limit = <700000000>;
	};
```

查找代码，其注册的地方在*\kernel\msm-4.4\drivers\media\platform\msm\camera_v2\msm.c*。

可以看出，msm-cam 是以平台驱动的形式注册在kernel 中。

```c
static const struct of_device_id msm_dt_match[] = {
	{.compatible = "qcom,msm-cam"},
	{}
};
MODULE_DEVICE_TABLE(of, msm_dt_match);

static struct platform_driver msm_driver = {
	.probe = msm_probe,
	.driver = {
		.name = "msm",
		.owner = THIS_MODULE,
		.of_match_table = msm_dt_match,
	},
};

static int __init msm_init(void)
{
	return platform_driver_register(&msm_driver);
}
```

初始化注册成功后，会调用`msm_probe`函数，在该函数中，主要工作如下：

1. 初始化 *v4l2_device*、*video_device* 、*media_device* 结构体并分配好*KERNEL* 内存。

2. 注册 *media_device* 、*v4l2_device*、*video_device* 设备。
3. 创建好 *camera* *debug* *root* *fs*

```c
\kernel\msm-4.4\drivers\media\platform\msm\camera_v2\msm.c

static struct v4l2_device *msm_v4l2_dev;  // 初始化一个 v4l2_device 类型的结构体

static int msm_probe(struct platform_device *pdev)
{
	struct msm_video_device *pvdev = NULL;
	static struct dentry *cam_debugfs_root;

	// 1. 初始化一个 v4l2_device 类型的结构体，并分配好结构体内存
	msm_v4l2_dev = kzalloc(sizeof(*msm_v4l2_dev), GFP_KERNEL);
	pvdev = kzalloc(sizeof(struct msm_video_device), GFP_KERNEL);

	// 2. 分配 video_device 结构体内存
	pvdev->vdev = video_device_alloc(); 
	// ---> kzalloc(sizeof(struct video_device), GFP_KERNEL); 
	
	// 3. 分配 media_device 结构体内存
	msm_v4l2_dev->mdev = kzalloc(sizeof(struct media_device), GFP_KERNEL);

	strlcpy(msm_v4l2_dev->mdev->model, MSM_CONFIGURATION_NAME, sizeof(msm_v4l2_dev->mdev->model)); // msm_config
	msm_v4l2_dev->mdev->dev = &(pdev->dev);
	
	// 4. 注册 media_device , 使用的 v4l2 
	rc = media_device_register(msm_v4l2_dev->mdev);
	pvdev->vdev->entity.type = MEDIA_ENT_T_DEVNODE_V4L;		 // V4L
	pvdev->vdev->entity.group_id = QCAMERA_VNODE_GROUP_ID;   // #define QCAMERA_VNODE_GROUP_ID 2

	msm_v4l2_dev->notify = msm_sd_notify;  // 用于发现对应的 subdev
	pvdev->vdev->v4l2_dev = msm_v4l2_dev;
	
	// 5. 设置父设备为 pdev->dev （也就是 qcom,msm-cam 的设备信息）
	rc = v4l2_device_register(&(pdev->dev), pvdev->vdev->v4l2_dev);

	// 6. 注册 video_device设备 
	strlcpy(pvdev->vdev->name, "msm-config", sizeof(pvdev->vdev->name));
	pvdev->vdev->release  = video_device_release;
	pvdev->vdev->fops     = &msm_fops;			// 配置 video_device 的字符设备操作函数
	pvdev->vdev->ioctl_ops = &g_msm_ioctl_ops;	// 配置 v4l2 IOCTRL
	pvdev->vdev->minor     = -1;
	pvdev->vdev->vfl_type  = VFL_TYPE_GRABBER;
	rc = video_register_device(pvdev->vdev, VFL_TYPE_GRABBER, -1);

	// 7. 将当前 msm_video_device 结构体设为私有数据 
	video_set_drvdata(pvdev->vdev, pvdev);
	
	// 8. 分配  msm_queue_head 结构体内存
	msm_session_q = kzalloc(sizeof(*msm_session_q), GFP_KERNEL);
	msm_init_queue(msm_session_q);
	
	// 9. 创建 camera 调试目录
	cam_debugfs_root = debugfs_create_dir(MSM_CAM_LOGSYNC_FILE_BASEDIR, NULL);

	rc = cam_ahb_clk_init(pdev);

	of_property_read_u32(pdev->dev.of_node,
		"qcom,gpu-limit", &gpu_limit);

	goto probe_end;

}
```

### 2.1 struct v4l2_device 结构体描述

```c
\kernel\msm-4.4\include\media\v4l2-device.h

struct v4l2_device {
	/* dev->driver_data points to this struct.
	   Note: dev might be NULL if there is no parent device
	   as is the case with e.g. ISA devices. */
	struct device *dev;
#if defined(CONFIG_MEDIA_CONTROLLER)
	struct media_device *mdev;
#endif
	/* used to keep track of the registered subdevs */
	struct list_head subdevs;
	/* lock this struct; can be used by the driver as well if this
	   struct is embedded into a larger struct. */
	spinlock_t lock;
	/* unique device name, by default the driver name + bus ID */
	char name[V4L2_DEVICE_NAME_SIZE];
	/* notify callback called by some sub-devices. */
	void (*notify)(struct v4l2_subdev *sd,
			unsigned int notification, void *arg);
	/* The control handler. May be NULL. */
	struct v4l2_ctrl_handler *ctrl_handler;
	/* Device's priority state */
	struct v4l2_prio_state prio;
	/* Keep track of the references to this struct. */
	struct kref ref;
	/* Release function that is called when the ref count goes to 0. */
	void (*release)(struct v4l2_device *v4l2_dev);
};
```

### 2.2 struct msm_video_device 结构体描述

```c
\kernel\msm-4.4\drivers\media\platform\msm\camera_v2\msm.h

struct msm_video_device {
	struct video_device *vdev;
	atomic_t opened;
	struct mutex video_drvdata_mutex;
};
```

### 2.3 struct media_device 结构体描述

```c
struct media_device {
	/* dev->driver_data points to this struct. */
	struct device *dev;				// Parent device
	struct media_devnode devnode;	// Media device node

	char model[32];					// Device model name
	char serial[40];				// Device serial number (optional)
	char bus_info[32];				// Unique and stable device location identifier
	u32 hw_revision;				// Hardware device revision
	u32 driver_version;				// Device driver version

	u32 entity_id;					// ID of the next entity to be registered
	struct list_head entities;		// List of registered entities

	/* Protects the entities list */
	spinlock_t lock;				// Entities list lock
	/* Serializes graph operations. */
	struct mutex graph_mutex;		// Entities graph operation lock

	int (*link_notify)(struct media_link *link, u32 flags, unsigned int notification); // Link state change notification callback
};
```

## 3. Sensor 驱动

还是从 *dts* 开始看，前面我们配置*camera* *sensor*时，节点为 *compatible = "qcom,camera"*;对应代码在*\kernel\msm-4.4\drivers\media\platform\msm\camera_v2\sensor\msm_sensor_driver.c*。

将 *qcom*,*camera*注册在平台驱动中，并注册对应的*sensor* *i2c* 驱动。

```c
\kernel\msm-4.4\drivers\media\platform\msm\camera_v2\sensor\msm_sensor_driver.c

static const struct of_device_id msm_sensor_driver_dt_match[] = {
	{.compatible = "qcom,camera"},
	{}
};

MODULE_DEVICE_TABLE(of, msm_sensor_driver_dt_match);

static struct platform_driver msm_sensor_platform_driver = {
	.probe = msm_sensor_driver_platform_probe,
	.driver = {
		.name = "qcom,camera",
		.owner = THIS_MODULE,
		.of_match_table = msm_sensor_driver_dt_match,
	},
	.remove = msm_sensor_platform_remove,
};

static struct i2c_driver msm_sensor_driver_i2c = {
	.id_table = i2c_id,
	.probe  = msm_sensor_driver_i2c_probe,
	.remove = msm_sensor_driver_i2c_remove,
	.driver = {
		.name = SENSOR_DRIVER_I2C,
	},
};

static int __init msm_sensor_driver_init(void)
{
	rc = platform_driver_register(&msm_sensor_platform_driver);
	rc = i2c_add_driver(&msm_sensor_driver_i2c);
	return rc;
}
```

### 3.1 平台驱动probe函数 msm_sensor_driver_platform_probe()

在 `msm_sensor_driver_platform_probe`函数中，其主要工作如下:

1. 创建并分配 `msm_sensor_ctrl_t`结构体内存。
2. 将`sensor device type`初始化为 `MSM_CAMERA_PLATFORM_DEVICE`
3. 解析 节点为 `compatible = "qcom,camera";`的 dts 内容
4. 解析dts 中配置的camera clk 信息
   `clock-names = "cam_src_clk", "cam_clk";`
   `qcom,clock-rates = <24000000 0>;`

```c
\kernel\msm-4.4\drivers\media\platform\msm\camera_v2\sensor\msm_sensor_driver.c

static int32_t msm_sensor_driver_platform_probe(struct platform_device *pdev)
{
	int32_t rc = 0;
	struct msm_sensor_ctrl_t *s_ctrl = NULL;

	// 1. 创建并分配 msm_sensor_ctrl_t 结构体内存。
	/* Create sensor control structure */
	s_ctrl = kzalloc(sizeof(*s_ctrl), GFP_KERNEL);
	platform_set_drvdata(pdev, s_ctrl);

	// 2. 将sensor device type 初始化为 MSM_CAMERA_PLATFORM_DEVICE
	/* Initialize sensor device type */
	s_ctrl->sensor_device_type = MSM_CAMERA_PLATFORM_DEVICE;
	s_ctrl->of_node = pdev->dev.of_node;

	/*fill in platform device*/
	s_ctrl->pdev = pdev;
	
	// 3. 解析 节点为  compatible = "qcom,camera"; 的 dts 内容
	rc = msm_sensor_driver_parse(s_ctrl);
	==========================>
	|	static int32_t msm_sensor_driver_parse(struct msm_sensor_ctrl_t *s_ctrl)
	|	{
	|		/* Allocate memory for sensor_i2c_client */
	|		s_ctrl->sensor_i2c_client = kzalloc(sizeof(*s_ctrl->sensor_i2c_client), GFP_KERNEL);
	|	
	|		/* Parse dt information and store in sensor control structure */
	|		rc = msm_sensor_driver_get_dt_data(s_ctrl);  // 解析Camera DTS 节点，详见 Chapter 3.2
	|
	|		/* Initilize v4l2 subdev info */
	|		s_ctrl->sensor_v4l2_subdev_info = msm_sensor_driver_subdev_info;
	|		s_ctrl->sensor_v4l2_subdev_info_size = ARRAY_SIZE(msm_sensor_driver_subdev_info);
	|	
	|		/* Initialize default parameters */
	|		rc = msm_sensor_init_default_params(s_ctrl);  // 初始化默认参数
	|
	|		// 将 sensor ctrl 节构体保存在 g_sctrl 数组中。
	|		/* Store sensor control structure in static database */
	|		g_sctrl[s_ctrl->id] = s_ctrl;
	|		CDBG("g_sctrl[%d] %pK", s_ctrl->id, g_sctrl[s_ctrl->id]);
	|		return rc;
	|	}
	<==========================
	
	// 4. 解析dts 中配置的camera clk 信息
	// 解析 clock-names = "cam_src_clk", "cam_clk";
	// 解析 qcom,clock-rates = <24000000 0>;
	/* Get clocks information */
	rc = msm_camera_get_clk_info(s_ctrl->pdev, &s_ctrl->sensordata->power_info.clk_info,
		&s_ctrl->sensordata->power_info.clk_ptr, &s_ctrl->sensordata->power_info.clk_info_size);
	===========================>
	|	// @\kernel\msm-4.4\drivers\media\platform\msm\camera_v2\common\cam_soc_api.c
	|	rc = msm_camera_get_clk_info_internal(&pdev->dev, clk_info, clk_ptr, num_clk);
	<==========================

	/* Fill platform device id*/
	pdev->id = s_ctrl->id;

	/* Fill device in power info */
	s_ctrl->sensordata->power_info.dev = &pdev->dev;
	return rc;
}
```

### 3.2 解析Camera DTS 节点 msm_sensor_driver_get_dt_data()

在该函数中主要是对*dts* 中配置的*camera* 节点解析，工作如下:

1. 初始化 *msm_camera_sensor_board_info*结构体并分配对应的内存
2. 解析 *cell-index = <0>*，以此为*camera* 数组计数 *id*
3. 检测 *camera id* 是否大于系统最大支持数量
4. 判断 *msm_sensor_ctrl* 全局数组中当前*id* 是否已经初始化
5. 解析*Camera* 外设信息，包括 "*qcom,actuator-src"、"qcom,ois-src"、"qcom,eeprom-src"、"qcom,led-flash-src"、"qcom,csiphy-sd-index"*
6. 解析*Camera* 电压配置*，qcom,cam-vreg-name = "cam_vio", "cam_vana", "cam_vdig", "cam_vaf";*
7. 解析*Camera GPIO*配置解析 *"qcom,gpio-req-tbl-num"、"qcom,gpio-req-tbl-flags"、"qcom,gpio-req-tbl-label"*
   如果配置了使用*gpio* 供电的话，则在此初化始化，*"qcom,gpio-vana"、"qcom,gpio-vio"、"qcom,gpio-vaf"、"qcom,gpio-vdig"、"qcom,gpio-reset"、"qcom,gpio-standby"、"qcom,gpio-flash-en"*
8. 解析*I2C master* ，*"qcom,cci-master = <0>;"*
9. 解析摄像头旋转角度"*qcom,mount-angle"*，如果没有配置默认设置为0
10. 解析*sensor* 前后摄"*qcom,sensor-position"*
11. 解析*"qcom,sensor-mode"*

```c
\kernel\msm-4.4\drivers\media\platform\msm\camera_v2\sensor\msm_sensor_driver.c

static int32_t msm_sensor_driver_get_dt_data(struct msm_sensor_ctrl_t *s_ctrl)
{
	int32_t                              rc = 0, i = 0;
	struct msm_camera_sensor_board_info *sensordata = NULL;
	struct device_node                  *of_node = s_ctrl->of_node;
	uint32_t	cell_id;

	// 1. 初始化 msm_camera_sensor_board_info 结构体并分配对应的内存
	s_ctrl->sensordata = kzalloc(sizeof(*sensordata), GFP_KERNEL);
	sensordata = s_ctrl->sensordata;
	
	// 2. 解析 cell-index = <0>，以此为camera 数组计数 id
	/** Read cell index - this cell index will be the camera slot where this camera will be mounted */
	rc = of_property_read_u32(of_node, "cell-index", &cell_id);
	s_ctrl->id = cell_id;

	// 3. 检测camera id是否大于系统最大支持数量
	if (cell_id >= MAX_CAMERAS) {
		pr_err("failed: invalid cell_id %d", cell_id);
		rc = -EINVAL;
		goto FREE_SENSOR_DATA;
	}
	// 4. 判断全msm_sensor_ctrl 全局数组中当前id 是否已经初始化
	/* Check whether g_sctrl is already filled for this cell_id */
	if (g_sctrl[cell_id]) {
		pr_err("failed: sctrl already filled for cell_id %d", cell_id);
		rc = -EINVAL;
		goto FREE_SENSOR_DATA;
	}

	// 5. 解析Camera 外设信息，包括 "qcom,actuator-src"、"qcom,ois-src"、"qcom,eeprom-src"、
	// 			"qcom,led-flash-src"、"qcom,csiphy-sd-index"
	/* Read subdev info */
	rc = msm_sensor_get_sub_module_index(of_node, &sensordata->sensor_info);
	==================>
	+	//@\kernel\msm-4.4\drivers\media\platform\msm\camera_v2\sensor\io\msm_camera_dt_util.c
	+	int msm_sensor_get_sub_module_index(struct device_node *of_node, struct  msm_sensor_info_t **s_info)
	+	{
	+		struct msm_sensor_info_t *sensor_info;
	+		sensor_info = kzalloc(sizeof(*sensor_info), GFP_KERNEL);
	+		
	+		for (i = 0; i < SUB_MODULE_MAX; i++) {
	+			sensor_info->subdev_id[i] = -1; /* Subdev expose additional interface for same sub module*/
	+			sensor_info->subdev_intf[i] = -1;
	+		}
	+		src_node = of_parse_phandle(of_node, "qcom,actuator-src", 0);
	+		sensor_info->subdev_id[SUB_MODULE_ACTUATOR] = val;
	+		
	+		src_node = of_parse_phandle(of_node, "qcom,ois-src", 0);
	+		sensor_info->subdev_id[SUB_MODULE_OIS] = val;
	+		
	+		src_node = of_parse_phandle(of_node, "qcom,eeprom-src", 0);
	+		sensor_info->subdev_id[SUB_MODULE_EEPROM] = val;
	+	
	+		src_node = of_parse_phandle(of_node, "qcom,led-flash-src", 0);
	+		sensor_info->subdev_id[SUB_MODULE_LED_FLASH] = val;
	+		
	+		if (of_get_property(of_node, "qcom,csiphy-sd-index", &count)) {
	+			count /= sizeof(uint32_t);
	+			val_array = kzalloc(sizeof(uint32_t) * count, GFP_KERNEL);
	+			rc = of_property_read_u32_array(of_node, "qcom,csiphy-sd-index", val_array, count);
	+			for (i = 0; i < count; i++) {
	+				sensor_info->subdev_id[SUB_MODULE_CSIPHY + i] = val_array[i];
	+				CDBG("%s csiphy_core[%d] = %d\n", __func__, i, val_array[i]);
	+			}
	+		//.... 省略一部分解析代码
	<==================
	
	// 6. 解析Camera 电压配置， qcom,cam-vreg-name = "cam_vio", "cam_vana", "cam_vdig", "cam_vaf";
	/* Read vreg information */
	rc = msm_camera_get_dt_vreg_data(of_node, &sensordata->power_info.cam_vreg, &sensordata->power_info.num_vreg);
	==================>
	+	//@\kernel\msm-4.4\drivers\media\platform\msm\camera_v2\sensor\io\msm_camera_dt_util.c
	+	int msm_camera_get_dt_vreg_data(struct device_node *of_node, struct camera_vreg_t **cam_vreg, int *num_vreg)
	+	{
	+		count = of_property_count_strings(of_node, "qcom,cam-vreg-name");
	+		CDBG("%s qcom,cam-vreg-name count %d\n", __func__, count);
	+		vreg = kzalloc(sizeof(*vreg) * count, GFP_KERNEL);
	+		*cam_vreg = vreg;
	+		*num_vreg = count;
	+		for (i = 0; i < count; i++) {
	+			rc = of_property_read_string_index(of_node, "qcom,cam-vreg-name", i, &vreg[i].reg_name);
	+			CDBG("%s reg_name[%d] = %s\n", __func__, i,vreg[i].reg_name);
	+		}
	+		//.... 省略一部分解析代码
	+	}
	<==================

	// 7. 解析Camera GPIO配置
	/* Read gpio information */
	rc = msm_sensor_driver_get_gpio_data(&(sensordata->power_info.gpio_conf), of_node);
	==================>
	+	//@\kernel\msm-4.4\drivers\media\platform\msm\camera_v2\sensor\io\msm_camera_dt_util.c
	+	int32_t msm_sensor_driver_get_gpio_data(struct msm_camera_gpio_conf **gpio_conf, struct device_node *of_node)
	+	{
	+		uint16_t                    *gpio_array = NULL;
	+		struct msm_camera_gpio_conf *gconf = NULL;
	+		gpio_array_size = of_gpio_count(of_node);
	+		gconf = kzalloc(sizeof(struct msm_camera_gpio_conf),
	+		*gpio_conf = gconf;
	+	
	+		// 解析 "qcom,gpio-req-tbl-num"、"qcom,gpio-req-tbl-flags"、"qcom,gpio-req-tbl-label"
	+		rc = msm_camera_get_dt_gpio_req_tbl(of_node, gconf, gpio_array, gpio_array_size);
	+		============>
	+		+	of_get_property(of_node, "qcom,gpio-req-tbl-num", &count)
	+		+	rc = of_property_read_u32_array(of_node, "qcom,gpio-req-tbl-num", val_array, count);
	+		+	rc = of_property_read_u32_array(of_node, "qcom,gpio-req-tbl-flags", val_array, count);
	+		+	for (i = 0; i < count; i++) {
	+		+		rc = of_property_read_string_index(of_node,"qcom,gpio-req-tbl-label", i,&gconf->cam_gpio_req_tbl[i].label);
	+		+		CDBG("%s cam_gpio_req_tbl[%d].label = %s\n", __func__, i,gconf->cam_gpio_req_tbl[i].label);
	+		+	}
	+		<===========
	+		+	如果配置了使用gpio 供电的话，则在此初化始化， "qcom,gpio-vana"、"qcom,gpio-vio"、"qcom,gpio-vaf"、"qcom,gpio-vdig"、"qcom,gpio-reset"、"qcom,gpio-standby"、"qcom,gpio-flash-en"
	+		rc = msm_camera_init_gpio_pin_tbl(of_node, gconf, gpio_array,gpio_array_size);
	+		============>
	+			rc = of_property_read_u32(of_node, "qcom,gpio-vana", &val);
	+			gconf->gpio_num_info->gpio_num[SENSOR_GPIO_VANA] = gpio_array[val];
	+			rc = of_property_read_u32(of_node, "qcom,gpio-vio", &val);
	+			gconf->gpio_num_info->gpio_num[SENSOR_GPIO_VIO] = gpio_array[val];
	+			rc = of_property_read_u32(of_node, "qcom,gpio-vaf", &val);
	+			gconf->gpio_num_info->gpio_num[SENSOR_GPIO_VAF] = gpio_array[val];
	+			rc = of_property_read_u32(of_node, "qcom,gpio-vdig", &val);
	+			gconf->gpio_num_info->gpio_num[SENSOR_GPIO_VDIG] = gpio_array[val];
	+			rc = of_property_read_u32(of_node, "qcom,gpio-reset", &val);
	+			gconf->gpio_num_info->gpio_num[SENSOR_GPIO_RESET] = gpio_array[val];
	+			rc = of_property_read_u32(of_node, "qcom,gpio-standby", &val);
	+			gconf->gpio_num_info->gpio_num[SENSOR_GPIO_STANDBY] =gpio_array[val];
	+			rc = of_property_read_u32(of_node, "qcom,gpio-af-pwdm", &val);
	+			gconf->gpio_num_info->gpio_num[SENSOR_GPIO_AF_PWDM] = gpio_array[val];
	+			rc = of_property_read_u32(of_node, "qcom,gpio-flash-en", &val);
	+			gconf->gpio_num_info->gpio_num[SENSOR_GPIO_FL_EN] = gpio_array[val];
	+			//.... 省略一部分解析代码
	+		<===========
	+	}
	<==================
	
	// 8. 解析I2C master ，"qcom,cci-master = <0>;"
	/* Get CCI master */
	rc = of_property_read_u32(of_node, "qcom,cci-master", &s_ctrl->cci_i2c_master);
	CDBG("qcom,cci-master %d, rc %d", s_ctrl->cci_i2c_master, rc);

	// 9. 解析摄像头旋转角度"qcom,mount-angle"，如果没有配置默认设置为0
	/* Get mount angle */
	if (0 > of_property_read_u32(of_node, "qcom,mount-angle", &sensordata->sensor_info->sensor_mount_angle)) {
		/* Invalidate mount angle flag */
		sensordata->sensor_info->is_mount_angle_valid = 0;
		sensordata->sensor_info->sensor_mount_angle = 0;
	} else {
		sensordata->sensor_info->is_mount_angle_valid = 1;
	}
	CDBG("%s qcom,mount-angle %d\n", __func__,sensordata->sensor_info->sensor_mount_angle);

	// 10. 解析sensor 前后摄"qcom,sensor-position"
	if (0 > of_property_read_u32(of_node, "qcom,sensor-position",&sensordata->sensor_info->position)) {
		CDBG("%s:%d Invalid sensor position\n", __func__, __LINE__);
		sensordata->sensor_info->position = INVALID_CAMERA_B;
	}
	// 11. 解析sensor-mode
	if (0 > of_property_read_u32(of_node, "qcom,sensor-mode",
		&sensordata->sensor_info->modes_supported)) {
		CDBG("%s:%d Invalid sensor mode supported\n", __func__, __LINE__);
		sensordata->sensor_info->modes_supported = CAMERA_MODE_INVALID;
	}
	/* Get vdd-cx regulator */
	/*Optional property, don't return error if absent */
	of_property_read_string(of_node, "qcom,vdd-cx-name",&sensordata->misc_regulator);
	CDBG("qcom,misc_regulator %s", sensordata->misc_regulator);

	s_ctrl->set_mclk_23880000 = of_property_read_bool(of_node,"qcom,mclk-23880000");
	CDBG("%s qcom,mclk-23880000 = %d\n", __func__,s_ctrl->set_mclk_23880000);
	return rc;
}
```

#### 3.2.1 msm_camera_sensor_board_info 结构体描述

在*msm_camera_sensor_board_info* 中保存了所有*camera* 及硬件相关的*name*及参数。

```c
\kernel\msm-4.4\include\soc\qcom\camera2.h
struct msm_camera_sensor_board_info {
	const char *sensor_name;		// camera sensor name
	const char *eeprom_name;		// eeprom name
	const char *actuator_name;		// actuator name
	const char *ois_name;			// ois name
	const char *flash_name;			// flashlight name
	const char *special_support_sensors[MAX_SPECIAL_SUPPORT_SIZE];
	int32_t special_support_size;
	struct msm_camera_slave_info *slave_info;	// i2c addr
	struct msm_camera_csi_lane_params *csi_lane_params;
	struct msm_camera_sensor_strobe_flash_data *strobe_flash_data;	
	struct msm_actuator_info *actuator_info;
	struct msm_sensor_info_t *sensor_info;
	const char *misc_regulator;
	struct msm_camera_power_ctrl_t power_info;
	struct msm_camera_sensor_slave_info *cam_slave_info;
};
```

#### 3.2.2 Camera 支持的外设类型 sensor_sub_module_t

```c
\kernel\msm-4.4\include\uapi\media\msm_cam_sensor.h
enum sensor_sub_module_t {
	SUB_MODULE_SENSOR,
	SUB_MODULE_CHROMATIX,
	SUB_MODULE_ACTUATOR,
	SUB_MODULE_EEPROM,
	SUB_MODULE_LED_FLASH,
	SUB_MODULE_STROBE_FLASH,
	SUB_MODULE_CSID,
	SUB_MODULE_CSID_3D,
	SUB_MODULE_CSIPHY,
	SUB_MODULE_CSIPHY_3D,
	SUB_MODULE_OIS,
	SUB_MODULE_EXT,
	SUB_MODULE_IR_LED,
	SUB_MODULE_IR_CUT,
	SUB_MODULE_MAX,
};
```

### 3.3 【重点】全局CameraSensorCtrol数组g_sctrl

*g_sctrl* 是静态全局的一个*msm_sensor_ctrl_t* * 结构体指针数组，其定义如下：

```c
/* Static declaration */
static struct msm_sensor_ctrl_t *g_sctrl[MAX_CAMERAS];
```

前面我们在配置*dts* 时也发现了，通常我们要配置的*camera*数量是大于1的，前面代码中，我们配置了3个*Camera*，两个后摄，一个前摄。而这三个*camera* *dts* 中的节点都是一样的*"qcom,camera"。*
可以看出，驱动和设备会匹配三次，换句话说，也就是 `msm_sensor_driver_platform_probe()`函数会走三次，每次传递的*dts*节点内容是不一样的，三个*camera*都会依次*probe* 一次，
从而，当*probe* 完毕后，会保存三个`struct msm_sensor_ctrl_t *`结构体的数据保存在全局 *g_sctrl* 中。

至此，我们代码中，就把*dts* 的内容成功的转化为了`msm_sensor_ctrl_t`结构体保存在 全局 *g_sctrl* 中。

### 3.4 【重点】摄像头probe 函数 msm_sensor_driver_probe()

3.1 中我们分析过了 平台驱动probe函数 `msm_sensor_driver_platform_probe()`，这个函数主要作用还是解析*DTS*，但并不会真正*probe camera sensor*。那，*camera sensor probe* 是在什么时候呢？

其实，*camera probe* 并不是和其他*kernerl* 驱动一样，在初始化时就*probe*，而是通过*hal* 层下发 *probe* 指令来控制*probe* 的。

#### 3.4.1 hal层函数 module_sensor_init()

hal层代码位于 *\vendor\qcom\proprietary\mm-camera\mm-camera2\media-controller\modules\sensors\module\module_sensor.c*

```c
\vendor\qcom\proprietary\mm-camera\mm-camera2\media-controller\modules\sensors\module\module_sensor.c
mct_module_t *module_sensor_init(const char *name)
{
	......
	/* module_sensor_probe_sensors */
 	ret = sensor_init_probe(module_ctrl);
 	
	/* find all the actuator, etc with sensor */
  	ret = module_sensor_find_other_subdev(module_ctrl);
  	
	/* Init sensor modules */
  	ret = mct_list_traverse(module_ctrl->sensor_bundle, module_sensors_subinit,NULL);

	/* intiialize the eeprom */
  	ret = mct_list_traverse(module_ctrl->sensor_bundle, module_sensor_init_eeprom,module_ctrl->eebin_hdl);

	/* Create chromatix manager */
  	ret = mct_list_traverse(module_ctrl->sensor_bundle, module_sensor_init_chromatix, module_ctrl->eebin_hdl);

	/* Initialize dual cam stream mutex */
  	pthread_mutex_init(&module_ctrl->dual_cam_mutex, NULL);
 }
```

#### 3.4.2 hal层函数 sensor_init_probe()

上层代码逻辑我们后续会详细分析，并不是我们本章的重点，我们重点关注`sensor_init_probe()`，其内容如下:

在 `sensor_init_eebin_probe()`中，我们可以看出，知道*camera* 数量后，在*for*循环中，依次调用 `sensor_probe()`函数初始化每个*camera*，我们当前代码中有三个*camera*，这面就会调用三次`sensor_probe()`。
至于 *hal* 层中又是如何知道我们是三个*camera*的，后续我们分析到*hal* 层再说吧。

```c
/** sensor_init_probe: probe available sensors
 *
 *  @module_ctrl: sensor ctrl pointer
 *
 *  Return: 0 for success and negative error on failure
 *
 *  1) Find sensor_init subdev and it
 *  2) Open EEPROM subdev and check whether any sensor library
 *  is present in EEPROM
 *  3) Open sensor libraries present in dumped firware location
 *  4) Check library version of EEPROM and dumped firmware
 *  5) Load latest of both
 *  6) Pass slave information, power up and probe sensors
 *  7) If probe succeeds, create video node and sensor subdev
 *  8) Repeat step 2-8 for all sensor libraries present in
 *  EEPROM
 *  9) Repeat step 6-8 for all sensor libraries present in
 *  absolute path
 **/

boolean sensor_init_probe(module_sensor_ctrl_t *module_ctrl)
{
	......
 	ret = sensor_init_eebin_probe(module_ctrl, sd_fd);
	......
	RETURN_ON_FALSE(sensor_init_xml_probe(module_ctrl, sd_fd));
}

static boolean sensor_init_eebin_probe(module_sensor_ctrl_t *module_ctrl,int32_t sd_fd)
{
	SLOW("Enter");
	bin_ctl.cmd = EEPROM_BIN_GET_NUM_DEV;
  	bin_ctl.ctl.q_num.type = EEPROM_BIN_LIB_SENSOR;
  	bin_ctl.ctl.q_num.num_devs = 0;
  	eebin_interface_control(module_ctrl->eebin_hdl, &bin_ctl);

 	num_devs = bin_ctl.ctl.q_num.num_devs;

  	SLOW("num_devs:%d", num_devs);

  	for (i = 0; i < num_devs; i++ ) {
    	bin_ctl.cmd = EEPROM_BIN_GET_DEV_DATA;
    	bin_ctl.ctl.dev_data.type = EEPROM_BIN_LIB_SENSOR;
    	bin_ctl.ctl.dev_data.num = i;
    	rc = eebin_interface_control(module_ctrl->eebin_hdl, &bin_ctl);
    	if (rc < 0)
      		continue;
    	ret = sensor_probe(module_ctrl,
                       sd_fd,
                       bin_ctl.ctl.dev_data.name,
                       bin_ctl.ctl.dev_data.path,
                       NULL,
                       FALSE,
                       FALSE);

    	if (ret == FALSE) {
      		SINFO("failed: to load %s", bin_ctl.ctl.dev_data.name);
    	}
  }

  SLOW("Exit");
  return TRUE;
}

```

虽然前面也有 *sensor_probe*,但正常流程中，我们走的不是*eebin*,而是通过 `sensor_init_xml_probe(module_ctrl, sd_fd)` 来解析
*vendor/qcom/proprietary/mm-camera/mm-camera2/media-controller/modules/sensors/configs/sdm660_camera.xml* 文件，
通过 *xml* 中 配置的*sensor name* 和 *subdev name*，来下发参数。

详细如下：

1. 首先拼凑 sdm660_camera.xml 的字符串路径
2. odm 公司如果要自已定制路径的话，也可以通过 属性 persist.vendor.camera.customer.config 来配置
3. 开始解析 sdm660_camera.xml文件中的 CameraConfigurationRoot 节点
4. 通过 CameraModuleConfig 的数量可以知道 ，当前支持多少个camera
5. 解析每个camera 的信息，并且在 sensor_probe[]数组中检查当前sensor 是否已经probe 过了
6. 根据xml 中解析的结果，调用 sensor_probe 开始正式probe
7. 当所有xml 中的项都遍历完成后，关闭xml

```c
static boolean sensor_init_xml_probe(module_sensor_ctrl_t *module_ctrl, int32_t sd_fd)
{
	// 首先拼凑 sdm660_camera.xml 的字符串路径
  /* Create the xml path from data partition */
  snprintf(config_xml_name, BUFF_SIZE_255, "%s%s", CONFIG_XML_PATH, CONFIG_XML);
	// odm 公司如果要自已定制路径的话，也可以通过 属性 persist.vendor.camera.customer.config 来配置
  if (access(config_xml_name, R_OK)) {
    SHIGH(" read fail (non-fatal) %s. Trying from system partition",config_xml_name);

    if (csidtg_enable) {
      /* Create the CSIDTG xml path from system partition */
      snprintf(config_xml_name, BUFF_SIZE_255, "%s%s", CONFIG_XML_SYSTEM_PATH, CSIDTG_CONFIG_XML);
    } else {
      property_get("persist.vendor.camera.customer.config", custom_xml_name, CONFIG_XML);
      /* Create the xml path from system partition */
      snprintf(config_xml_name, BUFF_SIZE_255, "%s%s",
        CONFIG_XML_SYSTEM_PATH, custom_xml_name);
    }
  }

  SHIGH("reading from file %s", config_xml_name);

	// 开始解析 sdm660_camera.xml文件中的 CameraConfigurationRoot 节点
  /* Get the Root pointer and Document pointer of XMl file */
  ret = sensor_xml_util_load_file(config_xml_name, &docPtr, &rootPtr, "CameraConfigurationRoot");
	// 通过 CameraModuleConfig 的数量可以知道 ，当前支持多少个camera
  /* Get number of camera module configurations */
  num_cam_config = sensor_xml_util_get_num_nodes(rootPtr, "CameraModuleConfig");
  SLOW("num_cam_config = %d", num_cam_config);


  xmlConfig.docPtr = docPtr;
  xmlConfig.configPtr = &camera_cfg;

	// 解析每个camera 的信息，并且在 sensor_probe[]数组中检查当前sensor 是否已经probe 过了
  for (i = 0; i < num_cam_config; i++) {
    nodePtr = sensor_xml_util_get_node(rootPtr, "CameraModuleConfig", i);
    RETURN_ON_NULL(nodePtr);

    xmlConfig.nodePtr = nodePtr;
    ret = sensor_xml_util_get_camera_probe_config(&xmlConfig, "CameraModuleConfig");
    if (slot_probed[camera_cfg.camera_id]) {
      SHIGH("slot %d already probed", camera_cfg.camera_id);
      continue;
    }
	// 根据xml 中解析的结果，调用 sensor_probe 开始正式probe
    rc = sensor_probe(module_ctrl,
                      sd_fd,
                      camera_cfg.sensor_name,
                      NULL,
                      &xmlConfig,
                      FALSE,
                      FALSE);
    } else {
      slot_probed[camera_cfg.camera_id] = TRUE;
    }
  }
	// 关闭xml 文件
XML_PROBE_EXIT:
  sensor_xml_util_unload_file(docPtr);
  return ret;
}


```

#### 3.4.3 hal层函数 sensor_probe() 下发 CFG_SINIT_PROBE

进入 `sensor_probe()`函数：
在函数中可以看出，首先会调用 `sensor_load_library()`加载*vendor* 中*camera sensor*的库文件。
接着通过 *IOCTRL* 向通过下发 `CFG_SINIT_PROBE`消息，通知驱动层作*probe* 初始化。

```c
/** sensor_probe: probe available sensors
 *  @fd: sensor_init fd
 *  @sensor_name: sensor name
 *  Return: TRUE for success and FALSE for failure
 *  1) Open sensor library
 *  2) Pass slave information, probe sensor
 *  3) If probe succeeds, create video node and sensor subdev is
 *  created in kernel
 **/
static boolean sensor_probe(module_sensor_ctrl_t *module_ctrl, int32_t fd, const char *sensor_name, char *path, struct xmlCameraConfigInfo *xmlConfig, boolean is_stereo_config, boolean bypass_video_node_creation)
{
	/* Load sensor library */
	rc = sensor_load_library(sensor_name, sensor_lib_params, path);
	......
 	/* Pass slave information to kernel and probe */
  	memset(&cfg, 0, sizeof(cfg));
  	cfg.cfgtype = CFG_SINIT_PROBE;
  	cfg.cfg.setting = slave_info;
  	if (ioctl(fd, VIDIOC_MSM_SENSOR_INIT_CFG, &cfg) < 0) {
    	SINFO("[%s]CFG_SINIT_PROBE failed",sensor_name);
    	ret = FALSE;
    	goto ERROR;
  	}

  	if (cfg.probed_info.session_id == 0 && FALSE == bypass_video_node_creation) {
    	SINFO("[%s] probe failed.", sensor_name);
   	 	ret = FALSE;
    	goto ERROR;
  	}
  	SHIGH("[%s] probe succeeded: session_id(%d) entity_name(%s)",sensor_name, cfg.probed_info.session_id, cfg.entity_name);
	......
}

```

*IOCTRL* 命令定义如下：

```c
/* sensor init structures and enums */
enum msm_sensor_init_cfg_type_t {
	CFG_SINIT_PROBE,
	CFG_SINIT_PROBE_DONE,
	CFG_SINIT_PROBE_WAIT_DONE,
};
```

#### 3.4.4 Kernel Ioctl 函数 msm_sensor_init_subdev_ioctl()

上层*IOCTRL* 命令下发到*kernerl* 中，进入`msm_sensor_init_subdev_ioctl()`中，接着转发到`msm_sensor_driver_cmd()`中，调用 `msm_sensor_driver_probe()`函数

```c
\kernel\msm-4.4\drivers\media\platform\msm\camera_v2\sensor\msm_sensor_init.c
static long msm_sensor_init_subdev_ioctl(struct v4l2_subdev *sd, unsigned int cmd, void *arg)
{
	switch (cmd) {
	case VIDIOC_MSM_SENSOR_INIT_CFG:
		rc = msm_sensor_driver_cmd(s_init, arg);
		break;
	}
}

/* Static function definition */
static int32_t msm_sensor_driver_cmd(struct msm_sensor_init_t *s_init, void *arg)
{
	switch (cfg->cfgtype) {
	case CFG_SINIT_PROBE:
		mutex_lock(&s_init->imutex);
		s_init->module_init_status = 0;
		rc = msm_sensor_driver_probe(cfg->cfg.setting,
			&cfg->probed_info,
			cfg->entity_name);
		mutex_unlock(&s_init->imutex);
		if (rc < 0)
			pr_err("%s failed (non-fatal) rc %d", __func__, rc);
		break;
	case CFG_SINIT_PROBE_DONE:
		s_init->module_init_status = 1;
		wake_up(&s_init->state_wait);
		break;
	case CFG_SINIT_PROBE_WAIT_DONE:
		msm_sensor_wait_for_probe_done(s_init);
		break;
	return rc;
}

```

#### 3.4.5 probe函数 msm_sensor_driver_probe()

从上层开始下发*probe* 命令，至此正式开始*probe* 初始化 *camera*，代码如下：

1. 初始化并分配 *slave_info* 内存
2. 将上层下发的 *slave_info*保存在 *slave_info32* 中
3. 将 *slave_info32* 中的信息保存到 *slave_info*中。
4. 打印 *slave info* 信息
5. 通过*camera id* 获取到对应的 *camera sensor ctrol* 信息，也就是对应的*camera* 的*dts* 信息。
6. 检测*sensor* 是否已经*probe* 过了，如果不是，直接跳过if 进行probe
7. 获取*camera*的power settting
8. 初始化 *msm_camera_slave_info* 结构体变量 *camera_info* ，用于保存 *camera* 的信息
9. 配置*camera i2c* 相关信息
10. 往*s_ctrl* 中填充 上下电相关信息
11. 解析该*camera* 中所有外设 *dts* 节点信息 "*qcom,eeprom-src*"、"*qcom,actuator-src*"、"*qcom,led-flash-src*"
12. 调用 `sensor_power_up()`给*sensor* 上电，开始*probe sensor* ,上电时调用 `msm_sensor_check_id()`，然后调用`msm_sensor_match_id()`检测 *sensor id* 是否区配。
13. 创建对应的 */dev/videox* 节点 及 */dev/mediax* 的节点
14. *probe* 成功后下电
15. 更新*s_ctrl* 结构体信息

```c
kernel\msm-4.4\drivers\media\platform\msm\camera_v2\sensor\msm_sensor_driver.c
int32_t msm_sensor_driver_probe(void *setting, struct msm_sensor_info_t *probed_info, char *entity_name)
{
	struct msm_sensor_ctrl_t            *s_ctrl = NULL;
	struct msm_camera_cci_client        *cci_client = NULL;
	struct msm_camera_sensor_slave_info *slave_info = NULL;
	struct msm_camera_slave_info        *camera_info = NULL;

	// 1. 初始化并分配 slave_info 内存
	/* Allocate memory for slave info */
	slave_info = kzalloc(sizeof(*slave_info), GFP_KERNEL);

	if (is_compat_task()) {
		// 2. 将上层下发的 slave_info保存在 slave_info32 中
		struct msm_camera_sensor_slave_info32 *slave_info32 =kzalloc(sizeof(*slave_info32), GFP_KERNEL);
		copy_from_user((void *)slave_info32, setting, sizeof(*slave_info32));
		
		// 3. 将 slave_info32 中的信息保存到 slave_info中。
		strlcpy(slave_info->actuator_name, slave_info32->actuator_name, sizeof(slave_info->actuator_name));
		strlcpy(slave_info->eeprom_name, slave_info32->eeprom_name, sizeof(slave_info->eeprom_name));
		strlcpy(slave_info->sensor_name, slave_info32->sensor_name, sizeof(slave_info->sensor_name));
		strlcpy(slave_info->ois_name, slave_info32->ois_name, sizeof(slave_info->ois_name));
		strlcpy(slave_info->flash_name, slave_info32->flash_name, sizeof(slave_info->flash_name));

		slave_info->addr_type = slave_info32->addr_type;
		slave_info->camera_id = slave_info32->camera_id;

		slave_info->i2c_freq_mode = slave_info32->i2c_freq_mode;
		slave_info->sensor_id_info = slave_info32->sensor_id_info;

		slave_info->slave_addr = slave_info32->slave_addr;
		slave_info->module_id_info =  slave_info32->module_id_info;
		slave_info->power_setting_array.size = slave_info32->power_setting_array.size;
		slave_info->power_setting_array.size_down = slave_info32->power_setting_array.size_down;
		slave_info->power_setting_array.size_down = slave_info32->power_setting_array.size_down;
		slave_info->power_setting_array.power_setting = compat_ptr(slave_info32->power_setting_array.power_setting);
		slave_info->power_setting_array.power_down_setting = compat_ptr(slave_info32->power_setting_array.power_down_setting);
		slave_info->sensor_init_params = slave_info32->sensor_init_params;
		slave_info->output_format =slave_ info32->output_format;
		kfree(slave_info32);  // 保存完毕合释放 slave_info32 内存。
	} else
#endif
	{
		if (copy_from_user(slave_info,(void *)setting, sizeof(*slave_info))) {
			pr_err("failed: copy_from_user");
			rc = -EFAULT;
			goto free_slave_info;
		}
	}
	// 4. 打印 slave info 信息
	/* Print slave info */
	CDBG("camera id %d Slave addr 0x%X addr_type %d\n", slave_info->camera_id, slave_info->slave_addr, slave_info->addr_type);
	CDBG("sensor_id_reg_addr 0x%X sensor_id 0x%X sensor id mask %d", slave_info->sensor_id_info.sensor_id_reg_addr, slave_info->sensor_id_info.sensor_id,slave_info->sensor_id_info.sensor_id_mask);
	CDBG("power up size %d power down size %d\n",slave_info->power_setting_array.size,slave_info->power_setting_array.size_down);
	CDBG("position %d",slave_info->sensor_init_params.position);
	CDBG("mount %d",slave_info->sensor_init_params.sensor_mount_angle);

	// 5. 通过camera id 获取到对应的 camera sensor ctrol 信息，也就是对应的camera 的dts 信息。
	/* Extract s_ctrl from camera id */
	s_ctrl = g_sctrl[slave_info->camera_id];
	CDBG("s_ctrl[%d] %pK", slave_info->camera_id, s_ctrl);
	// 6. 检测sensor 是否已经probe 过了，如果不是，直接跳过if 进行probe
	if (s_ctrl->is_probe_succeed == 1) {
		/*
		 * Different sensor on this camera slot has been connected
		 * and probe already succeeded for that sensor. Ignore this
		 * probe */
		......
	}
	// 7. 获取camera的power settting
	rc = msm_sensor_get_power_settings(setting, slave_info,&s_ctrl->sensordata->power_info);
	
	// 8. 初始化 msm_camera_slave_info 结构体变量 camera_info ，用于保存 camera 的信息
	camera_info = kzalloc(sizeof(struct msm_camera_slave_info), GFP_KERNEL);
	
	s_ctrl->sensordata->slave_info = camera_info;
	/* Fill sensor slave info */
	camera_info->sensor_slave_addr = slave_info->slave_addr;
	camera_info->eeprom_slave_addr = slave_info->module_id_info.module_slave_id;
	camera_info->eeprom_module_reg_addr = slave_info->module_id_info.module_id_reg_addr;
	camera_info->eeprom_module_id = 	slave_info->module_id_info.module_id;
	camera_info->eeprom_master_id = slave_info->module_id_info.master_id;
	camera_info->sensor_id_reg_addr =slave_info->sensor_id_info.sensor_id_reg_addr;
	camera_info->sensor_id = slave_info->sensor_id_info.sensor_id;
	camera_info->sensor_id_mask = slave_info->sensor_id_info.sensor_id_mask;


	s_ctrl->sensor_i2c_client->addr_type = slave_info->addr_type;
	if (s_ctrl->sensor_i2c_client->client)
		s_ctrl->sensor_i2c_client->client->addr =camera_info->sensor_slave_addr;

	// 9. 配置camera i2c 相关信息。
	cci_client = s_ctrl->sensor_i2c_client->cci_client;

	cci_client->cci_i2c_master = s_ctrl->cci_i2c_master;
	cci_client->sid = slave_info->slave_addr >> 1;
	cci_client->retries = 3;
	cci_client->id_map = 0;
	cci_client->i2c_freq_mode = slave_info->i2c_freq_mode;

	// 10. 往s_ctrl 中填充 上下电相关信息 
	/* Parse and fill vreg params for powerup settings */
	rc = msm_camera_fill_vreg_params(
		s_ctrl->sensordata->power_info.cam_vreg,
		s_ctrl->sensordata->power_info.num_vreg,
		s_ctrl->sensordata->power_info.power_setting,
		s_ctrl->sensordata->power_info.power_setting_size);

	/* Parse and fill vreg params for powerdown settings*/
	rc = msm_camera_fill_vreg_params(
		s_ctrl->sensordata->power_info.cam_vreg,
		s_ctrl->sensordata->power_info.num_vreg,
		s_ctrl->sensordata->power_info.power_down_setting,
		s_ctrl->sensordata->power_info.power_down_setting_size);


CSID_TG:
	/* Update sensor, actuator and eeprom name in
	*  sensor control structure */
	s_ctrl->sensordata->sensor_name = slave_info->sensor_name;
	s_ctrl->sensordata->eeprom_name = slave_info->eeprom_name;
	s_ctrl->sensordata->actuator_name = slave_info->actuator_name;
	s_ctrl->sensordata->ois_name = slave_info->ois_name;
	s_ctrl->sensordata->flash_name = slave_info->flash_name;
	/*
	 * Update eeporm subdevice Id by input eeprom name
	 */
	 // 11. 解析该camera 中所有外设节点信息 "qcom,eeprom-src"、"qcom,actuator-src"、"qcom,led-flash-src"
	rc = msm_sensor_fill_eeprom_subdevid_by_name(s_ctrl);	
		=====> src_node = of_parse_phandle(of_node, "qcom,eeprom-src", i);
	/*
	 * Update actuator subdevice Id by input actuator name
	 */
	rc = msm_sensor_fill_actuator_subdevid_by_name(s_ctrl);
		=====> src_node = of_parse_phandle(of_node, "qcom,actuator-src", 0);
	rc = msm_sensor_fill_ois_subdevid_by_name(s_ctrl);
	rc = msm_sensor_fill_flash_subdevid_by_name(s_ctrl);
		=====> src_node = of_parse_phandle(of_node, "qcom,led-flash-src", 0);

	// 12. 调用 sensor_power_up() 给sensor 上电，开始probe sensor ,上电时调用 msm_sensor_check_id()，然后调用msm_sensor_match_id()检测sensor id 是否区配。
	/* Power up and probe sensor */
	rc = s_ctrl->func_tbl->sensor_power_up(s_ctrl);
	==================>
		@ \kernel\msm-4.4\drivers\media\platform\msm\camera_v2\sensor\msm_sensor.c
		for (retry = 0; retry < 3; retry++) {
			/* session is secure */
			s_ctrl->sensor_i2c_client->i2c_func_tbl =&msm_sensor_secure_func_tbl;
			
			rc = msm_camera_power_up(power_info, s_ctrl->sensor_device_type, sensor_i2c_client);
	
			rc = msm_sensor_check_id(s_ctrl);
				========> 
					rc = msm_sensor_match_id(s_ctrl);
				<=======
			if (rc < 0) {
				msm_camera_power_down(power_info,s_ctrl->sensor_device_type, sensor_i2c_client);
				msleep(20);
				continue;
			} else {
				break;
			}
		}
	<==================

	// 13. 创建对应的 /dev/videox 节点 及 /dev/mediax 的节点
	pr_err("%s probe succeeded", slave_info->sensor_name);
	/*
	 * Create /dev/videoX node, comment for now until dummy /dev/videoX
	 * node is created and used by HAL
	 */
	if (s_ctrl->sensor_device_type == MSM_CAMERA_PLATFORM_DEVICE)
		rc = msm_sensor_driver_create_v4l_subdev(s_ctrl);

	// 14. probe 成功后下电
	/* Power down */
	s_ctrl->func_tbl->sensor_power_down(s_ctrl);

	rc = msm_sensor_fill_slave_info_init_params(slave_info,s_ctrl->sensordata->sensor_info);

	rc = msm_sensor_validate_slave_info(s_ctrl->sensordata->sensor_info);

	/* Update sensor mount angle and position in media entity flag */
	is_yuv = (slave_info->output_format == MSM_SENSOR_YCBCR) ? 1 : 0;
	mount_pos = ((s_ctrl->is_secure & 0x1) << 26) | is_yuv << 25 |
		(s_ctrl->sensordata->sensor_info->position << 16) |
		((s_ctrl->sensordata->
		sensor_info->sensor_mount_angle / 90) << 8);

	s_ctrl->msm_sd.sd.entity.flags = mount_pos | MEDIA_ENT_FL_DEFAULT;

	/*Save sensor info*/
	s_ctrl->sensordata->cam_slave_info = slave_info;

	// 15. 更新s_ctrl 结构体信息
	msm_sensor_fill_sensor_info(s_ctrl, probed_info, entity_name);

	/*
	 * Set probe succeeded flag to 1 so that no other camera shall
	 * probed on this slot
	 */
	s_ctrl->is_probe_succeed = 1;
	return rc;
}


```

#### 3.4.6 创建 /dev/videoX节点 msm_sensor_driver_create_v4l_subdev()

```c
\kernel\msm-4.4\drivers\media\platform\msm\camera_v2\camera\camera.c
static int32_t msm_sensor_driver_create_v4l_subdev(struct msm_sensor_ctrl_t *s_ctrl)
{
	int32_t rc = 0;
	uint32_t session_id = 0;
	
	// 1. 初始化 msm_video_device 结构体，调用video_register_device() 注册video 节点
	rc = camera_init_v4l2(&s_ctrl->pdev->dev, &session_id);	
	===============>
		pvdev = kzalloc(sizeof(struct msm_video_device),GFP_KERNEL);
		pvdev->vdev = video_device_alloc();
		v4l2_dev = kzalloc(sizeof(struct v4l2_device), GFP_KERNEL);
		rc = v4l2_device_register(dev, pvdev->vdev->v4l2_dev);
		strlcpy(pvdev->vdev->name, "msm-sensor", sizeof(pvdev->vdev->name));
		pvdev->vdev->release  = video_device_release;
		pvdev->vdev->fops     = &camera_v4l2_fops;
		pvdev->vdev->ioctl_ops = &camera_v4l2_ioctl_ops;
		pvdev->vdev->minor     = -1;
		pvdev->vdev->vfl_type  = VFL_TYPE_GRABBER;
		rc = video_register_device(pvdev->vdev,VFL_TYPE_GRABBER, -1);
		*session = pvdev->vdev->num;
		video_set_drvdata(pvdev->vdev, pvdev);
	<===============

	CDBG("rc %d session_id %d", rc, session_id);
	s_ctrl->sensordata->sensor_info->session_id = session_id;

	/* Create /dev/v4l-subdevX device */
	v4l2_subdev_init(&s_ctrl->msm_sd.sd, s_ctrl->sensor_v4l2_subdev_ops);
	snprintf(s_ctrl->msm_sd.sd.name, sizeof(s_ctrl->msm_sd.sd.name), "%s",s_ctrl->sensordata->sensor_name);
	v4l2_set_subdevdata(&s_ctrl->msm_sd.sd, s_ctrl->pdev);
	s_ctrl->msm_sd.sd.flags |= V4L2_SUBDEV_FL_HAS_DEVNODE;
	media_entity_init(&s_ctrl->msm_sd.sd.entity, 0, NULL, 0);
	s_ctrl->msm_sd.sd.entity.type = MEDIA_ENT_T_V4L2_SUBDEV;
	s_ctrl->msm_sd.sd.entity.group_id = MSM_CAMERA_SUBDEV_SENSOR;
	s_ctrl->msm_sd.sd.entity.name = s_ctrl->msm_sd.sd.name;
	s_ctrl->msm_sd.close_seq = MSM_SD_CLOSE_2ND_CATEGORY | 0x3;
	rc = msm_sd_register(&s_ctrl->msm_sd);
	if (rc < 0) {
		pr_err("failed: msm_sd_register rc %d", rc);
		return rc;
	}
	msm_cam_copy_v4l2_subdev_fops(&msm_sensor_v4l2_subdev_fops);
#ifdef CONFIG_COMPAT
	msm_sensor_v4l2_subdev_fops.compat_ioctl32 =msm_sensor_subdev_fops_ioctl;
#endif
	s_ctrl->msm_sd.sd.devnode->fops =&msm_sensor_v4l2_subdev_fops;

	return rc;
}

```

### 3.5 struct msm_sensor_ctrl_t 结构体描述

```c
\kernel\msm-4.4\drivers\media\platform\msm\camera_v2\sensor\msm_sensor.h
struct msm_sensor_ctrl_t {
	struct platform_device *pdev;
	struct mutex *msm_sensor_mutex;

	enum msm_camera_device_type_t sensor_device_type;
	struct msm_camera_sensor_board_info *sensordata;
	struct msm_sensor_power_setting_array power_setting_array;
	struct msm_sensor_packed_cfg_t *cfg_override;
	struct msm_sd_subdev msm_sd;
	enum cci_i2c_master_t cci_i2c_master;

	struct msm_camera_i2c_client *sensor_i2c_client;
	struct v4l2_subdev_info *sensor_v4l2_subdev_info;
	uint8_t sensor_v4l2_subdev_info_size;
	struct v4l2_subdev_ops *sensor_v4l2_subdev_ops;
	struct msm_sensor_fn_t *func_tbl;
	struct msm_camera_i2c_reg_setting stop_setting;
	void *misc_regulator;
	enum msm_sensor_state_t sensor_state;
	uint8_t is_probe_succeed;
	uint32_t id;
	struct device_node *of_node;
	enum msm_camera_stream_type_t camera_stream_type;
	uint32_t set_mclk_23880000;
	uint8_t is_csid_tg_mode;
	uint32_t is_secure;
};

```

## 4. Camera 驱动总结

通过前面的分析，我们再次验证了*Kernel Camera*中的 *camera*及 *Sensor* 的两部份。

1. *Camera* 部分
   通过解析*compatible = "qcom,msm-cam"*;来初始化并注册好*media_device* 、*v4l2_device*、*video_device* 设备，同时生成*/dev/media0*节点。

2. *Sensor* 部分
   通过解析*compatible = "qcom,camera"*;来初始化调用probe解析*camera sensor*的*dts*节点信息，保存在全局*g_sctrl* 数组中。
   然后，上层在初始化时，依次对每个*sensor*下发 *ioctl* 参数，触发其作初始化*probe* ，上电*check_sensor_id* 及 创建对应的 /dev/*videoX* 节点 及 /dev/*mediaX* 的节点

至此，给合我们之前的移植过程，我们就将Kernel 中的 dts 相关的部分通过代码流程分析清楚了。
