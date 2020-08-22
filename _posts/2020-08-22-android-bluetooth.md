---
title: Android中使用蓝牙通信
layout: post
---

> 蓝牙(Bluetooth)是一种无线数据和语音通信开放的全球规范，基于低成本的近距离无线连接，为固定和移动设备建立通信环境的一种特殊的近距离无线技术连接。相比WIFI，蓝牙的优势在于连接不需要借助路由器或AP设备，可通过简单配对过程实现设备之间的快速连接。
>
> 以下将通过将Android系统作为服务端实现一个基于蓝牙的Socket服务器，客户端通过蓝牙连接后与其实现Socket通信。

#### 1. Android的Service

> 为了让蓝牙服务在启动后一直能够保持运行状态，需要使用Android中提供的Service，其能够在后台也保持运行状态。如实现的BluetoothService需要继承自Service类，并覆盖onBind和onStartCommand两个方法来实现服务的逻辑。
>
```java
public class BluetoothService extends Service {
    public IBinder onBind(Intent intent) {
        return null;
    }
    public int onStartCommand(Intent intent, int flags, int startId) {
        return START_STICKY;
    }
}
```
> 在Activity中可以启动Service，启动的代码如下
>
```java
startService(new Intent(getBaseContext(), BluetoothService.class));
```

#### 2. Android中启动蓝牙
> 首先需要在App中设置访问设备蓝牙的权限，在AndroidManifest.xml中增加以下权限申请
>
```xml
    <uses-permission android:name="android.permission.BLUETOOTH"/>
    <uses-permission android:name="android.permission.BLUETOOTH_ADMIN"/>
```
> 使用BluetoothAdapter类中的getDefaultAdapter方法可以获得系统的蓝牙适配器，如果系统没有蓝牙设备将会返回null，需要对此进行判断防止发生空指针异常。正常获得系统的蓝牙适配器后调用enable方法开启系统的蓝牙功能
>
```java
BluetoothAdapter adapter = BluetoothAdapter.getDefaultAdapter();
if (adapter == null) {
    return -1;
}
//这里可以判断是否已经开启，不过重复开启也不会导致程序出现错误
adapter.enable();
```
>
> 为了让其他设备能够发现此蓝牙服务，需要请求开启蓝牙的可发现性，使用申请提示待用户需要点击确认后，可启动蓝牙的可发现性一段时间（可设置，最大300秒）
```java
Intent intent = new Intent(BluetoothAdapter.ACTION_REQUEST_DISCOVERABLE);
//早期版本的Android，支持设置0时长期保持可发现状态
intent.putExtra(BluetoothAdapter.EXTRA_DISCOVERABLE_DURATION, 300);
startActivity(intent);
```

#### 3. 发现其他蓝牙设备

> 调用蓝牙适配器的startDiscovery方法可以搜索附近的蓝牙设备，但从Android 6开始启动蓝牙搜索需要获取定位的权限，同时定位的权限是不支持通过AndroidManifest中进行预先申请的，需要使用申请提示待用户需要点击确认，在Activity中增加以下代码
```java
ActivityCompat.requestPermissions(this,
        new String[]{Manifest.permission.ACCESS_COARSE_LOCATION, 
        Manifest.permission.ACCESS_FINE_LOCATION, 
        Manifest.permission.ACCESS_BACKGROUND_LOCATION}, 1);
```
> startDiscovery方法为异步运行，若想要在发现设备后执行相应的操作，需要设置回调类
>
```java
//过滤器，仅关注发现事件
IntentFilter filter = new IntentFilter();
filter.addAction(BluetoothDevice.ACTION_FOUND);
//注册回调类
registerReceiver(new BroadcastReceiver() {
    //实现回调函数，在发现设备后输出设备的名称或地址
    @Override
    public void onReceive(Context context, Intent intent) {
        BluetoothDevice device = intent.getParcelableExtra(BluetoothDevice.EXTRA_DEVICE);
        if (device != null) {
            //当没有设置设备名称是getName方法将会返回null
            String name = device.getName();
            Log.i("bluetooth", name != null ? name : device.toString());
        }
    }
}, filter);
```

#### 4. 实现蓝牙的SocketServer
>
> 实现与普通的TCP SocketServer非常类似，首先通过listen方法得到一个Socket，然后循环调用accept方法等待连接，当连接成功后获得输入输出流实现接收和发送数据，并通过一个线程来启动Server
```java
new Thread(new Runnable() {
    @Override
    public void run() {
        try {
            UUID uuid = UUID.fromString("b383ee98-fd44-4972-8e3e-48bf542b0a7b");
            //设置名称和uid来供客户端连接时进行识别
            BluetoothServerSocket socket = adapter.listenUsingRfcommWithServiceRecord("Rfc Server", uuid);
            Log.i("bluetooth", uuid.toString());
            Log.i("bluetooth", adapter.getName());
            Log.i("bluetooth", adapter.getAddress());
            while (true) {
                BluetoothSocket client = socket.accept();
                Log.i("bluetooth", "Accept");
                if (client != null) {
                    OutputStream os = client.getOutputStream();
                    PrintStream out = new PrintStream(os);
                    out.println("Hello!");
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}).start();
```

#### 5. 实现蓝牙的Client

> 这里使用Python实现了一个蓝牙的Client，使用pybluez库作为操作蓝牙的基础库，连接完成后接收数据，并关闭连接
>
```python
import bluetooth;
sock = bluetooth.BluetoothSocket(bluetooth.RFCOMM);
host = 'D4:12:43:66:C4:A3';
port = 3;
sock.connect((host, port));
print(sock.recv(1024));
sock.close();
```
> Client将会从Server收到一个Hello!的字符串，并输出。
>
> 总体感觉Android的坑还是挺多的，各个版本直接的差异还是比较明显的，同时在相关的文档中又没有很清楚的说明，如果需要适配多个版本的Android系统的额外工作量还是比较大的，同时需要的各种访问权限需要在UI中手动确认，作为服务程序每次启动和重启都要手动点击确认确实还是不太方便。
