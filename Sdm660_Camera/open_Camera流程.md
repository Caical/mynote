# Camera Open 总结

整个*Camera Open* 过程总结如下：

## 一、*Java APP* 层

+ 调用 *Frameworks* 层 *Camera.java* 中的 `open()` 方法；
+ 在*open camera* 后，才开始调用对应的 `getParameters()`，`setParameters()`， `startPreview()` 等 函数；

## 二、*Framework*层 *Camera.java* 中的 *open()*

+ 在`Camera open()` 函数中，如果没有指定*open* 对应的*camera id*，则默认打开后摄
+ 根据传入的 *camera id* 进行初始化
+ 注册 *EventHandler* 循环监听*Camera* 事件
+ 调用 `native_setup() JNI 函数`，并下发 *camera id* 、当前包名、*hal*层版本号

## 三、JNI 层 *android_hardware_Camera_native_setup*

+ 调用 `Camera::connect()` 函数，*connect* 为 *AIDL* 方法，通过*binder* 调用到服务端CameraService.cpp中的 `CameraService::connect()` 方法中
+ 获取 *Camera* *Class*，建立专门负责与*java* *Camera*对象通信的对象
+ 如果没有设置屏幕显示方向，则默认为0，如果不为0，调用`sendCommand` 下发屏幕显示方向

## 四、*C++* *Native* 层 *CameraService.cpp* 中

+ *CameraService.cpp* 是camera 初始化中讲过了，在开机时会注册好对应的 *cameraservice* 服务
+ 前面AIDL中，调用到 *CameraService.cpp*中的 `CameraService::connect()` 方法 在该方法中，
+ 首先检查是否存在 *client* ，如果存在的话直接返回
+ 关闭所有的 *flashlight* 对象
+ 调用`makeClient()` 创建 *Camera Client* 对象。 *client*是通过 `Camera2ClientBase()` 进行创建的，同时创建*Camera3Device* 对象
+ 调用`client->initialize`对前面创建好的 *Camera client* 进行初始化
+ 初始化前先对*device* 进行初始化 ，在`Camera3Device->initialize()`中打开对应的 *Camera* 节点*/dev/videox*，进行初始化，获取到 *Camera* 的操作方法
+ 初始化 *Camera* 默认参数
+ 接着对创建 *Camera* 运行时的六大线程，处理*preview*和录像 *stream*的线程、处理*3A*和人脸识别的线程、拍照流程状态机线程、处理拍照 *jpeg stream* 线程、处理零延时*zsl stream*使用的线程、处理callback的线程