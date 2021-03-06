# Android 蓝牙连接功能的相关问题整理

## 1. 蓝牙连接需要的相关权限及说明

* `BLUETOOTH`：您需要此权限才能执行任何蓝牙通信，例如请求连接、接受连接和传输数据等。
* `ACCESS_FINE_LOCATION`：您的应用需要此权限，因为蓝牙扫描可用于收集用户的位置信息。此类信息可能来自用户自己的设备，以及在商店和交通设施等位置使用的蓝牙信标。
* `BLUETOOTH_ADMIN` ：大多数应用只是需利用此权限发现本地蓝牙设备。除非应用是根据用户请求修改蓝牙设置的“超级管理员”，否则不应使用此权限所授予的其他功能。
* `ACCESS_COARSE_LOCATION` ：如果您的应用适配 Android 9（API 级别 28）或更低版本，则您可以声明 `ACCESS_COARSE_LOCATION` 权限而非 `ACCESS_FINE_LOCATION` 权限。

```markup
<manifest ... >
  <uses-permission android:name="android.permission.BLUETOOTH" />
  <uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />

  <!-- If your app targets Android 9 or lower, you can declare
       ACCESS_COARSE_LOCATION instead. -->
  <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
  ...
</manifest>
```

* 如果您要声明您的应用仅适用于支持 BLE 的设备，请在应用清单中添加以下内容：

  ```markup
  <uses-feature android:name="android.hardware.bluetooth_le" android:required="true"/>
  ```

  可以在代码中进行判断是否支持BLE：

  ```java
  // Use this check to determine whether BLE is supported on the device. Then
  // you can selectively disable BLE-related features.
  if (!getPackageManager().hasSystemFeature(PackageManager.FEATURE_BLUETOOTH_LE)) {
      Toast.makeText(this, R.string.ble_not_supported, Toast.LENGTH_SHORT).show();
      finish();
  }
  ```

## 2. Android各版本BLE方案可能存在的问题总结

1. 从 Android 3.0 开始，Bluetooth API 便支持使用蓝牙配置文件。_蓝牙配置文件_是适用于设备间蓝牙通信的无线接口规范。举个例子：免提配置文件。如果手机要与无线耳机进行连接，则两台设备都必须支持免提配置文件。
2. Android 4.0（API 级别 14）引入了对蓝牙健康设备配置文件 \(HDP\) 的支持。该配置文件允许您创建应用，从而使用蓝牙与支持蓝牙功能的健康设备（例如心率监测仪、血糖仪、温度计、台秤等）进行通信。有关支持的设备及其相应的设备数据专业化代码列表，请参阅蓝牙的 [HDP 设备数据专业化](https://www.bluetooth.com/specifications/assigned-numbers/health-device-profile)。这些值在 ISO/IEEE 11073-20601 \[7\] 规范的“命名法规附录”中也被称为 MDC_DEV\_SPEC\_PROFILE_\*。有关 HDP 的详细讨论，请参阅[健康设备规范](https://developer.android.com/guide/topics/connectivity/bluetooth?hl=zh-cn#HDP)。
3. Android 5.0，开始支持外设API的支持（BluetoothLeAdvertiser）蓝牙广播。
4. Android 6.0 以上版本需要位置权限授予，否则可能会扫描不到设备（Android 7.0 有可能需要手动打开权限配置）
5. Android 7.0 为了防止BLE扫描滥用，做了一些限制，即不要在30s内对蓝牙扫描 重复开启-关闭超过5次。所以建议单次扫描时间间隔大于等于6s。或者在30s内不要连续调用stopScan这个方法，连续调用startScan这个方法很多次都不会存在问题，会持续返回扫描到的设备数据
6. Android 8.0 支持自定义在尝试通过蓝牙、BLE 和 WLAN 与配套设备进行配对时显示的配对请求对话框。
7. Android 8.1系统，在屏幕关闭之后，扫描会被暂停，如果需要在屏幕被关闭之后还继续进行扫描的话，需要在扫描配置那里，加入一个空的过滤器，如下：

   ```java
   List<ScanFilter> filters = Collections.singletonList(new Builder().build());
   ```

8. Androoid 9.0系统，息屏扫描问题解决方案，三星、华为等手机通过8.1的方案处理还是出现了息屏无法进行扫描的问题，需要通过过滤配置来解决，添加了过滤条件后，如以下操作，在息屏状态下，系统也会回调EddyStone和ibeacon的设备数据到APP里，该方案只能解决扫描配置过的设备，无法扫描没有配置过的设备类型；

   ```java
   //添加EddyStone的过滤条件
   scanFilter.add(new ScanFilter.Builder()
             .setServiceUuid(ParcelUuid.fromString("0000feaa-0000-1000-8000-00805f9b34fb"))
             .build());
   //添加ibeacon 的过滤条件
   scanFilter.add(new ScanFilter.Builder()
             .setManufacturerData(0x004c, new byte[]{})
             .build());
   scanner.startScan(scanFilter, scanSettings, mScanCallback);
   ```

9. Android 10.0，华为和其他一部分手机如果设备广播间隔稍微长了一些，可能就会出现扫描不到设备的问题，处理的方案为改为批量回调筛选设备，该方案针对扫描特定型号设备时比较好使，如果换成其他手机，还是改回单个回调的方法较优。

   ```java
   ScanSettings.Builder builder = new ScanSettings.Builder()
                   .setScanMode(ScanSettings.SCAN_MODE_LOW_LATENCY);
   //改动这个值，改为不为0，即可开启批量回调，这时候单个回调方法不执行，单位毫秒
   builder.setReportDelay(600);

   private ScanCallback mScanCallback = new ScanCallback() {
           @Override
           public void onScanResult(int callbackType, final ScanResult result) {
               //系统单个回调方法
           }

           @Override
           public void onBatchScanResults(List<ScanResult> results) {
               super.onBatchScanResults(results);
               //批量回调方法
           }

           @Override
           public void onScanFailed(int errorCode) {
               super.onScanFailed(errorCode);
           }
       };
   ```

## 3. 通过BLE连接

### 3.1 概述

Android 4.3（API 级别 18）为发挥_核心作用_的蓝牙低功耗 \(BLE\) 引入内置平台支持，并提供相应 API，方便应用发现设备、查询服务和传输信息。与[传统蓝牙](https://developer.android.com/guide/topics/connectivity/bluetooth?hl=zh-cn)不同，蓝牙低功耗 \(BLE\) 旨在提供显著降低的功耗。这使 Android 应用可与功率要求更严格的 BLE 设备（例如近程传感器、心率监测仪和健身设备）通信。

**注意：**当用户使用 BLE 将其设备与其他设备配对时，用户设备上的**所有**应用都可以访问在这两个设备间传输的数据。因此，如果您的应用捕获敏感数据，您应实现应用层安全以保护此类数据的私密性。

### 3.2 关键术语和概念

以下是对 BLE 关键术语和概念的总结：

* **通用属性配置文件 \(GATT\)**— GATT 配置文件是一种通用规范，内容针对在 BLE 链路上发送和接收称为“属性”的简短数据片段。目前所有低功耗应用配置文件均以 GATT 为基础。
  * 蓝牙特别兴趣小组 \(Bluetooth SIG\) 为低功耗设备定义诸多[配置文件](https://www.bluetooth.org/en-us/specification/adopted-specifications)。配置文件是描述设备如何在特定应用中工作的规范。请注意，一台设备可以实现多个配置文件。例如，一台设备可能包含心率监测仪和电池电量检测器。
* **属性协议 \(ATT\)** — 属性协议 \(ATT\) 是 GATT 的构建基础，二者的关系也被称为 GATT/ATT。ATT 经过优化，可在 BLE 设备上运行。为此，该协议尽可能少地使用字节。每个属性均由通用唯一标识符 \(UUID\) 进行唯一标识，后者是用于对信息进行唯一标识的字符串 ID 的 128 位标准化格式。由 ATT 传输的_属性_采用_特征_和_服务_格式。
* **特征** — 特征包含一个值和 0 至多个描述特征值的描述符。您可将特征理解为类型，后者与类类似。 
* **描述符** — 描述符是描述特征值的已定义属性。例如，描述符可指定人类可读的描述、特征值的可接受范围或特定于特征值的度量单位。
* **Service** — 服务是一系列特征。例如，您可能拥有名为“心率监测器”的服务，其中包括“心率测量”等特征。您可以在 [bluetooth.org](https://www.bluetooth.org/en-us/specification/adopted-specifications) 上找到基于 GATT 的现有配置文件和服务的列表。

### 3.3 初始化BLE

一切的的前提是需要设备支持BLE可以参考第1节中对BLE检测的代码，进行配置。

1. 获取 `BluetoothAdapter`

   `BluetoothAdapter` 代表设备自身的蓝牙适配器（蓝牙无线装置），整个系统有一个蓝牙适配器。

   ```java
   private BluetoothAdapter bluetoothAdapter;
   ...
   // Initializes Bluetooth adapter.
   final BluetoothManager bluetoothManager =
           (BluetoothManager) getSystemService(Context.BLUETOOTH_SERVICE);
   bluetoothAdapter = bluetoothManager.getAdapter();
   ```

2. 启用蓝牙

   下一步，您需要确保蓝牙已启用。调用 `isEnabled()`，以检查当前是否已启用蓝牙。如果此方法返回 false，则表示蓝牙处于停用状态。如果未开启可以通过下面的代码前往设置开启蓝牙

   ```java
   // Ensures Bluetooth is available on the device and it is enabled. If not,
   // displays a dialog requesting user permission to enable Bluetooth.
   if (bluetoothAdapter == null || !bluetoothAdapter.isEnabled()) {
       Intent enableBtIntent = new Intent(BluetoothAdapter.ACTION_REQUEST_ENABLE);
       startActivityForResult(enableBtIntent, REQUEST_ENABLE_BT);
   }
   ```

### 3.4 扫描设备

Android提供了两套扫描设备的API ：

* `startLeScan()`&`stopLeScan()`：这套API已经被标记为过时了，主要针对于5.0以前的系统。
* `startScan()`&`stopScan()`：这套API针对于5.1以后的版本，新的API中已经封装了方法来解析广播数据。

  \`\`\`java //新API,需要Android 5.0\(API Level 21\)及以上版本才能使用 //启动扫描 private void scanNew\(\) { BluetoothManager bluetoothManager= \(BluetoothManager\) getSystemService\(Context.BLUETOOTH\_SERVICE\); //基本的扫描方法 bluetoothManager .getAdapter\(\) .getBluetoothLeScanner\(\) .startScan\(mScanCallback\);

```text
//设置一些扫描参数
ScanSettings settings=new ScanSettings
    .Builder()
    //例如这里设置的低延迟模式，也就是更快的扫描到周围设备，相应耗电也更厉害
    .setScanMode(ScanSettings.SCAN_MODE_LOW_LATENCY)
    .build();

//你需要设置的过滤条件,不只可以像旧API中的按服务UUID过滤
//还可以按设备名称，MAC地址等条件过滤
List<ScanFilter> scanFilters=new ArrayList<>();

//如果你需要过滤扫描到的设备可以用下面的这种构造方法
bluetoothManager
    .getAdapter()
    .getBluetoothLeScanner()
    .startScan(scanFilters,settings,mScanCallback);
```

}

//扫描结果回调 ScanCallback mScanCallback = new ScanCallback\(\) { @Override public void onScanResult\(int callbackType, ScanResult result\) { //callbackType:扫描模式 //result：扫描到的设备数据，包含蓝牙设备对象，解析完成的广播数据等 } };

//停止扫描 private void stopNewScan\(\){ BluetoothManager bluetoothManager= \(BluetoothManager\) getSystemService\(Context.BLUETOOTH\_SERVICE\); bluetoothManager.getAdapter\(\).getBluetoothLeScanner\(\).stopScan\(mScanCallback\); }

```text
**补充：**

ScanFilter是用来过滤的，目前支持UUID，mac地址，设备名称等。

#### 3.5 连接设备

同一时间我们只能对一个外围设备发起连接，如果需要对多个设备连接可以等上一个连接成功后再进行下一个连接，否则如果前面的某个连接操作失败了没有回调，后面的操作会被一直阻塞。

```java
//发起连接
private void connect(BluetoothDevice device){
  mBluetoothGatt = device.connectGatt(context, false, mBluetoothGattCallback);
}

//Gatt操作回调，此回调很重要，后面所有的操作结果都会在此方法中回调
BluetoothGattCallback mBluetoothGattCallback = new BluetoothGattCallback() {
   @Override
   public void onConnectionStateChange(BluetoothGatt gatt, int status, int newState) {
     //gatt:GATT客户端
     //status:此次操作的状态码，返回0时代表操作成功，返回其他值就是各种异常
     //newState:当前连接处于的状态，例如连接成功，断开连接等

     //当连接状态改变时触发此回调
   }

   @Override
   public void onServicesDiscovered(BluetoothGatt gatt, int status) {
     //gatt:GATT客户端
     //status:此次操作的状态码，返回0时代表操作成功，返回其他值就是各种异常

     //成功获取服务时触发此回调，
   }

   @Override
   public void onCharacteristicRead(BluetoothGatt gatt,
       final BluetoothGattCharacteristic characteristic, final int status) {
         //gatt:GATT客户端
         //status:此次操作的状态码，返回0时代表操作成功，返回其他值就是各种异常
         //characteristic:被读的特征

         //当对特征的读操作完成时触发此回调，
   }

   @Override
   public void onCharacteristicWrite(BluetoothGatt gatt,
       final BluetoothGattCharacteristic characteristic, final int status) {
         //gatt:GATT客户端
         //status:此次操作的状态码，返回0时代表操作成功，返回其他值就是各种异常
         //characteristic:被写的特征

         //当对特征的写操作完成时触发此回调
   }

   @Override
   public void onCharacteristicChanged(BluetoothGatt gatt,
       final BluetoothGattCharacteristic characteristic) {
         //gatt:GATT客户端
         //status:此次操作的状态码，返回0时代表操作成功，返回其他值就是各种异常
         //characteristic：特征值改变的特征

         //当特征值改变时触发此回调，
   }

   @Override
   public void onDescriptorRead(BluetoothGatt gatt, BluetoothGattDescriptor descriptor,
       int status) {
         //gatt:GATT客户端
         //status:此次操作的状态码，返回0时代表操作成功，返回其他值就是各种异常
         //descriptor:被读的descriptor

         //当对descriptor的读操作完成时触发
   }

   @Override
   public void onDescriptorWrite(BluetoothGatt gatt, BluetoothGattDescriptor descriptor,
       int status) {
         //gatt:GATT客户端
         //status:此次操作的状态码，返回0时代表操作成功，返回其他值就是各种异常
         //descriptor:被写的descriptor

   }
 };
```

#### 3.5.1 发现服务

连接成功后，触发`BluetoothGattCallback#onConnectionStateChange()`方法。

```java
@Override
   public void onConnectionStateChange(BluetoothGatt gatt, int status, int newState) {
     //gatt:GATT客户端
     //status:此次操作的状态码，返回0时代表操作成功，返回其他值就是各种异常
     //newState:当前连接处于的状态，例如连接成功，断开连接等

     //当连接状态改变时触发此回调

            //判断是否连接码
      if (newState == BluetoothProfile.STATE_CONNECTED) {
              //发现服务
          mBluetoothGatt.discoverServices();
      }else if(newState == BluetoothProfile.STATE_DISCONNECTED){
                    //判断是否断开连接码           
      }     
   }
```

当发现服务成功后，会触发`BluetoothGattCallback#onServicesDiscovered()`回调：

```java
        //服务发现回调
    @Override
    public void onServicesDiscovered(BluetoothGatt gatt, int status) {
        super.onServicesDiscovered(gatt, status);
        if (status == BluetoothGatt.GATT_SUCCESS) {
            mHandler.post(() ->
                //获取指定uuid的service
                BluetoothGattService gattService = mBluetoothGatt.getService(mServiceUUID);
                //获取到特定的服务不为空
                if(gattService != null){
                    //获取指定uuid的Characteristic
                    BluetoothGattCharacteristic gattCharacteristic = gattService.getCharacteristic(mCharacteristicUUID);
                    //获取特定特征成功
                    if（gattCharacteristic != null）{
                        //写入你需要传递给外设的特征值（即传递给外设的信息）
                        gattCharacteristic.setValue(bytes);
                        //通过GATt实体类将，特征值写入到外设中。
                        mBluetoothGatt.writeCharacteristic(gattCharacteristic);

                        //如果只是需要读取外设的特征值：
                        //通过Gatt对象读取特定特征（Characteristic）的特征值
                        mBluetoothGatt.readCharacteristic(gattCharacteristic);
                    }
                }else{
                    //获取特定服务失败

                }
            );
        }
    }
```

#### 3.5.2 读取和修改特征值

当成功读取特征值时，会触发`BluetoothGattCallback#onCharacteristicRead()`回调。

```java
@Override
public void onCharacteristicRead(BluetoothGatt gatt, BluetoothGattCharacteristic characteristic, int status) {
    super.onCharacteristicRead(gatt, characteristic, status);
    if (status == BluetoothGatt.GATT_SUCCESS) {
        //获取读取到的特征值
        characteristic.getValue()
    ｝
}
```

当成功写入特征值到外设时，会触发`BluetoothGattCallback#onCharacteristicWrite()`回调。

```java
@Override
public void onCharacteristicWrite(BluetoothGatt gatt, BluetoothGattCharacteristic characteristic, int status) {
    super.onCharacteristicWrite(gatt, characteristic, status);
    if (status == BluetoothGatt.GATT_SUCCESS) {
        //获取写入到外设的特征值
        characteristic.getValue()
    ｝
}
```

#### 3.5.3 监听外设特征值改变

无论是对外设写入新值，还是读取外设特定`Characteristic`的值，其实都只是单方通信。如果需要双向通信，可以在`BluetoothGattCallback#onServicesDiscovered`中对某个特征值设置监听（**前提是该`Characteristic`具有NOTIFY属性**）：

```java
//设置订阅notificationGattCharacteristic值改变的通知
mBluetoothGatt.setCharacteristicNotification(notificationGattCharacteristic, true);
//获取其对应的通知Descriptor
BluetoothGattDescriptor descriptor = notificationGattCharacteristic.getDescriptor(UUID.fromString("00002902-0000-1000-8000-00805f9b34fb"));
if (descriptor != null){ 
    //设置通知值
    descriptor.setValue(BluetoothGattDescriptor.ENABLE_INDICATION_VALUE);
    boolean descriptorResult = mBluetoothGatt.writeDescriptor(descriptor);
}
```

当写入完特征值后，外设修改自己的特征值进行回复时，手机端会触发`BluetoothGattCallback#onCharacteristicChanged()`方法，获取到外设回复的值，从而实现双向通信。

```java
@Override
public void onCharacteristicChanged(BluetoothGatt gatt, BluetoothGattCharacteristic characteristic) {
     if (status == BluetoothGatt.GATT_SUCCESS) {
        //获取外设修改的特征值
        String value = characteristic.getValue()
        //对特征值进行解析

    ｝
｝
```

## 4. BLE服务端——健康设备

### 4.1 BLE广播

在蓝牙开发中，有些情况是不需要连接的，只要外设设备（以下简称外设）广播自己的数据即可，例如苹果的`ibeacon`。自`Android 5.0`更新蓝牙API后，手机可以作为外设广播数据。

广播包有两种：

* 广播包（Advertising Data）
* 响应包（Scan Response）

其中**广播包是每个外设都必须广播的，而响应包是可选的**。

开启广播一般需要3~4对象：广播设置（AdvertiseSettings）、广播包（AdvertiseData）、扫描包（可选）、广播回调（AdvertiseCallback）。

### 4.2 广播设置

```java
//初始化广播设置
mAdvertiseSettings = new AdvertiseSettings.Builder()
        //设置广播模式，以控制广播的功率和延迟。
        .setAdvertiseMode(AdvertiseSettings.ADVERTISE_MODE_LOW_POWER)
        //发射功率级别
        .setTxPowerLevel(AdvertiseSettings.ADVERTISE_TX_POWER_HIGH)
        //不得超过180000毫秒。值为0将禁用时间限制即持续发送广播。
        .setTimeout(3000)
        //设置是否可以连接
        .setConnectable(false)
        .build();
```

setAdvertiseMode共有三种模式：

* 使用高TX功率级别进行广播：`AdvertiseSettings#ADVERTISE_TX_POWER_HIGH`
* 使用低TX功率级别进行广播：`AdvertiseSettings#ADVERTISE_TX_POWER_LOW`
* 使用中等TX功率级别进行广播：`AdvertiseSettings#ADVERTISE_TX_POWER_MEDIUM`
* 使用最低传输（TX）功率级别进行广播：`AdvertiseSettings#ADVERTISE_TX_POWER_ULTRA_LOW`

### 4.3 广播包和扫描包（扫描包可选）

之前说过，外设必须广播广播包，扫描包是可选。但添加扫描包也意味着广播更多得数据，即可广播62个字节。

```java
//初始化广播包
mAdvertiseData = new AdvertiseData.Builder()
        //设置广播设备名称
        .setIncludeDeviceName(true)
        //设置发射功率级别
        .setIncludeDeviceName(true)
        .build();

//初始化扫描响应包
mScanResponseData = new AdvertiseData.Builder()
        //隐藏广播设备名称
        .setIncludeDeviceName(false)
        //隐藏发射功率级别
        .setIncludeDeviceName(false)
        //设置广播的服务`UUID`
        .addService`UUID`(new Parcel`UUID`(UUID_SERVICE))
        //设置厂商数据
        .addManufacturerData(0x11,hexStrToByte(mData))
        .build();
```

可见无论是广播包还是扫描包，其广播的内容都是用`AdvertiseData`类封装的。

（1）`AdvertiseData.Builder#setIncludeDeviceName()`方法，可以设置广播包中是否包含蓝牙的名称。

（2）`AdvertiseData.Builder#setIncludeTxPowerLevel()`方法，可以设置广播包中是否包含蓝牙的发射功率。

（3）`AdvertiseData.Builder#addService`UUID`(Parcel`UUID`)`方法，可以设置特定的`UUID`在广播包中。

（4）`AdvertiseData.Builder#addServiceData(Parcel`UUID`，byte[])`方法，可以设置特定的`UUID`和其数据在广播包中。

（5）`AdvertiseData.Builder#addManufacturerData(int，byte[])`方法，可以设置特定厂商Id和其数据在广播包中。

从`AdvertiseData.Builder`的设置中可以看出，如果一个外设需要在不连接的情况下对外广播数据，其数据可以存储在`UUID`对应的数据中，也可以存储在厂商数据中。但由于厂商ID是需要由Bluetooth SIG进行分配的，厂商间一般都将数据设置在厂商数据。

### 4.4 发送广播

发送之前需要进一步确定广播是否开启成功，设置 AdvertiseCallback

```java
private class LKAdvertiseCallback extends AdvertiseCallback {
    //开启广播成功回调
    @Override
    public void onStartSuccess(AdvertiseSettings settingsInEffect){
        super.onStartSuccess(settingsInEffect);
        Log.d("leke","开启服务成功");
    }

    //无法启动广播回调。
    @Override
    public void onStartFailure(int errorCode) {
        super.onStartFailure(errorCode);
        Log.d("leke","开启服务失败，失败码 = " + errorCode);
    }
}
```

初始化完毕上面的对象后，就可以进行广播：

```java
//获取BLE广播的操作对象。
//如果蓝牙关闭或此设备不支持蓝牙LE广播，则返回null。
mBluetoothLeAdvertiser = mBluetoothAdapter.getBluetoothLeAdvertiser();
//mBluetoothLeAdvertiser不为空，且蓝牙已开打
if(mBluetoothAdapter.isEnabled()){
    if (mBluetoothLeAdvertiser != null){
         //开启广播
        mBluetoothLeAdvertiser.startAdvertising(mAdvertiseSettings,
            mAdvertiseData, mScanResponseData, mAdvertiseCallback);
    }else {
        Log.d("leke","该手机不支持ble广播");
    }
}else{
    Log.d("leke","手机蓝牙未开启");
}
```

此时手机通过BLE扫描可以检索到这台设备了。

### 4.5 创建Gatt Service

虽然通过广播告知外边自身拥有这些Service,但手机自身并没有初始化Gattd的Service。导致外部的中心设备连接手机后，并不能找到对应的`GATT Service` 和 获取对应的数据。

Service类型有两个级别：

* `BluetoothGattService#SERVICE_TYPE_PRIMARY` 主服务
* `BluetoothGattService#SERVICE_TYPE_SECONDARY`次要服务（存在于主服务中的服务）

创建`BluetoothGattService`时，传入两个参数：`UUID`和Service类型

```java
BluetoothGattService service = new BluetoothGattService(UUID_SERVICE,
                                BluetoothGattService.SERVICE_TYPE_PRIMARY);
```

### 4.6 创建Gatt Characteristic

我们都知道Gatt中，`Service`的下一级是`Characteristic`，`Characteristic`是最小的通信单元，通过对`Characteristic`进行读写操作来进行通信。

```java
//初始化特征值
mGattCharacteristic = new BluetoothGattCharacteristic(UUID_CHARACTERISTIC,
        BluetoothGattCharacteristic.PROPERTY_WRITE|
                BluetoothGattCharacteristic.PROPERTY_NOTIFY|
                BluetoothGattCharacteristic.PROPERTY_READ,
        BluetoothGattCharacteristic.PERMISSION_WRITE|
                BluetoothGattCharacteristic.PERMISSION_READ);
```

特征属性表示该`BluetoothGattCharacteristic`拥有什么功能，即能对`BluetoothGattCharacteristic`进行什么操作。其中主要有3种：

* `BluetoothGattCharacteristic#PROPERTY_WRITE` 表示特征支持写
* `BluetoothGattCharacteristic#PROPERTY_READ`  表示特征支持读
* `BluetoothGattCharacteristic#PROPERTY_NOTIFY`  表示特征支持通知

权限属性用于配置该特征值所具有的功能。主要两种：

* `BluetoothGattCharacteristic#PERMISSION_WRITE`  特征写权限
* `BluetoothGattCharacteristic#PERMISSION_READ`  特征读权限

### 4.7 创建Gatt Descriptor

`Characteristic`下还有`Descriptor`，初始化`BluetoothGattDescriptor`时传入：`Descriptor UUID` 和 权限属性

```java
//初始化描述
mGattDescriptor = new BluetoothGattDescriptor(UUID_DESCRIPTOR,BluetoothGattDescriptor.PERMISSION_WRITE);
```

### 4.8 添加 Characteristic 和 Descriptor

为`Service`添加`Characteristic`，为`Characteristic`添加`Descriptor`：

```java
//Service添加特征值
mGattService.addCharacteristic(mGattCharacteristic);
mGattService.addCharacteristic(mGattReadCharacteristic);
//特征值添加描述
mGattCharacteristic.addDescriptor(mGattDescriptor);
```

通过蓝牙管理器`mBluetoothManager`获取`Gatt Server`，用来添加`Gatt Service`。添加完`Gatt Service`后，外部中心设备连接手机时，将能获取到对应的`GATT Service` 和 获取对应的数据

```java
//初始化GattServer回调
mBluetoothGattServerCallback = new daqiBluetoothGattServerCallback();

if (mBluetoothManager != null)
    mBluetoothGattServer = mBluetoothManager.openGattServer(this, mBluetoothGattServerCallback);
boolean result = mBluetoothGattServer.addService(mGattService);
if (result){
    Toast.makeText(daqiActivity.this,"添加服务成功",Toast.LENGTH_SHORT).show();
}else {
    Toast.makeText(daqiActivity.this,"添加服务失败",Toast.LENGTH_SHORT).show();
}
```

定义`Gatt Server`回调。当中心设备连接该手机外设、修改特征值、读取特征值等情况时，会得到相应情况的回调。

```java
private class LKBluetoothGattServerCallback extends BluetoothGattServerCallback{

    //设备连接/断开连接回调
    @Override
    public void onConnectionStateChange(BluetoothDevice device, int status, int newState) {
        super.onConnectionStateChange(device, status, newState);
      //int: Returns the new connection state. Can be one of BluetoothProfile.STATE_DISCONNECTED or BluetoothProfile#STATE_CONNECTED
    }

    //添加本地服务回调
    @Override
    public void onServiceAdded(int status, BluetoothGattService service) {
        super.onServiceAdded(status, service);
      //int: Returns BluetoothGatt#GATT_SUCCESS if the service was added successfully.
    }

    //特征值读取回调
    @Override
    public void onCharacteristicReadRequest(BluetoothDevice device, int requestId, int offset, BluetoothGattCharacteristic characteristic) {
        super.onCharacteristicReadRequest(device, requestId, offset, characteristic);
    }

    //特征值写入回调 切片后的数据会多次返回
    @Override
    public void onCharacteristicWriteRequest(BluetoothDevice device, int requestId, BluetoothGattCharacteristic characteristic, boolean preparedWrite, boolean responseNeeded, int offset, byte[] value) {
        super.onCharacteristicWriteRequest(device, requestId, characteristic, preparedWrite, responseNeeded, offset, value);
    }

    //描述读取回调
    @Override
    public void onDescriptorReadRequest(BluetoothDevice device, int requestId, int offset, BluetoothGattDescriptor descriptor) {
        super.onDescriptorReadRequest(device, requestId, offset, descriptor);
    }

    //描述写入回调
    @Override
    public void onDescriptorWriteRequest(BluetoothDevice device, int requestId, BluetoothGattDescriptor descriptor, boolean preparedWrite, boolean responseNeeded, int offset, byte[] value) {
        super.onDescriptorWriteRequest(device, requestId, descriptor, preparedWrite, responseNeeded, offset, value);
    }
}
```

## 5. 对接过程中遇到奇怪的问题

### 5.1 外围设备连接不上，时而成功时而失败

autoConnect设置为true，Android底层会帮你发起重连请求，即时你断开连接，如果需要显示正常，可以有两种操作

1. 设置autoConnect为false
2. 每次断开连接调用disconnect的方法

### 5.2 ios手机连接设备的时候连接不上，或者连上之后马上断开

调查原因，发现是定义了两个BluetoothGattCharacteristic，且两个特征都具有读写特性，可以只保留一个具有读写和notify 的特征

### 5.3 数据发送过程中分包，支持20字节

默认每次传输只有20字节，且需要转成16进制，可以写个切片工具，自定义字符串起始节点，方便判断数据为同一个指令。

## 参考文档

1. [Android官方对蓝牙的描述](https://developer.android.com/guide/topics/connectivity/bluetooth?hl=zh-cn)
2. [Android官方蓝牙低功耗概览](https://developer.android.com/guide/topics/connectivity/bluetooth-le?hl=zh-cn#roles)
3. [Android BLE 处理方案](https://blog.csdn.net/chen_xi_hao/article/details/86664197)
4. [Android 版本与 Bluetooth 版本之间的关系](https://www.ifeegoo.com/relationship-between-android-version-and-bluetooth-version.html)
5. [BLE开发快速上手](http://noharry.cc/2018/10/24/AndroidBLE%E5%BF%AB%E9%80%9F%E4%B8%8A%E6%89%8B%E6%8C%87%E5%8D%97/)’
6. [BLE蓝牙广播](https://juejin.cn/post/6844904030645256205#heading-4)
7. [Android BLE开发详解和FastBle源码解析](https://www.jianshu.com/p/795bb0a08beb)

