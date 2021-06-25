# ADB常用命令

## 设备

adb devices ：列出所有已连接的设备 （设备号）

adb -s [设备号] install xxx.apk：指定设备安装apk

## wifi

通过wifi连接设备：

adb tcpip 5555：启动端口

adb connect [设备ip]:5555：通过wifi连接设备

adb disconnect：断开所有设备

adb disconnect <设备的IP地址>:5555：断开指定设备

