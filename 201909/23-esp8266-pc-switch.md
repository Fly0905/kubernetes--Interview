---
title: ESP8266 开关
data: 2019-09-23
---

在办公室希望连接到家里的电脑，内网穿透方案选的是 frp，但家里的电脑在走的时候会忘记打开，亦或者死机蓝屏的时候出现故障无法手动重启。正好手头里有两台路由器，一台 `R6300 V2` 刷了梅林固件，一台 `WNDR 3700 V4` 刷了 `OpenWRT`  ，另外还有一块 `ESP8266 NodeMcu Lua WIFI V3 ESP-12N `的板子。另外还需要一个 3.3 v/5v 的继电器，某宝上 3.9 包邮。

方案图

### 路由器控制 USB 电源开关

其实我第一个想到的方案就是直接通过路由器的 USB 接口控制继电器来控制 PC 主板开关，网上也有文章可以通过 echo 命令控制 USB 电源。Linux 内核官方网也有说明 [Power Management for USB](https://www.kernel.org/doc/Documentation/usb/power-management.txt) 

[Turning USB power on and off](https://openwrt.org/docs/guide-user/hardware/usb.overview) 这个是通过 GPIO 引脚驱动来控制的

```bash
# 打开电源
echo 1 > /sys/class/gpio/gpioN/value
# 关闭电源
echo 0 > /sys/class/gpio/gpioN/value
```

但我的 R6300V2 上没有这些 GPIO 的引脚设备文件

```bash
lede@R6300V2-5501:/tmp/home/root# tree /sys/class/gpio/
/sys/class/gpio/
└── gpio
    ├── dev
    ├── subsystem -> ../../gpio
    └── uevent
lede@R6300V2-5501:/sys/class/gpio/gpio# cat dev
254:0
lede@R6300V2-5501:/sys/class/gpio/gpio# cat uevent
MAJOR=254
MINOR=0
DEVNAME=gpio
```

无功而返，最后在又找到了另外一种通过命令行来控制 USB 电源开关的办法 [Controlling a USB power supply (on/off) with linux](https://stackoverflow.com/questions/4702216/controlling-a-usb-power-supply-on-off-with-linux) 

```bash
# disable external wake-up; do this only once
echo disabled > /sys/bus/usb/devices/usb1/power/wakeup 

echo on > /sys/bus/usb/devices/usb1/power/level       # turn on
echo suspend > /sys/bus/usb/devices/usb1/power/level  # turn off
```

但我的R6300V2路由器里也没有这个文件，无功而返。但在这个 USB 设备文件里却发现了有意思的事情

```bash
lede@R6300V2-5501:/tmp/home/root# ls -al /sys/bus/usb/devices/ | awk '{print $9,$10,$11}'
./
../
1-0:1.0 -> ../../../devices/pci0000:00/0000:00:0b.1/usb1/1-0:1.0/
1-1 -> ../../../devices/pci0000:00/0000:00:0b.1/usb1/1-1/
1-1:1.0 -> ../../../devices/pci0000:00/0000:00:0b.1/usb1/1-1/1-1:1.0/
2-0:1.0 -> ../../../devices/pci0000:00/0000:00:0b.0/usb2/2-0:1.0/
usb1 -> ../../../devices/pci0000:00/0000:00:0b.1/usb1/
usb2 -> ../../../devices/pci0000:00/0000:00:0b.0/usb2/
lede@R6300V2-5501:/tmp/home/root#
```

R6300V2 支持的 USB 设备类型驱动有以下几种

```bash
lede@R6300V2-5501:/sys/bus/usb/drivers# ls
asix/        cdc_mbim/    cdc_wdm/     qmi_wwan/    usb/         usbfs/
cdc_ether/   cdc_ncm/     hub/         rndis_host/  usb-storage/ usblp/
```

虽然 R6300V2 号称有一个 USB3.0 和一个 USB2.0 ，但设备速度还是 480Mbps ，USB2.0 的速度，不明白这是为什么？使用 dd 测了一下速度，

```bash
lede@R6300V2-5501:/sys/devices/pci0000:00/0000:00:0b.1/usb1# cat speed
480 
```

## 还是用吃灰的  ESP8266 😂

尝试了多种办法试着使用路由器的 USB 口当继电器的控制端，均无功而返，看来是不能使用路由器来搞了。正好手里还有一块 `ESP8266 NodeMcu Lua WIFI V3 ESP-12N ` ，大二时好奇就入手了一块搞 WiFi 攻击了😂，玩了几次一直在吃灰了，今天终于派上用场了。

```c
#include <ESP8266WiFi.h>
#include <WiFiClient.h>
#include <ESP8266WebServer.h>

//设置路由器和静态 IP
const char* ssid = "r6300v2";
const char* password = "r6300v2r6300v2";
IPAddress staticIP(192, 168, 0, 230);
IPAddress gateway(192, 168, 0, 1);
IPAddress subnet(255, 255, 255, 0);

// 启动 web 服务器监听端口
ESP8266WebServer server(80);

// 定义相关控制针脚，我使用的 D4 引脚，因为和 3.3v 、GND 三个一块方便固定
const int SW = D4;

void handleon() {
    Serial.println("PC POWER TRUN ON");  
    digitalWrite(D4,LOW);
    server.send(200, "text/html", "PC POWER TRUN ON");
}

void handleoff() {
    Serial.println("PC POWER TRUN OFF");
    digitalWrite(D4,HIGH); //LED off
    server.send(200, "text/html", "PC POWER TRUN OFF");
}    
void setup(void) {
    pinMode(LED, OUTPUT);
    digitalWrite(D4, 0);
    Serial.begin(115200);
    WiFi.begin(ssid, password);
    WiFi.config(staticIP, subnet, gateway);
    WiFi.mode(WIFI_STA); 
    Serial.println("");
    while (WiFi.status() != WL_CONNECTED) {
        delay(500);
       Serial.print(".");
    }
    Serial.println("");
    Serial.print("Connected to ");
    Serial.println(ssid);
    Serial.print("IP address: ");
    Serial.println(WiFi.localIP());

    server.on("/on", handleon);   
    server.on("/off", handleoff);
}

void loop(void) {
   server.handleClient();
}
```

本来第一版写的很复杂，还整了个 HTML  页面，加了个权限认证什么的，杂七杂八的的，最后还是全都去掉了，因为开机的话我习惯于命令行，还是直接在路由器上使用一个 curl 命令就能搞定（已配置好 frp 内网穿透）。另外在路由器上配置好防火墙规则，仅允许路由器本机 IP 访问  ESP8266 的 IP，这样就免去的认证的麻烦，省事儿😂。为了安全起见，就没有把 ESP8266 监听的端口内网穿透到我的服务器。我一般都是通过 frp 连接到我家里的路由器，然后再在上面操作，当然写了别名就更方便了。

