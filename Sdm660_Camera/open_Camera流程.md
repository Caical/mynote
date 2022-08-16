# Camera Open 流程

# 一、Java 层 Camera Class 使用介绍

先看下谷歌官方对 Camera Class 的介绍：

```java
The Camera class is used to set image capture settings, start/stop preview, 
					snap pictures, and retrieve frames for encoding for video. 
This Class is a client for the Camera service, which manages the actual camera hardware.

To access the device camera, 
you must declare the {@link android.Manifest.permission#CAMERA} permission in your Android Manifest. 

Also be sure to include the uses-feature manifest element to declare camera features used by your application.

For example, 
if you use the camera and auto-focus feature, your Manifest should include the following:

uses-permission android:name = “android.permission.CAMERA”
uses-feature android:name = “android.hardware.camera”
uses-feature android:name = "android.hardware.camera.autofocus"
```

## 1.1 拍照官方流程

1. 通过`open(int)`获取一个Camera实例。

2. 使用`getParameters()`取得现有的设置（例如默认分辨率、帧率等 ）。

3. 设置并下发 *Camera* 参数。如果有必要，修改返回的*Camera.Parameters*，并调用`setParameters(Camera.Parameters)。`

4. 设置屏幕显示方向，如果需要，调用`setDisplayOrientation(int)。`

5. 传递一个完全初始化的 *SurfaceHolder*，调用`setPreviewDisplay(SurfaceHolder)`，否则无法预览。

6. 调用 `startPreview()` 开始预览，拍照之前必须先预览。

7. 调用`takePicture()` 开始拍照时，等待它的回调函数回传数据。
   `takePicture(Camera.ShutterCallback,Camera.PictureCallback, Camera.PictureCallback, Camera.PictureCallback)`

8. 拍照完后，preview会停止，如果想继续拍照，需重新调用 startPreview()。

9. 调用stopPreview()停止更新预览界面。

10. 调用 release() 释放Camera 资源。

## 1.2 录像官方流程

1. 获取并初始化 Camera 对像，并调用 startPreview()。
2. 调用 unlock() 允许 media 进程操作Camera。
3. 调用 setCamera(Camera) 将 Camera 对像传递给 MediaRecorder。
4. 当录像完毕后，调用 reconnect() 重新锁定Camera。
5. 如果想拍照，重新preview ，然后拍照。
6. 调用 stopPreview() 和 release() 退出预览并释放Camera资源。
   

# 二、 Frameworks 层 Camera.java 分析

Java 层中调用的 Camera Class 源码位于 `frameworks/base/core/java/android/hardware/Camera.java`

```JAVA
frameworks/base/core/java/android/hardware/Camera.java

public class Camera {
    private static final String TAG = "Camera";
    
    private CameraDataCallback mCameraDataCallback;
    private CameraMetaDataCallback mCameraMetaDataCallback;
    
	public static final int CAMERA_HAL_API_VERSION_1_0 = 0x100;

	public static class CameraInfo {
        public static final int CAMERA_FACING_BACK = 0;
        public static final int CAMERA_FACING_FRONT = 1;
        public static final int CAMERA_SUPPORT_MODE_ZSL = 2;
        public static final int CAMERA_SUPPORT_MODE_NONZSL = 3;
        public int facing;
        public int orientation;
        public boolean canDisableShutterSound;
    };
	
	// 1. 找开指定 id 的Camera
	public static Camera open(int cameraId) {
        return new Camera(cameraId);
    }
    
	// 2. 在Camera open() 函数中，如果没有指定open 对应的camera id，则默认打开后摄
	public static Camera open() {
        int numberOfCameras = getNumberOfCameras();
        CameraInfo cameraInfo = new CameraInfo();
        for (int i = 0; i < numberOfCameras; i++) {
            getCameraInfo(i, cameraInfo);
            if (cameraInfo.facing == CameraInfo.CAMERA_FACING_BACK) {
                return new Camera(i);
            }
        }
        return null;
    }
    
	// 3. 调用 cameraInitNormal() 初始对应 id 的Camera
	/** used by Camera#open, Camera#open(int) */
    Camera(int cameraId) {
        int err = cameraInitNormal(cameraId);
    }
    
	private int cameraInitNormal(int cameraId) {
        return cameraInitVersion(cameraId, CAMERA_HAL_API_VERSION_NORMAL_CONNECT);  // -2
    }

	private int cameraInitVersion(int cameraId, int halVersion) {
        mShutterCallback = null;	
        mRawImageCallback = null;
        mJpegCallback = null;
        mPreviewCallback = null;
        mPostviewCallback = null;
        mUsingPreviewAllocation = false;
        mZoomListener = null;
        /* ### QC ADD-ONS: START */
        mCameraDataCallback = null;
        mCameraMetaDataCallback = null;
        /* ### QC ADD-ONS: END */

		// 1. 注册 EventHandler 循环监听Camera 事件
        Looper looper;
        if ((looper = Looper.myLooper()) != null) {
            mEventHandler = new EventHandler(this, looper);
        } else if ((looper = Looper.getMainLooper()) != null) {
            mEventHandler = new EventHandler(this, looper);
        } else {
            mEventHandler = null;
        }
		// 2. 获取当前包名
        String packageName = ActivityThread.currentOpPackageName();

		// 3. 如果定义了 prop 属性 "camera.hal1.packagelist" ， 则强制使用 HAL1
        //Force HAL1 if the package name falls in this bucket
        String packageList = SystemProperties.get("camera.hal1.packagelist", "");
        if (packageList.length() > 0) {
            TextUtils.StringSplitter splitter = new TextUtils.SimpleStringSplitter(',');
            splitter.setString(packageList);
            for (String str : splitter) {
                if (packageName.equals(str)) {
                    halVersion = CAMERA_HAL_API_VERSION_1_0;
                    break;
                }
            }
        }
        // 4. 调用 native_setup() JNI 函数，并下发 camera id 、当前包名、hal层版本号
        return native_setup(new WeakReference<Camera>(this), cameraId, halVersion, packageName);
    }
```

## 2.1 【JNI】 CameraService初始化 native_setup( ) —> android_hardware_Camera_native_setup( )

java 中通过JNI 调用 native 方法，Camera JNI 代码位于

```c++
//cameraId = 0
//halVersion = -2
native_setup(new WeakReference<Camera>(this), cameraId, halVersion, packageName);

@ frameworks/base/core/jni/android_hardware_Camera.cpp
static const JNINativeMethod camMethods[] = {
  { "_getNumberOfCameras", 	"()I", 		(void *)android_hardware_Camera_getNumberOfCameras },
  { "_getCameraInfo", 		"(ILandroid/hardware/Camera$CameraInfo;)V", (void*)android_hardware_Camera_getCameraInfo },
  { "native_setup", 		"(Ljava/lang/Object;IILjava/lang/String;)I", (void*)android_hardware_Camera_native_setup },
  { "native_release", 		"()V", 		(void*)android_hardware_Camera_release },
  { "setPreviewSurface", 	"(Landroid/view/Surface;)V",	 (void *)android_hardware_Camera_setPreviewSurface },
  { "setPreviewTexture", 	"(Landroid/graphics/SurfaceTexture;)V",	(void *)android_hardware_Camera_setPreviewTexture },
  { "setPreviewCallbackSurface", "(Landroid/view/Surface;)V", (void *)android_hardware_Camera_setPreviewCallbackSurface },
  { "startPreview",			"()V",	 	(void *)android_hardware_Camera_startPreview },
  { "_stopPreview",			"()V",		(void *)android_hardware_Camera_stopPreview },
  { "previewEnabled", 		"()Z", 		(void *)android_hardware_Camera_previewEnabled },
  { "setHasPreviewCallback","(ZZ)V",	(void *)android_hardware_Camera_setHasPreviewCallback },
  { "_addCallbackBuffer",	"([BI)V",	(void *)android_hardware_Camera_addCallbackBuffer },
  { "native_autoFocus",	 	"()V",		(void *)android_hardware_Camera_autoFocus },
  { "native_cancelAutoFocus","()V",		(void *)android_hardware_Camera_cancelAutoFocus },
  { "native_takePicture",	"(I)V",		(void *)android_hardware_Camera_takePicture },
  { "native_setHistogramMode","(Z)V",	(void *)android_hardware_Camera_setHistogramMode },
  { "native_setMetadataCb",	"(Z)V",	 	(void *)android_hardware_Camera_setMetadataCb },
  { "native_sendHistogramData","()V",	(void *)android_hardware_Camera_sendHistogramData },
 { "native_setLongshot",	"(Z)V",		(void *)android_hardware_Camera_setLongshot },
  { "native_setParameters",	"(Ljava/lang/String;)V",	(void *)android_hardware_Camera_setParameters },
  { "native_getParameters", "()Ljava/lang/String;",		(void *)android_hardware_Camera_getParameters },
  { "reconnect", 			"()V",		(void*)android_hardware_Camera_reconnect },
  { "lock",					"()V", 		(void*)android_hardware_Camera_lock },
  { "unlock",				"()V",		(void*)android_hardware_Camera_unlock },
  { "startSmoothZoom",		"(I)V",		(void *)android_hardware_Camera_startSmoothZoom },
  { "stopSmoothZoom",		"()V",		(void *)android_hardware_Camera_stopSmoothZoom },
  { "setDisplayOrientation","(I)V",		(void *)android_hardware_Camera_setDisplayOrientation },
  { "_enableShutterSound",	"(Z)Z",		(void *)android_hardware_Camera_enableShutterSound },
  { "_startFaceDetection",	"(I)V",		(void *)android_hardware_Camera_startFaceDetection },
  { "_stopFaceDetection",	"()V",		(void *)android_hardware_Camera_stopFaceDetection},
  { "enableFocusMoveCallback","(I)V",	(void *)android_hardware_Camera_enableFocusMoveCallback},
};
```

```c++
// connect to camera service
static jint android_hardware_Camera_native_setup(JNIEnv *env, jobject thiz, jobject weak_this, jint cameraId, jint halVersion, jstring clientPackageName)
{
    // Convert jstring to String16
    const char16_t *rawClientName = reinterpret_cast<const char16_t*>(env->GetStringChars(clientPackageName, NULL));
    jsize rawClientNameLen = env->GetStringLength(clientPackageName);
    String16 clientName(rawClientName, rawClientNameLen);
    env->ReleaseStringChars(clientPackageName, reinterpret_cast<const jchar*>(rawClientName));

    sp<Camera> camera;
    if (halVersion == CAMERA_HAL_API_VERSION_NORMAL_CONNECT) {		// -2
        // Default path: hal version is don't care, do normal camera connect.
        // 调用 Camera::connect() 函数，connect 为 AIDL 方法，
        // 通过binder 调用到服务端CameraService.cpp中的 CameraService::connect() 方法中
        camera = Camera::connect(cameraId, clientName, Camera::USE_CALLING_UID, Camera::USE_CALLING_PID);
        ======================> 
        +	// frameworks/av/camera/Camera.cpp
        +	return CameraBaseT::connect(cameraId, clientPackageName, clientUid, clientPid);
        +		========> 
        +		// frameworks/av/camera/CameraBase.cpp
        +		sp<TCam> CameraBase<TCam, TCamTraits>::connect(int cameraId, const String16& clientPackageName, int clientUid, int clientPid)
        +		{
        +		-	sp<TCam> c = new TCam(cameraId);
        +		-	sp<TCamCallbacks> cl = c;
        +			const sp<::android::hardware::ICameraService> cs = getCameraService();
        +			TCamConnectService fnConnectService = TCamTraits::fnConnectService;
        +			ret = (cs.get()->*fnConnectService)(cl, cameraId, clientPackageName, clientUid, clientPid, /*out*/ &c->mCamera);
        +			--------->
        +				CameraTraits<Camera>::TCamConnectService CameraTraits<Camera>::fnConnectService = 
        +								&::android::hardware::ICameraService::connect;
        +				
        +			IInterface::asBinder(c->mCamera)->linkToDeath(c);
        +			return c;
        +		}
        +		<========
        <======================
    } else {
        jint status = Camera::connectLegacy(cameraId, halVersion, clientName, Camera::USE_CALLING_UID, camera);
        if (status != NO_ERROR) {
            return status;
        }
    }
    // make sure camera hardware is alive
    if (camera->getStatus() != NO_ERROR) {
        return NO_INIT;
    }
	// 获取 Camera Class
    jclass clazz = env->GetObjectClass(thiz);
    if (clazz == NULL) {
        // This should never happen
        jniThrowRuntimeException(env, "Can't find android/hardware/Camera");
        return INVALID_OPERATION;
    }

    // We use a weak reference so the Camera object can be garbage collected.
    // The reference is only used as a proxy for callbacks.
    sp<JNICameraContext> context = new JNICameraContext(env, weak_this, clazz, camera);
    context->incStrong((void*)android_hardware_Camera_native_setup);
    camera->setListener(context);	// 专门负责与java Camera对象通信的对象 

    // save context in opaque field
    env->SetLongField(thiz, fields.context, (jlong)context.get());

    // Update default display orientation in case the sensor is reverse-landscape
    CameraInfo cameraInfo;
    status_t rc = Camera::getCameraInfo(cameraId, &cameraInfo);

    int defaultOrientation = 0;
    switch (cameraInfo.orientation) {
        case 0: break;
        case 90:
            if (cameraInfo.facing == CAMERA_FACING_FRONT) {
                defaultOrientation = 180;
            }
            break;
        case 180:
            defaultOrientation = 180;
            break;
        case 270:
            if (cameraInfo.facing != CAMERA_FACING_FRONT) {
                defaultOrientation = 180;
            }
            break;
        default:
            ALOGE("Unexpected camera orientation %d!", cameraInfo.orientation);
            break;
    }
    // 如果没有设置屏幕显示方向，则默认为0，如果不为0，调用sendCommand 下发屏幕显示方向
    if (defaultOrientation != 0) {
        ALOGV("Setting default display orientation to %d", defaultOrientation);
        rc = camera->sendCommand(CAMERA_CMD_SET_DISPLAY_ORIENTATION,defaultOrientation, 0);
    }
    return NO_ERROR;
}
```

## 2.2 【AIDL】接口ICameraService

在前面代码中， 经过如下一系列调用，最终走到了 AIDL 层。

*native_setup()   →   android_hardware_Camera_native_setup()   →   CameraBaseT::connect   →   sp<TCam> CameraBase<TCam, TCamTraits>::connect   →   ret = (cs.get()->*fnConnectService)   →  android::hardware::ICameraService::connect()
***fnConnectService*** 定义如下：

*CameraTraits<Camera>::TCamConnectService  CameraTraits<Camera>::fnConnectService = &::android::hardware::ICameraService::connect;*

```java
frameworks/av/camera/aidl/android/hardware/ICameraService.aidl

/**
 * Binder interface for the native camera service running in mediaserver.
 * @hide
 */
interface ICameraService
{
	 /** Open a camera device through the old camera API */
    ICamera connect(ICameraClient client, int cameraId, String opPackageName, int clientUid, int clientPid);

	/** Open a camera device through the new camera API Only supported for device HAL versions >= 3.2 */
    ICameraDeviceUser connectDevice(ICameraDeviceCallbacks callbacks, int cameraId,String opPackageName,int clientUid);
}
```

AIDL 中，我们走的是老方法， `ICamera connect().`

> Android Interface Definition Language,是一种接口定义语言，用于生成可以在Android设备上两个进程间进行通信的代码。
> Android Java Service Framework提供的大多数系统服务都是使用AIDL语言生成的。使用AIDL语言，可以自动生成服务接口、服务代理、服务Stub代码。

## 2.3 【原生】CameraService.cpp

通过 AIDL 的 Binder 通信，跳转到 CameraService.cpp 中。

CameraService.cpp 是camera 初始化中，在开机时会注册好对应的 cameraservice 服务。
此时就是通过Binder通信，调用CameraService 服务中的 connect 方法：

```c++
frameworks/av/services/camera/libcameraservice/CameraService.cpp 
Status CameraService::connect(
        const sp<ICameraClient>& cameraClient,
        int cameraId,
        const String16& clientPackageName,
        int clientUid,
        int clientPid,
        /*out*/
        sp<ICamera>* device) {

    String8 id = String8::format("%d", cameraId);
    sp<Client> client = nullptr;
    ret = connectHelper<ICameraClient,Client>(cameraClient, id,
            CAMERA_HAL_API_VERSION_UNSPECIFIED, clientPackageName, clientUid, clientPid, API_1,
            /*legacyMode*/ false, /*shimUpdateOnly*/ false,
            /*out*/client);

    *device = client;
    return ret;
}
```

进入 connectHelper 函数看下：

1. 检查 Client 的权限
2. 如果已存在 client 则直接返回
3. 在打开Camera 前，所有Flashlight 都应该被关闭
4. 调用makeClient() 创建 Camera Client 对象。 client是通过 Camera2ClientBase() 进行创建的，同时创建Camera3Device 对象
5. 调用client->initialize对前面创建好的 Camera client 进行初始化

```c++
template<class CALLBACK, class CLIENT>
binder::Status CameraService::connectHelper(const sp<CALLBACK>& cameraCb, const String8& cameraId,
        int halVersion, const String16& clientPackageName, int clientUid, int clientPid,
        apiLevel effectiveApiLevel, bool legacyMode, bool shimUpdateOnly,
        /*out*/sp<CLIENT>& device) {
    binder::Status ret = binder::Status::ok();

    String8 clientName8(clientPackageName);

    int originalClientPid = 0;

    ALOGI("CameraService::connect call (PID %d \"%s\", camera ID %s) for HAL version %s and "
            "Camera API version %d", clientPid, clientName8.string(), cameraId.string(),
            (halVersion == -1) ? "default" : std::to_string(halVersion).c_str(), static_cast<int>(effectiveApiLevel));

    sp<CLIENT> client = nullptr;
    {
    	// 1. 检查 Client 的权限
        // Enforce client permissions and do basic sanity checks
        if(!(ret = validateConnectLocked(cameraId, clientName8,  /*inout*/clientUid, /*inout*/clientPid, /*out*/originalClientPid)).isOk()) {
            return ret;
        }

        sp<BasicClient> clientTmp = nullptr;
        std::shared_ptr<resource_policy::ClientDescriptor<String8, sp<BasicClient>>> partial;
        err = handleEvictionsLocked(cameraId, originalClientPid, effectiveApiLevel,
                IInterface::asBinder(cameraCb), clientName8, /*out*/&clientTmp, /*out*/&partial));
		// 2. 如果已存在 client 则直接返回
        if (clientTmp.get() != nullptr) {
            // Handle special case for API1 MediaRecorder where the existing client is returned
            device = static_cast<CLIENT*>(clientTmp.get());
            return ret;
        }
		// 3. 在打开Camera 前，所有Flashlight 都应该被关闭
        // give flashlight a chance to close devices if necessary.
        mFlashlight->prepareDeviceOpen(cameraId);

        // TODO: Update getDeviceVersion + HAL interface to use strings for Camera IDs
        int id = cameraIdToInt(cameraId);

        int facing = -1;
        int deviceVersion = getDeviceVersion(id, /*out*/&facing);
        sp<BasicClient> tmp = nullptr;
		
		// 4.  调用makeClient() 创建 Camera Client 对象。 client是通过 Camera2ClientBase() 进行创建的，同时创建Camera3Device 对象
        if(!(ret = makeClient(this, cameraCb, clientPackageName, id, facing, clientPid,
                clientUid, getpid(), legacyMode, halVersion, deviceVersion, effectiveApiLevel,
                /*out*/&tmp)).isOk()) {
            return ret;
        }
        client = static_cast<CLIENT*>(tmp.get());  // Camera2Client

        LOG_ALWAYS_FATAL_IF(client.get() == nullptr, "%s: CameraService in invalid state",
                __FUNCTION__);
		// 5. 调用client->initialize对前面创建好的 Camera client 进行初始化
        if ((err = client->initialize(mModule)) != OK) {
            ALOGE("%s: Could not initialize client from HAL module.", __FUNCTION__);
            // Errors could be from the HAL module open call or from AppOpsManager
            switch(err) {
                case BAD_VALUE:
                    return STATUS_ERROR_FMT(ERROR_ILLEGAL_ARGUMENT,
                            "Illegal argument to HAL module for camera \"%s\"", cameraId.string());
                case -EBUSY:
                    return STATUS_ERROR_FMT(ERROR_CAMERA_IN_USE,
                            "Camera \"%s\" is already open", cameraId.string());
                case -EUSERS:
                    return STATUS_ERROR_FMT(ERROR_MAX_CAMERAS_IN_USE,
                            "Too many cameras already open, cannot open camera \"%s\"",
                            cameraId.string());
                case PERMISSION_DENIED:
                    return STATUS_ERROR_FMT(ERROR_PERMISSION_DENIED,
                            "No permission to open camera \"%s\"", cameraId.string());
                case -EACCES:
                    return STATUS_ERROR_FMT(ERROR_DISABLED,
                            "Camera \"%s\" disabled by policy", cameraId.string());
                case -ENODEV:
                default:
                    return STATUS_ERROR_FMT(ERROR_INVALID_OPERATION,
                            "Failed to initialize camera \"%s\": %s (%d)", cameraId.string(),
                            strerror(-err), err);
            }
        }

        // Update shim paremeters for legacy clients
        if (effectiveApiLevel == API_1) {
            // Assume we have always received a Client subclass for API1
            sp<Client> shimClient = reinterpret_cast<Client*>(client.get());
            String8 rawParams = shimClient->getParameters();
            CameraParameters params(rawParams);

            auto cameraState = getCameraState(cameraId);
            if (cameraState != nullptr) {
                cameraState->setShimParams(params);
            } else {
                ALOGE("%s: Cannot update shim parameters for camera %s, no such device exists.",
                        __FUNCTION__, cameraId.string());
            }
        }

        if (shimUpdateOnly) {
            // If only updating legacy shim parameters, immediately disconnect client
            mServiceLock.unlock();
            client->disconnect();
            mServiceLock.lock();
        } else {
            // Otherwise, add client to active clients list
            finishConnectLocked(client, partial);
        }
    } // lock is destroyed, allow further connect calls

    // Important: release the mutex here so the client can call back into the service from its
    // destructor (can be at the end of the call)
    device = client;
    return ret;
}
```

### 2.3.1 【HAL】 创建 Camera 客户端 makeClient( )

接着调用

*makeClient(this, cameraCb, clientPackageName, id, facing, clientPid, clientUid, getpid(), legacyMode, halVersion, deviceVersion, effectiveApiLevel, /out/&tmp))*

```c++
frameworks/av/services/camera/libcameraservice/CameraService.cpp

Status CameraService::makeClient(const sp<CameraService>& cameraService,
        const sp<IInterface>& cameraCb, const String16& packageName, int cameraId,
        int facing, int clientPid, uid_t clientUid, int servicePid, bool legacyMode,
        int halVersion, int deviceVersion, apiLevel effectiveApiLevel,
        /*out*/sp<BasicClient>* client) {

    if (halVersion < 0 || halVersion == deviceVersion) {
        // Default path: HAL version is unspecified by caller, create CameraClient
        // based on device version reported by the HAL.
        switch(deviceVersion) {
          case CAMERA_DEVICE_API_VERSION_1_0:
            if (effectiveApiLevel == API_1) {  // Camera1 API route
                sp<ICameraClient> tmp = static_cast<ICameraClient*>(cameraCb.get());
                *client = new CameraClient(cameraService, tmp, packageName, cameraId, facing,
                        clientPid, clientUid, getpid(), legacyMode);
            } else { // Camera2 API route
                ALOGW("Camera using old HAL version: %d", deviceVersion);
                return STATUS_ERROR_FMT(ERROR_DEPRECATED_HAL,
                        "Camera device \"%d\" HAL version %d does not support camera2 API",
                        cameraId, deviceVersion);
            }
            break;
          case CAMERA_DEVICE_API_VERSION_3_0:
          case CAMERA_DEVICE_API_VERSION_3_1:
          case CAMERA_DEVICE_API_VERSION_3_2:
          case CAMERA_DEVICE_API_VERSION_3_3:
          case CAMERA_DEVICE_API_VERSION_3_4:
            if (effectiveApiLevel == API_1) { // Camera1 API route
                sp<ICameraClient> tmp = static_cast<ICameraClient*>(cameraCb.get());
                *client = new Camera2Client(cameraService, tmp, packageName, cameraId, facing,
                        clientPid, clientUid, servicePid, legacyMode);
            } else { // Camera2 API route
                sp<hardware::camera2::ICameraDeviceCallbacks> tmp =
                        static_cast<hardware::camera2::ICameraDeviceCallbacks*>(cameraCb.get());
                *client = new CameraDeviceClient(cameraService, tmp, packageName, cameraId,
                        facing, clientPid, clientUid, servicePid);
            }
            break;
        }
    } else {
        // A particular HAL version is requested by caller. Create CameraClient
        // based on the requested HAL version.
        if (deviceVersion > CAMERA_DEVICE_API_VERSION_1_0 &&
            halVersion == CAMERA_DEVICE_API_VERSION_1_0) {
            // Only support higher HAL version device opened as HAL1.0 device.
            sp<ICameraClient> tmp = static_cast<ICameraClient*>(cameraCb.get());
            *client = new CameraClient(cameraService, tmp, packageName, cameraId, facing,
                    clientPid, clientUid, servicePid, legacyMode);
        } else {
            // Other combinations (e.g. HAL3.x open as HAL2.x) are not supported yet.
            ALOGE("Invalid camera HAL version %x: HAL %x device can only be"
                    " opened as HAL %x device", halVersion, deviceVersion,
                    CAMERA_DEVICE_API_VERSION_1_0);
            return STATUS_ERROR_FMT(ERROR_ILLEGAL_ARGUMENT,
                    "Camera device \"%d\" (HAL version %d) cannot be opened as HAL version %d",
                    cameraId, deviceVersion, halVersion);
        }
    }
    return Status::ok();
}
```

接着调用 

```c++
client = new Camera2Client(cameraService, tmp, packageName, cameraId, facing, clientPid, clientUid, servicePid, legacyMode);
```

client是通过 Camera2ClientBase() 进行创建的

```c++
frameworks/av/services/camera/libcameraservice/api1/Camera2Client.cpp
Camera2Client::Camera2Client(const sp<CameraService>& cameraService,
        const sp<hardware::ICameraClient>& cameraClient,
        const String16& clientPackageName,
        int cameraId,
        int cameraFacing,
        int clientPid,
        uid_t clientUid,
        int servicePid,
        bool legacyMode):
        Camera2ClientBase(cameraService, cameraClient, clientPackageName,
                cameraId, cameraFacing, clientPid, clientUid, servicePid),
		mParameters(cameraId, cameraFacing)
{
    ATRACE_CALL();

    SharedParameters::Lock l(mParameters);
    l.mParameters.state = Parameters::DISCONNECTED;

    mLegacyMode = legacyMode;
}
```

创建Camera3Device 对象

```c++
// Interface used by CameraService

template <typename TClientBase>
Camera2ClientBase<TClientBase>::Camera2ClientBase(
        const sp<CameraService>& cameraService,
        const sp<TCamCallbacks>& remoteCallback,
        const String16& clientPackageName,
        int cameraId,
        int cameraFacing,
        int clientPid,
        uid_t clientUid,
        int servicePid):
        TClientBase(cameraService, remoteCallback, clientPackageName,cameraId, cameraFacing, clientPid, clientUid, servicePid),
        mSharedCameraCallbacks(remoteCallback),
        mDeviceVersion(cameraService->getDeviceVersion(cameraId)),
        mDeviceActive(false)
{
    ALOGI("Camera %d: Opened. Client: %s (PID %d, UID %d)", cameraId, String8(clientPackageName).string(), clientPid, clientUid);

    mInitialClientPid = clientPid;
    mDevice = new Camera3Device(cameraId);
    LOG_ALWAYS_FATAL_IF(mDevice == 0, "Device should never be NULL here.");
}
```

```c++
frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp

Camera3Device::Camera3Device(int id):
        mId(id),
        mIsConstrainedHighSpeedConfiguration(false),
        mHal3Device(NULL),
        mStatus(STATUS_UNINITIALIZED),
        mStatusWaiters(0),
        mUsePartialResult(false),
        mNumPartialResults(1),
        mTimestampOffset(0),
        mNextResultFrameNumber(0),
        mNextReprocessResultFrameNumber(0),
        mNextShutterFrameNumber(0),
        mNextReprocessShutterFrameNumber(0),
        mListener(NULL)
{
    ATRACE_CALL();
    camera3_callback_ops::notify = &sNotify;
    camera3_callback_ops::process_capture_result = &sProcessCaptureResult;
    ALOGV("%s: Created device for camera %d", __FUNCTION__, id);
}
```

### 2.3.2 【HAL】 Camera2Client初始化 client->initialize(mModule))

1. 调用 Camera3Device::initialize 对/dev/videox 调备节点进行初始化，获取对应的设备的opt 操作函数
2. 初始化 Camera 默认参数
3. 创建 Camera 运行时的六大线程
   + StreamingProcessor：用来处理preview和录像 stream的线程。
   + FrameProcessor：用来处理3A和人脸识别的线程。
   + CaptureSequencer：拍照流程状态机线程，拍照的场景非常多，后面会发出状态机的运转流程。
   + JpegProcessor：用来处理拍照 jpeg stream 线程
   + ZslProcessor3：这个处理零延时zsl stream使用的线程
   + mCallbackProcessor：处理callback的线程，主要包含对 callback stream 的创建函数 updateStream()以及处理 HAL 层传上来的 callback stream 的线程

```c++
frameworks/av/services/camera/libcameraservice/api1/Camera2Client.cpp

status_t Camera2Client::initialize(CameraModule *module)
{
    ATRACE_CALL();
    ALOGV("%s: Initializing client for camera %d", __FUNCTION__, mCameraId);
    status_t res;

	// 1. 调用 Camera3Device::initialize 进行设备初始化,获取对应的 camera device 的opt 操作函数
    res = Camera2ClientBase::initialize(module);
    ==============> 
    	@ frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp
    	mDevice->initialize(module);
    	mDevice->setNotifyCallback(weakThis);
	<==============
	
	// 2. 初始化 Camera  默认参数
	{
        SharedParameters::Lock l(mParameters);

        res = l.mParameters.initialize(&(mDevice->info()), mDeviceVersion);
		================>
		+	@ frameworks/av/services/camera/libcameraservice/api1/client2/Parameters.cpp
		+	status_t Parameters::initialize(const CameraMetadata *info, int deviceVersion) {
		+		......
		+		videoWidth = previewWidth;
    	+		videoHeight = previewHeight;
    	+		params.setPreviewSize(previewWidth, previewHeight);
		+	    params.setVideoSize(videoWidth, videoHeight);
		+	    params.set(CameraParameters::KEY_PREFERRED_PREVIEW_SIZE_FOR_VIDEO, 
		+	    				String8::format("%dx%d", previewWidth, previewHeight));
		+	    params.set(CameraParameters::KEY_SUPPORTED_PREVIEW_FORMATS, supportedPreviewFormats);
		+	    ......  省略一系列 参数初始化代码
		<================
    }


    String8 threadName;
	// 3. 创建 Camera 运行时的六大线程
	// 3.1 StreamingProcessor：用来处理preview和录像 stream的线程。
    mStreamingProcessor = new StreamingProcessor(this);
    threadName = String8::format("C2-%d-StreamProc", mCameraId);
    mFrameProcessor->run(threadName.string());
    
	// 3.2 FrameProcessor：用来处理3A和人脸识别的线程。
    mFrameProcessor = new FrameProcessor(mDevice, this);
    threadName = String8::format("C2-%d-FrameProc",mCameraId);
    mFrameProcessor->run(threadName.string());
	
	// 3.3 CaptureSequencer：拍照流程状态机线程，拍照的场景非常多，后面会发出状态机的运转流程。
    mCaptureSequencer = new CaptureSequencer(this);
    threadName = String8::format("C2-%d-CaptureSeq",mCameraId);
    mCaptureSequencer->run(threadName.string());

	// 3.4 JpegProcessor：用来处理拍照 jpeg stream 线程
    mJpegProcessor = new JpegProcessor(this, mCaptureSequencer);
    threadName = String8::format("C2-%d-JpegProc",mCameraId);
    mJpegProcessor->run(threadName.string());

	// 3.5 ZslProcessor3：这个处理零延时zsl stream使用的线程
    mZslProcessor = new ZslProcessor(this, mCaptureSequencer);
    threadName = String8::format("C2-%d-ZslProc",mCameraId);
    mZslProcessor->run(threadName.string());

	// 3.6 mCallbackProcessor：处理callback的线程，主要包含对 callback stream 的创建函数 updateStream()以及处理 HAL 层传上来的 callback stream 的线程
    mCallbackProcessor = new CallbackProcessor(this);
    threadName = String8::format("C2-%d-CallbkProc",mCameraId);
    mCallbackProcessor->run(threadName.string());

    return OK;
}
```

#### 2.3.2.1【HAL】Camera device 初始化 mDevice->initialize(module)

1. 调用 hardware 层的camera_module_t.common 中的 open 方法，传入Camera ID
2. 获取 Camera相关的信息
3. 初始化 Camera 设备

```c++
frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp
status_t Camera3Device::initialize(CameraModule *module)
{
    ALOGV("%s: Initializing device for camera %d", __FUNCTION__, mId);
    /** Open HAL device */
    String8 deviceName = String8::format("%d", mId);

    camera3_device_t *device;
    ATRACE_BEGIN("camera3->open");
    // 1. 调用 hardware 层的camera_module_t.common 中的 open 方法，传入Camera ID
    res = module->open(deviceName.string(), reinterpret_cast<hw_device_t**>(&device));
    ===============> @ frameworks/av/services/camera/libcameraservice/common/CameraModule.cpp 
    +	res = filterOpenErrorCode(mModule->common.methods->open(&mModule->common, id, device));
    +	=========>
    +		@ hardware/qcom/camera/QCamera2/QCamera2Hal.cpp 
    +		struct hw_module_methods_t QCamera2Factory::mModuleMethods = {
	+			.open = QCamera2Factory::camera_device_open,
	+				=====>
	+					@ hardware/qcom/camera/QCamera2/QCamera2Factory.cpp
	+					rc = gQCamera2Factory->cameraDeviceOpen(atoi(id), hw_device);
	+		};
	+	<=========
	+<===============
	// 2. 获取 Camera相关的信息
    camera_info info;
    res = module->getCameraInfo(mId, &info);

	// 3. 初始化 Camera 设备，设置回调函数
    /** Initialize device with callback functions */
    ATRACE_BEGIN("camera3->initialize");
    res = device->ops->initialize(device, this);
    ATRACE_END();
	================>
	+	@ hardware/qcom/camera/QCamera2/HAL3/QCamera3HWI.cpp
	+	camera3_device_ops_t QCamera3HardwareInterface::mCameraOps = {
    +		.initialize			= QCamera3HardwareInterface::initialize,
    +		============>
    +			int rc = hw->initialize(callback_ops);
	<================
	
    /** Start up status tracker thread */
    mStatusTracker = new StatusTracker(this);
    res = mStatusTracker->run(String8::format("C3Dev-%d-Status", mId).string());
    if (res != OK) {
        SET_ERR_L("Unable to start status tracking thread: %s (%d)",
                strerror(-res), res);
        device->common.close(&device->common);
        mStatusTracker.clear();
        return res;
    }

    /** Register in-flight map to the status tracker */
    mInFlightStatusId = mStatusTracker->addComponent();

    /** Create buffer manager */
    mBufferManager = new Camera3BufferManager();

    bool aeLockAvailable = false;
    camera_metadata_ro_entry aeLockAvailableEntry;
    res = find_camera_metadata_ro_entry(info.static_camera_characteristics,
            ANDROID_CONTROL_AE_LOCK_AVAILABLE, &aeLockAvailableEntry);
    if (res == OK && aeLockAvailableEntry.count > 0) {
        aeLockAvailable = (aeLockAvailableEntry.data.u8[0] ==
                ANDROID_CONTROL_AE_LOCK_AVAILABLE_TRUE);
    }

    /** Start up request queue thread */
    mRequestThread = new RequestThread(this, mStatusTracker, device, aeLockAvailable);
    res = mRequestThread->run(String8::format("C3Dev-%d-ReqQueue", mId).string());
    if (res != OK) {
        SET_ERR_L("Unable to start request queue thread: %s (%d)",
                strerror(-res), res);
        device->common.close(&device->common);
        mRequestThread.clear();
        return res;
    }

    mPreparerThread = new PreparerThread();

    /** Everything is good to go */

    mDeviceVersion = device->common.version;
    mDeviceInfo = info.static_camera_characteristics;
    mHal3Device = device;

    // Determine whether we need to derive sensitivity boost values for older devices.
    // If post-RAW sensitivity boost range is listed, so should post-raw sensitivity control
    // be listed (as the default value 100)
    if (mDeviceVersion < CAMERA_DEVICE_API_VERSION_3_4 &&
            mDeviceInfo.exists(ANDROID_CONTROL_POST_RAW_SENSITIVITY_BOOST_RANGE)) {
        mDerivePostRawSensKey = true;
    }

    internalUpdateStatusLocked(STATUS_UNCONFIGURED);
    mNextStreamId = 0;
    mDummyStreamId = NO_STREAM;
    mNeedConfig = true;
    mPauseStateNotify = false;

    // Measure the clock domain offset between camera and video/hw_composer
    camera_metadata_entry timestampSource =
            mDeviceInfo.find(ANDROID_SENSOR_INFO_TIMESTAMP_SOURCE);
    if (timestampSource.count > 0 && timestampSource.data.u8[0] ==
            ANDROID_SENSOR_INFO_TIMESTAMP_SOURCE_REALTIME) {
        mTimestampOffset = getMonoToBoottimeOffset();
    }

    // Will the HAL be sending in early partial result metadata?
    if (mDeviceVersion >= CAMERA_DEVICE_API_VERSION_3_2) {
        camera_metadata_entry partialResultsCount =
                mDeviceInfo.find(ANDROID_REQUEST_PARTIAL_RESULT_COUNT);
        if (partialResultsCount.count > 0) {
            mNumPartialResults = partialResultsCount.data.i32[0];
            mUsePartialResult = (mNumPartialResults > 1);
        }
    } else {
        camera_metadata_entry partialResultsQuirk =
                mDeviceInfo.find(ANDROID_QUIRKS_USE_PARTIAL_RESULT);
        if (partialResultsQuirk.count > 0 && partialResultsQuirk.data.u8[0] == 1) {
            mUsePartialResult = true;
        }
    }

    camera_metadata_entry configs =
            mDeviceInfo.find(ANDROID_SCALER_AVAILABLE_STREAM_CONFIGURATIONS);
    for (uint32_t i = 0; i < configs.count; i += 4) {
        if (configs.data.i32[i] == HAL_PIXEL_FORMAT_IMPLEMENTATION_DEFINED &&
                configs.data.i32[i + 3] ==
                ANDROID_SCALER_AVAILABLE_STREAM_CONFIGURATIONS_INPUT) {
            mSupportedOpaqueInputSizes.add(Size(configs.data.i32[i + 1],
                    configs.data.i32[i + 2]));
        }
    }

    return OK;
}
```

#### 2.3.2.2【Hardware】打开设备 gQCamera2Factory->cameraDeviceOpen( )

```c++
hardware/qcom/camera/QCamera2/QCamera2Factory.cpp

int QCamera2Factory::cameraDeviceOpen(int camera_id, struct hw_device_t **hw_device)
{
    LOGI("Open camera id %d API version %d",camera_id, mHalDescriptors[camera_id].device_version);

    if ( mHalDescriptors[camera_id].device_version == CAMERA_DEVICE_API_VERSION_3_0 ) {
        CAMSCOPE_INIT(CAMSCOPE_SECTION_HAL);
        QCamera3HardwareInterface *hw = new QCamera3HardwareInterface(mHalDescriptors[camera_id].cameraId, mCallbacks);

        rc = hw->openCamera(hw_device);
    }
#ifdef QCAMERA_HAL1_SUPPORT
    else if (mHalDescriptors[camera_id].device_version == CAMERA_DEVICE_API_VERSION_1_0) {
        QCamera2HardwareInterface *hw = new QCamera2HardwareInterface((uint32_t)camera_id);

        rc = hw->openCamera(hw_device);
    }
#endif
    return rc;
}
```

初始化 `QCamera3HardwareInterface()`

```c++
hardware/qcom/camera/QCamera2/HAL3/QCamera3HWI.cpp
QCamera3HardwareInterface::QCamera3HardwareInterface(uint32_t cameraId, const camera_module_callbacks_t *callbacks)
    : mCameraId(cameraId),
      mCameraHandle(NULL),
      mCameraInitialized(false),
      mCallbacks(callbacks),
	...... //省略一系列Camera 初始化默认值
{
	// 1. 获取HAL 层 log 等级
    getLogLevel();
    =========>
    	property_get("persist.camera.hal.debug", prop, "0");
    	property_get("persist.camera.kpi.debug", prop, "0");
    	property_get("persist.camera.global.debug", prop, "0");
    	
    mCommon.init(gCamCapability[cameraId]);
    mCameraDevice.common.tag = HARDWARE_DEVICE_TAG;
    mCameraDevice.common.version = CAMERA_DEVICE_API_VERSION_3_4;

    mCameraDevice.common.close = close_camera_device;
    mCameraDevice.ops = &mCameraOps;
    	===⇒ @ hardware/qcom/camera/QCamera2/HAL3/QCamera3HWI.cpp
    mCameraDevice.priv = this;
    gCamCapability[cameraId]->version = CAM_HAL_V3;
    
    // TODO: hardcode for now until mctl add support for min_num_pp_bufs
    //TBD - To see if this hardcoding is needed. Check by printing if this is filled by mctl to 3
    gCamCapability[cameraId]->min_num_pp_bufs = 3;
    pthread_condattr_t mCondAttr;

    pthread_condattr_init(&mCondAttr);
    pthread_condattr_setclock(&mCondAttr, CLOCK_MONOTONIC);

    pthread_cond_init(&mBuffersCond, &mCondAttr);

    pthread_cond_init(&mRequestCond, &mCondAttr);
    pthread_cond_init(&mHdrRequestCond, &mCondAttr);

    pthread_condattr_destroy(&mCondAttr);

    mPendingLiveRequest = 0;
    mCurrentRequestId = -1;
    pthread_mutex_init(&mMutex, NULL);

    for (size_t i = 0; i < CAMERA3_TEMPLATE_COUNT; i++)
        mDefaultMetadata[i] = NULL;

    // Getting system props of different kinds
    char prop[PROPERTY_VALUE_MAX];
    memset(prop, 0, sizeof(prop));
    // 获取 persist.camera.raw.dump，是否使能 RAW dump 功能
    property_get("persist.camera.raw.dump", prop, "0");
    mEnableRawDump = atoi(prop);
    // 获取 persist.camera.hal3.force.hdr，是否使能 hdr 功能
    property_get("persist.camera.hal3.force.hdr", prop, "0");
    mForceHdrSnapshot = atoi(prop);

    if (mEnableRawDump)
        LOGD("Raw dump from Camera HAL enabled");

    memset(&mInputStreamInfo, 0, sizeof(mInputStreamInfo));
    memset(mLdafCalib, 0, sizeof(mLdafCalib));

    memset(prop, 0, sizeof(prop));
    property_get("persist.camera.tnr.preview", prop, "0");
    m_bTnrPreview = (uint8_t)atoi(prop);

    memset(prop, 0, sizeof(prop));
    property_get("persist.camera.swtnr.preview", prop, "1");
    m_bSwTnrPreview = (uint8_t)atoi(prop);

    memset(prop, 0, sizeof(prop));
    property_get("persist.camera.tnr.video", prop, "0");
    m_bTnrVideo = (uint8_t)atoi(prop);

    memset(prop, 0, sizeof(prop));
    property_get("persist.camera.avtimer.debug", prop, "0");
    m_debug_avtimer = (uint8_t)atoi(prop);
    LOGI("AV timer enabled: %d", m_debug_avtimer);

    memset(prop, 0, sizeof(prop));
    property_get("persist.camera.cacmode.disable", prop, "0");
    m_cacModeDisabled = (uint8_t)atoi(prop);

    mRdiModeFmt = gCamCapability[mCameraId]->rdi_mode_stream_fmt;
    //Load and read GPU library.
    lib_surface_utils = NULL;
    LINK_get_surface_pixel_alignment = NULL;
    mSurfaceStridePadding = CAM_PAD_TO_32;
	// 打开 libadreno_utils.so
    lib_surface_utils = dlopen("libadreno_utils.so", RTLD_NOW);
    if (lib_surface_utils) {
        *(void **)&LINK_get_surface_pixel_alignment =
                dlsym(lib_surface_utils, "get_gpu_pixel_alignment");
         if (LINK_get_surface_pixel_alignment) {
             mSurfaceStridePadding = LINK_get_surface_pixel_alignment();
         }
         dlclose(lib_surface_utils);
    }

    if (gCamCapability[cameraId]->is_quadracfa_sensor) {
        LOGI("Sensor support Quadra CFA mode");
        m_bQuadraCfaSensor = true;
    }

    m_bQuadraCfaRequest = false;
    m_bQuadraSizeConfigured = false;
    memset(&mStreamList, 0, sizeof(camera3_stream_configuration_t));
}
```

接下来，调用 `hw->openCamera(hw_device);`

```c++
hardware/qcom/camera/QCamera2/HAL3/QCamera3HWI.cpp

int QCamera3HardwareInterface::openCamera(struct hw_device_t **hw_device)
{
    if (mState != CLOSED) {
        *hw_device = NULL;
        return PERMISSION_DENIED;
    }

    mPerfLockMgr.acquirePerfLock(PERF_LOCK_OPEN_CAMERA);
    LOGI("[KPI Perf]: E PROFILE_OPEN_CAMERA camera id %d",mCameraId);

    rc = openCamera();
    if (rc == 0) {
        *hw_device = &mCameraDevice.common;
    } else {
        *hw_device = NULL;
    }

    LOGI("[KPI Perf]: X PROFILE_OPEN_CAMERA camera id %d, rc: %d", mCameraId, rc);

    if (rc == NO_ERROR) {
        mState = OPENED;
    }
    return rc;
}
```

接下来调用 `int QCamera3HardwareInterface::openCamera()`

```c++
@ hardware/qcom/camera/QCamera2/HAL3/QCamera3HWI.cpp
int QCamera3HardwareInterface::openCamera()
{
    KPI_ATRACE_CAMSCOPE_CALL(CAMSCOPE_HAL3_OPENCAMERA);
    if (mCameraHandle) {
        LOGE("Failure: Camera already opened");
        return ALREADY_EXISTS;
    }

    rc = QCameraFlash::getInstance().reserveFlashForCamera(mCameraId);
    if (rc < 0) {
        LOGE("Failed to reserve flash for camera id: %d",
                mCameraId);
        return UNKNOWN_ERROR;
    }

    rc = camera_open((uint8_t)mCameraId, &mCameraHandle);
    
    rc = mCameraHandle->ops->register_event_notify(mCameraHandle->camera_handle, camEvtHandle, (void *)this);


    mExifParams.debug_params = (mm_jpeg_debug_exif_params_t *) malloc (sizeof(mm_jpeg_debug_exif_params_t));
    if (mExifParams.debug_params) {
        memset(mExifParams.debug_params, 0, sizeof(mm_jpeg_debug_exif_params_t));
    } else {
        LOGE("Out of Memory. Allocation failed for 3A debug exif params");
        return NO_MEMORY;
    }
    mFirstConfiguration = true;

    //Notify display HAL that a camera session is active.
    //But avoid calling the same during bootup because camera service might open/close
    //cameras at boot time during its initialization and display service will also internally
    //wait for camera service to initialize first while calling this display API, resulting in a
    //deadlock situation. Since boot time camera open/close calls are made only to fetch
    //capabilities, no need of this display bw optimization.
    //Use "service.bootanim.exit" property to know boot status.
    property_get("service.bootanim.exit", value, "0");
    if (atoi(value) == 1) {
        pthread_mutex_lock(&gCamLock);
        if (gNumCameraSessions++ == 0) {
            setCameraLaunchStatus(true);
        }
        pthread_mutex_unlock(&gCamLock);
    }

    //fill the session id needed while linking dual cam
    pthread_mutex_lock(&gCamLock);
    rc = mCameraHandle->ops->get_session_id(mCameraHandle->camera_handle,
        &sessionId[mCameraId]);
    pthread_mutex_unlock(&gCamLock);

    if (rc < 0) {
        LOGE("Error, failed to get sessiion id");
        return UNKNOWN_ERROR;
    } else {
        //Allocate related cam sync buffer
        //this is needed for the payload that goes along with bundling cmd for related
        //camera use cases
        m_pDualCamCmdHeap = new QCamera3HeapMemory(1);
        rc = m_pDualCamCmdHeap->allocate(sizeof(cam_dual_camera_cmd_info_t));
        if(rc != OK) {
            rc = NO_MEMORY;
            LOGE("Dualcam: Failed to allocate Related cam sync Heap memory");
            return NO_MEMORY;
        }

        //Map memory for related cam sync buffer
        rc = mCameraHandle->ops->map_buf(mCameraHandle->camera_handle,
                CAM_MAPPING_BUF_TYPE_DUAL_CAM_CMD_BUF,
                m_pDualCamCmdHeap->getFd(0),
                sizeof(cam_dual_camera_cmd_info_t),
                m_pDualCamCmdHeap->getPtr(0));
        if(rc < 0) {
            LOGE("Dualcam: failed to map Related cam sync buffer");
            rc = FAILED_TRANSACTION;
            return NO_MEMORY;
        }
        m_pDualCamCmdPtr = (cam_dual_camera_cmd_info_t*) DATA_PTR(m_pDualCamCmdHeap,0);
    }

    LOGH("mCameraId=%d",mCameraId);

    return NO_ERROR;
}
```

**(1) camera_open((uint8_t)mCameraId, &mCameraHandle)**

打开Camera 成功后，hardware 层中将Camera 所有信息保存在 `g_cam_ctrl.cam_obj[cam_idx]` 中，同时返回对应的camera 的opt 操作函数

```c
hardware/qcom/camera/QCamera2/stack/mm-camera-interface/src/mm_camera_interface.c
int32_t camera_open(uint8_t camera_idx, mm_camera_vtbl_t **camera_vtbl)
{
    int32_t rc = 0;
    mm_camera_obj_t *cam_obj = NULL;
    uint32_t cam_idx = camera_idx;
    uint32_t aux_idx = 0;
    uint8_t is_multi_camera = 0;

#ifdef QCAMERA_REDEFINE_LOG
    mm_camera_debug_open();
#endif

    LOGD("E camera_idx = %d\n", camera_idx);
    if (is_dual_camera_by_idx(camera_idx)) {
        is_multi_camera = 1;
        cam_idx = mm_camera_util_get_handle_by_num(0,g_cam_ctrl.cam_index[camera_idx]);
        aux_idx = (get_aux_camera_handle(g_cam_ctrl.cam_index[camera_idx]) >> MM_CAMERA_HANDLE_SHIFT_MASK);
        LOGH("Dual Camera: Main ID = %d Aux ID = %d", cam_idx, aux_idx);
    }

    pthread_mutex_lock(&g_intf_lock);
    /* opened already */
    if(NULL != g_cam_ctrl.cam_obj[cam_idx] && g_cam_ctrl.cam_obj[cam_idx]->ref_count != 0) {
        pthread_mutex_unlock(&g_intf_lock);
        LOGE("Camera %d is already open", cam_idx);
        return -EBUSY;
    }

    cam_obj = (mm_camera_obj_t *)malloc(sizeof(mm_camera_obj_t));

    /* initialize camera obj */
    memset(cam_obj, 0, sizeof(mm_camera_obj_t));
    cam_obj->ctrl_fd = -1;
    cam_obj->ds_fd = -1;
    cam_obj->ref_count++;
    cam_obj->my_num = 0;
    cam_obj->my_hdl = mm_camera_util_generate_handler(cam_idx);
    cam_obj->vtbl.camera_handle = cam_obj->my_hdl; /* set handler */
    cam_obj->vtbl.ops = &mm_camera_ops;
    pthread_mutex_init(&cam_obj->cam_lock, NULL);
    pthread_mutex_init(&cam_obj->muxer_lock, NULL);
    /* unlock global interface lock, if not, in dual camera use case,
      * current open will block operation of another opened camera obj*/
    pthread_mutex_lock(&cam_obj->cam_lock);
    pthread_mutex_unlock(&g_intf_lock);

    rc = mm_camera_open(cam_obj);

    if (is_multi_camera) {
        /*Open Aux camer's*/
        pthread_mutex_lock(&g_intf_lock);
        if(NULL != g_cam_ctrl.cam_obj[aux_idx] &&
                g_cam_ctrl.cam_obj[aux_idx]->ref_count != 0) {
            pthread_mutex_unlock(&g_intf_lock);
            LOGE("Camera %d is already open", aux_idx);
            rc = -EBUSY;
        } else {
            pthread_mutex_lock(&cam_obj->muxer_lock);
            pthread_mutex_unlock(&g_intf_lock);
            rc = mm_camera_muxer_camera_open(aux_idx, cam_obj);
        }
    }

    LOGH("Open succeded: handle = %d", cam_obj->vtbl.camera_handle);
    g_cam_ctrl.cam_obj[cam_idx] = cam_obj;
    *camera_vtbl = &cam_obj->vtbl;
    return 0;
}
```


此时Open 对应的 /dev/video 节点，获得对应的session_id，创建好相关的 socket ，最终所有信息，保存在 `mm_camera_obj_t * cam_obj`节构体中。

```c
@ hardware/qcom/camera/QCamera2/stack/mm-camera-interface/src/mm_camera.c

int32_t mm_camera_open(mm_camera_obj_t *my_obj)
{
    char dev_name[MM_CAMERA_DEV_NAME_LEN];
    int8_t n_try=MM_CAMERA_DEV_OPEN_TRIES;
    uint8_t sleep_msec=MM_CAMERA_DEV_OPEN_RETRY_SLEEP;
    int cam_idx = 0;
    const char *dev_name_value = NULL;

    LOGD("begin\n");

    dev_name_value = mm_camera_util_get_dev_name_by_num(my_obj->my_num, my_obj->my_hdl);

    snprintf(dev_name, sizeof(dev_name), "/dev/%s",dev_name_value);
    sscanf(dev_name, "/dev/video%d", &cam_idx);
    LOGD("dev name = %s, cam_idx = %d", dev_name, cam_idx);

    do{
        n_try--;
        errno = 0;
        my_obj->ctrl_fd = open(dev_name, O_RDWR | O_NONBLOCK);
        l_errno = errno;
        LOGD("ctrl_fd = %d, errno == %d", my_obj->ctrl_fd, l_errno);
        if((my_obj->ctrl_fd >= 0) || (errno != EIO && errno != ETIMEDOUT) || (n_try <= 0 )) {
            break;
        }
        LOGE("Failed with %s error, retrying after %d milli-seconds", strerror(errno), sleep_msec);
        usleep(sleep_msec * 1000U);
    }while (n_try > 0);

  	mm_camera_get_session_id(my_obj, &my_obj->sessionid);
  	LOGH("Camera Opened id = %d sessionid = %d", cam_idx, my_obj->sessionid);

#ifdef DAEMON_PRESENT
    /* open domain socket*/
    n_try = MM_CAMERA_DEV_OPEN_TRIES;
    do {
        n_try--;
        my_obj->ds_fd = mm_camera_socket_create(cam_idx, MM_CAMERA_SOCK_TYPE_UDP);
        l_errno = errno;
        LOGD("ds_fd = %d, errno = %d", my_obj->ds_fd, l_errno);
        if((my_obj->ds_fd >= 0) || (n_try <= 0 )) {
            LOGD("opened, break out while loop");
            break;
        }
        LOGD("failed with I/O error retrying after %d milli-seconds", sleep_msec);
        usleep(sleep_msec * 1000U);
    } while (n_try > 0);

#else /* DAEMON_PRESENT */
    cam_status_t cam_status;
    cam_status = mm_camera_module_open_session(my_obj->sessionid,mm_camera_module_event_handler);
#endif /* DAEMON_PRESENT */

    pthread_condattr_init(&cond_attr);
    pthread_condattr_setclock(&cond_attr, CLOCK_MONOTONIC);

    pthread_mutex_init(&my_obj->msg_lock, NULL);
    pthread_mutex_init(&my_obj->cb_lock, NULL);
    pthread_mutex_init(&my_obj->evt_lock, NULL);
    pthread_cond_init(&my_obj->evt_cond, &cond_attr);
    pthread_condattr_destroy(&cond_attr);

    LOGD("Launch evt Thread in Cam Open");
    snprintf(my_obj->evt_thread.threadName, THREAD_NAME_SIZE, "CAM_Dispatch");
    mm_camera_cmd_thread_launch(&my_obj->evt_thread, mm_camera_dispatch_app_event, (void *)my_obj);

    /* launch event poll thread
     * we will add evt fd into event poll thread upon user first register for evt */
    LOGD("Launch evt Poll Thread in Cam Open");
    snprintf(my_obj->evt_poll_thread.threadName, THREAD_NAME_SIZE, "CAM_evntPoll");
    mm_camera_poll_thread_launch(&my_obj->evt_poll_thread, MM_CAMERA_POLL_TYPE_EVT);
    mm_camera_evt_sub(my_obj, TRUE);

    /* unlock cam_lock, we need release global intf_lock in camera_open(),
     * in order not block operation of other Camera in dual camera use case.*/
    pthread_mutex_unlock(&my_obj->cam_lock);
    LOGD("end (rc = %d)\n", rc);
    return rc;
}
```

**(2) Camera 操作节构体 cam_obj->vtbl.ops**

```c
hardware/qcom/camera/QCamera2/stack/mm-camera-interface/src/mm_camera_interface.c

static mm_camera_ops_t mm_camera_ops = {
    .query_capability = mm_camera_intf_query_capability,
    .register_event_notify = mm_camera_intf_register_event_notify,
    .close_camera = mm_camera_intf_close,
    .set_parms = mm_camera_intf_set_parms,
    .get_parms = mm_camera_intf_get_parms,
    .do_auto_focus = mm_camera_intf_do_auto_focus,
    .cancel_auto_focus = mm_camera_intf_cancel_auto_focus,
    .prepare_snapshot = mm_camera_intf_prepare_snapshot,
    .start_zsl_snapshot = mm_camera_intf_start_zsl_snapshot,
    .stop_zsl_snapshot = mm_camera_intf_stop_zsl_snapshot,
    .map_buf = mm_camera_intf_map_buf,
    .map_bufs = mm_camera_intf_map_bufs,
    .unmap_buf = mm_camera_intf_unmap_buf,
    .add_channel = mm_camera_intf_add_channel,
    .delete_channel = mm_camera_intf_del_channel,
    .get_bundle_info = mm_camera_intf_get_bundle_info,
    .add_stream = mm_camera_intf_add_stream,
    .link_stream = mm_camera_intf_link_stream,
    .delete_stream = mm_camera_intf_del_stream,
    .config_stream = mm_camera_intf_config_stream,
    .qbuf = mm_camera_intf_qbuf,
    .cancel_buffer = mm_camera_intf_cancel_buf,
    .get_queued_buf_count = mm_camera_intf_get_queued_buf_count,
    .map_stream_buf = mm_camera_intf_map_stream_buf,
    .map_stream_bufs = mm_camera_intf_map_stream_bufs,
    .unmap_stream_buf = mm_camera_intf_unmap_stream_buf,
    .set_stream_parms = mm_camera_intf_set_stream_parms,
    .get_stream_parms = mm_camera_intf_get_stream_parms,
    .start_channel = mm_camera_intf_start_channel,
    .stop_channel = mm_camera_intf_stop_channel,
    .request_super_buf = mm_camera_intf_request_super_buf,
    .cancel_super_buf_request = mm_camera_intf_cancel_super_buf_request,
    .flush_super_buf_queue = mm_camera_intf_flush_super_buf_queue,
    .configure_notify_mode = mm_camera_intf_configure_notify_mode,
    .process_advanced_capture = mm_camera_intf_process_advanced_capture,
    .get_session_id = mm_camera_intf_get_session_id,
    .set_dual_cam_cmd = mm_camera_intf_set_dual_cam_cmd,
    .flush = mm_camera_intf_flush,
    .register_stream_buf_cb = mm_camera_intf_register_stream_buf_cb,
    .register_frame_sync = mm_camera_intf_reg_frame_sync,
    .handle_frame_sync_cb = mm_camera_intf_handle_frame_sync_cb
};
```

#### 2.3.2.3 【Hardware】 初始化Framewroks层的Callback 函数 QCamera3HardwareInterface::initialize( )

```c++
hardware/qcom/camera/QCamera2/HAL3/QCamera3HWI.cpp

/*===========================================================================
 * FUNCTION   : initialize
 * DESCRIPTION: Initialize frameworks callback functions
 *   @callback_ops : callback function to frameworks
 *==========================================================================*/
int QCamera3HardwareInterface::initialize(const struct camera3_callback_ops *callback_ops)
{
    ATRACE_CAMSCOPE_CALL(CAMSCOPE_HAL3_INIT);
    LOGI("E :mCameraId = %d mState = %d", mCameraId, mState);

    rc = initParameters();
    
    mCallbackOps = callback_ops;
    =====> 该Callback 是从 frameworks/av/services/camera/libcameraservice/CameraService.cpp 中传递下来的
    	if (mModule->getModuleApiVersion() >= CAMERA_MODULE_API_VERSION_2_1) {
	        mModule->setCallbacks(this);
	    }

    mChannelHandle = mCameraHandle->ops->add_channel( mCameraHandle->camera_handle, NULL, NULL, this);
    
    mCameraInitialized = true;
    mState = INITIALIZED;
    LOGI("X");
    return 0;
}
```

## 2.4 事件处理程序（）

在 EventHandler 中，根据具体的事件，调用不同的 callback 函数。

```java
@ frameworks/base/core/java/android/hardware/Camera.java
private class EventHandler extends Handler
    {
        private final Camera mCamera;

        public EventHandler(Camera c, Looper looper) {
            super(looper);
            mCamera = c;
        }

        @Override
        public void handleMessage(Message msg) {
            switch(msg.what) {
            case CAMERA_MSG_SHUTTER:
                if (mShutterCallback != null) {
                    mShutterCallback.onShutter();
                }
                return;

            case CAMERA_MSG_RAW_IMAGE:
                if (mRawImageCallback != null) {
                    mRawImageCallback.onPictureTaken((byte[])msg.obj, mCamera);
                }
                return;

            case CAMERA_MSG_COMPRESSED_IMAGE:
                if (mJpegCallback != null) {
                    mJpegCallback.onPictureTaken((byte[])msg.obj, mCamera);
                }
                return;

            case CAMERA_MSG_PREVIEW_FRAME:
                PreviewCallback pCb = mPreviewCallback;
                if (pCb != null) {
                    if (mOneShot) {
                        // Clear the callback variable before the callback
                        // in case the app calls setPreviewCallback from
                        // the callback function
                        mPreviewCallback = null;
                    } else if (!mWithBuffer) {
                        // We're faking the camera preview mode to prevent
                        // the app from being flooded with preview frames.
                        // Set to oneshot mode again.
                        setHasPreviewCallback(true, false);
                    }
                    pCb.onPreviewFrame((byte[])msg.obj, mCamera);
                }
                return;

            case CAMERA_MSG_POSTVIEW_FRAME:
                if (mPostviewCallback != null) {
                    mPostviewCallback.onPictureTaken((byte[])msg.obj, mCamera);
                }
                return;

            case CAMERA_MSG_FOCUS:
                AutoFocusCallback cb = null;
                synchronized (mAutoFocusCallbackLock) {
                    cb = mAutoFocusCallback;
                }
                if (cb != null) {
                    boolean success = msg.arg1 == 0 ? false : true;
                    cb.onAutoFocus(success, mCamera);
                }
                return;

            case CAMERA_MSG_ZOOM:
                if (mZoomListener != null) {
                    mZoomListener.onZoomChange(msg.arg1, msg.arg2 != 0, mCamera);
                }
                return;

            case CAMERA_MSG_PREVIEW_METADATA:
                if (mFaceListener != null) {
                    mFaceListener.onFaceDetection((Face[])msg.obj, mCamera);
                }
                return;

            case CAMERA_MSG_ERROR :
                Log.e(TAG, "Error " + msg.arg1);
                if (mErrorCallback != null) {
                    mErrorCallback.onError(msg.arg1, mCamera);
                }
                return;

            case CAMERA_MSG_FOCUS_MOVE:
                if (mAutoFocusMoveCallback != null) {
                    mAutoFocusMoveCallback.onAutoFocusMoving(msg.arg1 == 0 ? false : true, mCamera);
                }
                return;
            /* ### QC ADD-ONS: START */
            case CAMERA_MSG_STATS_DATA:
                int statsdata[] = new int[257];
                for(int i =0; i<257; i++ ) {
                   statsdata[i] = byteToInt( (byte[])msg.obj, i*4);
                }
                if (mCameraDataCallback != null) {
                     mCameraDataCallback.onCameraData(statsdata, mCamera);
                }
                return;

            case CAMERA_MSG_META_DATA:
                if (mCameraMetaDataCallback != null) {
                    mCameraMetaDataCallback.onCameraMetaData((byte[])msg.obj, mCamera);
                }
                return;
            /* ### QC ADD-ONS: END */
            default:
                Log.e(TAG, "Unknown message type " + msg.what);
                return;
            }
        }
    }
```

至此，整个open Camera 流程就完了。
