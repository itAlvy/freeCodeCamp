# S10 Configuration Tool

> GL S10 蓝牙快速配置工具

IDE: Visual Studio Code

框架：Google Flutter

开发语言：dart.js

- [运行](#运行)
- [数据流](#数据流)
- [指令码](#指令码)
- [数据转义，解析](#数据转义，解析)
- [指令下发](#指令下发)
- [数据处理中心](#数据处理中心)

### 运行

```bash
# dev app
flutter run
```


### 数据流

每当连接上一个蓝牙设备时，可以拿到设备的信息 **device**，监听 **device.state** 可以读取蓝牙实时上报的信息 **characteristic** 和特征值写入的方法 **cmdCharacteristic**

![Data Flow](assets/data_flow.png)

### 指令码
```dart
const START_CODE = 0xFE;
const VERSION = 0x01;

const SET_WIFI_CONFIG = 0x01;
const START_WIFI = 0x02;
const STOP_WIFI = 0x03;
const RESTART_WIFI = 0x04;
const GET_NETWORK_STATE = 0x05;
const CLEAR_WIFI_CONFIG = 0x06;
const SET_MQTT_URI = 0x65;
const START_MQTT = 0x66;
const STOP_MQTT = 0x67;
const RESTART_MQTT = 0x68;
const GET_MQTT_CONFIG = 0x69;
const CLEAR_MQTT_CONFIG = 0x6A;
const MQTT_SUBSCRIBE = 0x6B;
const MQTT_UNSUBSCRIBE = 0x6C;
const MQTT_PUBLISH = 0x6D;
const MQTT_SUBSCRIBE_MSG = 0x6E;

const GET_VERSION = 0xF0;
const SET_OTA_URL = 0xF1;
const GET_STA_OTA = 0xF2;
```

### 数据转义，解析

所有数据解析的方法都封装在 **Util类** 中

- 蓝牙上报信息解析

  ```dart
  static Map parseMessage(List<int> data) {
    var back = Map();
    if (data.length >= 5) {
      var cmd = data[3];
      if (data[0] != START_CODE) {
        back['code'] = -3;
        back['msg'] = 'start code error';
        back['data'] = data;
        return back;
      }
      if (((data[1] << 8) + data[2]) != data.length) {
        back['code'] = -4;
        back['msg'] = 'length error';
        return back;
      }

      if (data.length == 5) {
        if (data[4] == 0) {
          back['code'] = 0;
          back['msg'] = 'success';
          back['cmd'] = cmd;
          return back;
        } else {
          back['code'] = -1;
          back['msg'] = 'failed';
          back['cmd'] = cmd;
          return back;
        }
      } else {
        data = cmd == MQTT_SUBSCRIBE_MSG ? data.sublist(4) : data.sublist(5);
        back['code'] = 0;
        back['msg'] = 'success';
        back['cmd'] = cmd;
        back['data'] = utf8.decode(data).toString();
        return back;
      }
    } else {
      back['code'] = -2;
      back['msg'] = 'length too short';
      return back;
    }
  }
  ```

- 写入数据解析

  ```dart
  static List<int> buildMessage(int cmdID, String msg) {
    List<int> tmp = List();
    tmp.add(START_CODE);
    int len = msg.length + 4;
    tmp.add((len >> 8) & 0xff);
    tmp.add(len & 0xff);
    tmp.add(cmdID);
    tmp.addAll(utf8.encode(msg));
    return tmp;
  }
  ```
### 指令下发


```dart
示例:

/// 传入指令码，消息参数
cmdCharacteristic.write(Util.buildMessage(0x01, "${_ssid.text}@${_password.text}"))
```

1. 特征值写入方法：`cmdCharacteristic.write()`
2. 传入 **指令码** 并**解析参数**：`Util.buildMessage(0x01, "ssid@psw")`

##### 1.设置 WiFi

> 参数格式：`ssid@password`

```dart
cmdCharacteristic.write(Util.buildMessage(SET_WIFI_CONFIG, "${_ssid.text}@${_password.text}"));
```

##### 2.设置 Mqtt

> 参数格式：`mqtt://username:password@host:port`

```dart
cmdCharacteristic.write(Util.buildMessage(SET_MQTT_URI, "mqtt://${_username.text}:${_password.text}@${_host.text}:${_port.text}"));
```

##### 3.设置 OTA url

> 参数格式：`0@url`

```dart
cmdCharacteristic.write(Util.buildMessage(SET_OTA_URL, '0@${_ota.text}'));
```

##### 4.Mqtt 订阅
- 发布
- 订阅
- 取消订阅
- 消息通知
```dart

/// 发布
cmdCharacteristic.write(Util.buildMessage(MQTT_PUBLISH, "${_publishQos.text}@${_publishTopic.text}@${_publishMessage.text}"))

/// 订阅
cmdCharacteristic.write(Util.buildMessage(MQTT_SUBSCRIBE, "${_subscribeQos.text}@${_subscribeTopic.text}"));

/// 取消订阅
cmdCharacteristic.write(Util.buildMessage(MQTT_UNSUBSCRIBE, "${_unsubscribeTopic.text}"))

```

注意：**上面的指令下发后，所有响应的操作都在数据处理中心处理**

### 数据处理中心

指令下发处理 + loading销毁 + 实时消息分发

```dart
if (data['code'] == 0) {
  bool isAction =
      Provider.of<BlueToothData>(context, listen: false)
          .isAction;
  switch (data['cmd']) {

    /// -----------> 实时消息更新 <-----------

    case GET_NETWORK_STATE: /// [get wifi status]
      Provider.of<BlueToothData>(context, listen: false)
          .updateWifiData(data['data'].toString().split('@'));
      break;
    case GET_MQTT_CONFIG: /// [get mqtt config]
      Provider.of<BlueToothData>(context, listen: false)
          .updateMqttData(data['data'].toString().split('@'));
      break;
    case GET_VERSION: /// [get version]
      Provider.of<BlueToothData>(context, listen: false)
          .updateVersion(data['data'].toString());
      break;
    case MQTT_SUBSCRIBE_MSG: /// [get mqtt subscribe message]
      print('MQTT_SUBSCRIBE_MSG');
      Provider.of<BlueToothData>(context, listen: false)
          .updateTimer();
      Provider.of<BlueToothData>(context, listen: false)
          .updateNotifyMessage(
              data['data'].toString().split('@'));
      break;
    case GET_STA_OTA: /// [get ota status]
      Provider.of<BlueToothData>(context, listen: false)
          .updateUpgradeStatus('upgraded');

      Future.delayed(Duration(seconds: 5), () {
        if (_isConnenct) widget.device.disconnect();
        FlutterBlue.instance
            .startScan(timeout: Duration(seconds: 4));
        Provider.of<BlueToothData>(context, listen: false)
            .clear();
        Navigator.pushAndRemoveUntil(
          context,
          MaterialPageRoute(
            builder: (BuildContext buildContext) =>
                ScanDevice(),
          ),
          (route) => false,
        );
      });
      break;


    /// -----------> loading销毁 + 指令下发处理 <-----------
    
    case SET_OTA_URL: /// [set ota url]
      if (isAction) {
        BotToast.closeAllLoading();
        Provider.of<BlueToothData>(context, listen: false)
            .updateUpgradeStatus('upgrading');
        Provider.of<BlueToothData>(context, listen: false)
            .updateAction(false);
        Navigator.of(context).push(MaterialPageRoute(
            builder: (BuildContext context) => Upgrade()));
      }
      print('SET_OTA_URL');
      break;
    case MQTT_PUBLISH: /// [set mqtt]
    case MQTT_SUBSCRIBE:
    case MQTT_UNSUBSCRIBE:
      print('MQTT ACTIVE');
      if (isAction) {
        BotToast.closeAllLoading();
        GLToast.message();
        Provider.of<BlueToothData>(context, listen: false)
            .updateAction(false);
      }
      break;
  }
} else {
  if (data['cmd'] == GET_STA_OTA) {
    print('GET_STA_OTA FAILED');
    Provider.of<BlueToothData>(context, listen: false)
        .updateUpgradeStatus('upgradeFailed');
  }
}

```



