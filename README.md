# Android CHIPTool

This directory contains the Android Studio project for CHIPTool, an Android
application for commissioning and controlling CHIP accessories.

CHIPTool offers the following features:

-   Scan a CHIP QR code and display payload information to the user
-   Read the NFC tag containing CHIP onboarding information
-   Commission a CHIP device
-   Send echo requests to the CHIP echo server
-   Send on/off cluster requests to a CHIP device

> :warning: Connection to the CHIP device's soft AP will be a manual step until
> pairing is implemented.

For information about how to build the application, see the
[Building Android CHIPTool](../../../docs/guides/android_building.md) guide.
# ChipTool-Android

> 项目提取自matter项目的`ChipTool`，已经运行脚本生成jar、so库，但`/app/libs/jniLibs/arm64-v8a/libCHIPController.so`文件大小超过300m，所以压缩分包成`lib.zip.00x`等6个包，clone后先解压出来`libCHIPController.so`

## 主要代码
目前demo里主要代码，单配网使用仅需要几个主要工具类以及jar、so
- matter/src/main/java/com/orvibo/matter
  - /attestation
    - ExampleAttestationTrustStoreDelegate.kt // 设备证书
  - /bluetooth
    - BluetoothManager.kt //蓝牙工具类
  - /setuppayloadscanner
    - CHIPDeviceInfo.kt //解密后的二维码信息类
    - QrCodeInfo.kt //二维码信息
  - /util
    - DeviceIdUtil.kt // 工具类
  - ChipClient.kt // 配网native方法调用
  - GenericChipDeviceListener.kt // 过程回调接口
  - NetworkCredentialsParcelable.kt // Wi-Fi、Thread信息配置类

## 配网流程
- 通过扫码获取到二维码字符串，识别二维码为`MT：`开头，即为matter二维码
  - 调用`matter`库中的`SetupPayloadParser`解析二维码信息，转为`SetupPayload`对象
  - 通过`CHIPDeviceInfo.Companion.fromSetupPayload(payload, isShortDiscriminator, mac)`将`SetupPayload`地址转为`CHIPDeviceInfo`设备信息对象
    - 携带`CHIPDeviceInfo`进入绑定流程页面
        - 获取到Wi-Fi列表、上一次成功的ssid，进入Wi-Fi配置页面
            - 选择Wi-Fi、输入密码之后，组合成`NetworkCredentialsParcelable`传递到下一步
        - 进入配网、绑定页面：
            - 连接成功后，获取到`BluetoothGatt`以及connectionId，在`onConnected`回调里进入下一步
            - 连接失败，进入提示页面
                - 问题：
                    - `ChipDeviceController.setDeviceAttestationDelegate(timeout, delegate);`中的timeout是wifi配网超时时间，即配网之后多久重启蓝牙广播，即便失败，也需要继续等待，所以配网失败之后导致扫码之后一直连不上蓝牙的问题
                    - 扫码后蓝牙扫描直接跳转失败页面问题，`onScanFailed`返回`errCode=2`，是因短时间不停发起蓝牙连接、没有正常关闭蓝牙移除回调导致，见`https://blog.csdn.net/sxk874890728/article/details/95049676`
            - 第一步，检查权限之后，开始蓝牙搜索
                - 开启一个新线程，初始化`BluetoothManager`蓝牙连接工具类，是matter的demo中提取的类
                - `BluetoothManager.getBluetoothDevice()`搜索设备，matter库是用kotlin写的，这里会返回一个携程执行
                    - 如果没有发现设备或者sdk报错，`resumeWith`会返回一个空对象（需要日志检查c层报错），跳转错误页面
                    - 返回的是一个`BluetoothDevice`对象则代表搜索到matter设备，可以开始连接
                    - `BluetoothManager.connect()`开始连接设备
                        - 如果连接成功，`resumeWith`中会返回一个`BluetoothGatt`对象，即蓝牙服务集合实例，以及`connectionId`用于后面配网使用，调用`onConnected`函数进入下一步
                        - 如果连接失败，跳到错误页面（连接蓝牙失败、配网失败两者错误页面提示语不一样）
            - 第二步，设备配网连接wifi：
                - 调用`pairDevice()`方法发送Wi-Fi及密码给到设备，开始配网
                  - 注意：调用这一步之后，点击返回后下一次配网有概率出现so库空指针闪退
                - 问题：
                    - 错误码：
                        - 返回错误码50：大概率是超时，返回也有可能连上Wi-Fi了，但如果故意输错密码有时也会返回50
                        - 返回错误码172：内部错误，有可能是输错密码/Wi-Fi连接不上
                    - 返回错误码不为0，即失败，也按照正常流程查询uid、判断设备在不在线，因为错误码不明确，有时连上wifi超时也会50
                    - 主动关闭蓝牙连接，Wi-Fi配网失败之后，蓝牙一直没有断开，导致下一次扫码后连不上蓝牙
