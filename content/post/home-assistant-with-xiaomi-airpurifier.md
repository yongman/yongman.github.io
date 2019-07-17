---
title: 玩转Home Assistant - 集成小米空气净化器、米家智能家居、broadlink等设备
categories:
  - [生命不息]
date: 2018-11-15 01:27:53
---

Home Assistant是一个开源智能自动化平台，python语言编写，支持多种平台，官方支持树莓派、普通linux主机和docker方式，支持的外围设备和平台也非常多，当然小米智能家居的产品也是支持的。(持续更新中...)

### Docker环境准备

前段时间搞的黑群晖DS918+，放在角落里只用来跑vm软路由和nas心有不甘，加上docker的支持，可以很方便的运行多种服务。
![](https://raw.githubusercontent.com/yongman/i/img/picgo/VVnaj6R.png)

目前已经在跑的docker有：aria2、迅雷远程、人人影视和home assistant。

### Hass环境安装

官方文档的docker安装方式也有专门的群晖nas教程：

1. 群晖上安装docker套件
2. 点击注册表
3. 搜索homeassistant/home-assistant
4. 下载docker镜像
5. 点击镜像，选中后点击启动镜像
6. 填docker实例名
7. 点击高级设置
8. 设置自动启动
9. 网络，使用宿主机网络
10. 添加TZ环境变量
11. 增加卷，挂载/docker/homeassistant到/conf
12. 确认下一步。

### 接入小米空气净化器

1. 查找小米空气净化器地址和token

miio是小米智能家居通信协议的库，这里使用miio来进行局域网发现。

2. 安装miio
```
npm install -g miio

miio <command>

Commands:
  miio configure <idOrIp>                   Control a device by invoking the
                                            given method
  miio control <idOrIp> <method>            Control a device by invoking the
  [params..]                                given method
  miio discover                             Discover devices on the local
                                            network
  miio inspect <idOrIp>                     Inspect a device
  miio protocol <command>                   Inspect and test raw miIO-commands
  miio tokens <command>                     Manage tokens of devices
```

3. 发现

![](https://raw.githubusercontent.com/yongman/i/img/picgo/GSFJzPt.png)

4. 路由器配置mac地址静态地址分配

![](https://raw.githubusercontent.com/yongman/i/img/picgo/BJapq5m.png)

5. 增加homeassistant配置

修改配置文件夹中的configuration.yaml配置，增加：
```
fan:
  - platform: xiaomi_miio
    host: 192.168.31.137
    token: 3bbd8a71d60bf6d507ab54e8c******
    model: zhimi.airpurifier.v6
    name: airpurifier
```

重启home assistant后就可以对小米空气净化器进行控制。

![](https://raw.githubusercontent.com/yongman/i/img/picgo/iuPASEO.png)

![](https://raw.githubusercontent.com/yongman/i/img/picgo/fhSO9RN.png)

这种虽然可以控制，但是空气净化器中的aqi、温度、湿度等信息不能直接的展示，还需要查看fan的属性。

```
# Sensors
sensor:
  - platform: template
    sensors:
      airpurifier_aqi:
        friendly_name: 卧室AQI
        value_template: '{{ states.fan.airpurifier.attributes.aqi }}'
        unit_of_measurement: 'idx'
      airpurifier_temp:
        friendly_name: 卧室温度
        value_template: '{{ states.fan.airpurifier.attributes.temperature }}'
        unit_of_measurement: 'C'
      airpurifier_humidity:
        friendly_name: 卧室湿度
        value_template: '{{ states.fan.airpurifier.attributes.humidity }}'
        unit_of_measurement: '%'
      airpurifier_mode:
        friendly_name: 空净模式
        value_template: '{{ states.fan.airpurifier.attributes.mode }}'
      airpurifier_mode:
        friendly_name: 空净转速
        value_template: '{{ states.fan.airpurifier.attributes.motor_speed }}'
        unit_of_measurement: 'r/m'
```

上面是通过取实体中对应的属性来当作传感器信息，直接显示在主页上。

![](https://raw.githubusercontent.com/yongman/i/img/picgo/deye51x.png)

![](https://raw.githubusercontent.com/yongman/i/img/picgo/YWkP8rb.png)

上面的图中也增加了caiyun的天气信息，最后的几个是小米空气净化器中的数据。

因为家里只有这一个可以接入的设备，所以主页信息很简陋，也没有进行页面展示的定制。

后续：
1. 会尝试接入多种智能家居设备
2. ~~通过homebridge对接home assistant来实现和ios的HomeKit交互~~
3. 通过HAdashboard更形象的展示和进行控制
4. ~~通过传感器数据实现各种自动化~~
5. ~~尝试接入小度音箱(因为我只有一个小度音箱)~~
6. 更多玩法有待探索...(探索中)

==========
**更新**

- 2019-05-28: 很久没有折腾了，已经是稳定状态，上个截图。![](https://raw.githubusercontent.com/yongman/i/img/picgo/20190528090105.png)
- 2019-03-17: HA直接通过小度音箱DLNA和百度的TTS实现了TTS输出，不需要有线或蓝牙连接其他输出设备。不过看到有人说升级了小度的最新固件后即使播放完成，media_player的状态无法自动恢复到idle，[这里](https://github.com/yongman/homeassistant-components/commit/1bd0bfcb3eb0efa94064cb2ceb712ca2d2aa16e0)通过在更新状态的时候判断播放位置是否达到最后来强制结束设备播放状态，可以完成更多的自动化语音提醒功能。
- 2019-03-17: 改装双控墙壁开关，增加客厅和餐厅灯的控制，并接入小度音箱，在没有测电笔的情况下通过试验，搞清楚了双控两路负载线、火线和其他多个灯负载线的分布，并重新排了墙壁开关的顺序。
- 2019-03-06: 支持小度查询房间温湿度，[插件修改]( https://github.com/yongman/homeassistant-dueros/commit/80a200b299ccb944007cdbb830bb7bfc69f5a5f6) ，dueros的文档非常齐全，各种类型设备控制查询接入简单，小度音箱app中的小度训练师，能够实现对话的自然流畅，以你习惯的方式对话，而不是让人机械的适应机器。
- 2019-03-03: HA接入BroadLink rm3 mini，实现对空调和电视机顶盒的红外遥控。![](https://raw.githubusercontent.com/yongman/i/img/picgo/z89K5T6.png)
- 2019-01-01: HA在配置了https的synology camera后疑似存在内存泄露，运行一天，HA占用内存高达1G，修改成http，继续观察。前段时间群晖出现过几次无法响应，高度怀疑是HA内存耗尽的原因。
- 2018-12-16: HA接入BroadLink智能插座和BroadLink万能遥控，智能插座比小米的智能插座更开放，不需要想办法获取token，直接ip和mac地址就可以了，只要能接入HA的设备就能互相联动，比如通过HA实现了小度音箱控制BroadLink的智能插座。
- 2018-12-11: HA接入小度智能音箱，HA的设备可以和小度音箱联动，能动口就绝不动手，完美控制家里的各种电器，可以考虑每个房间一个小度音箱了。
- 2018-11-17: 接入homekit
  homekit的接入非常简单，home assistant已经内置home bridge，直接在配置文件中添加`homekit:`项，即可自动添加homekit支持，不再需要额外安装homebridge和各种配置。

**参考**
1. [Home Assistant](https://www.home-assistant.io/)
2. [Installation on Docker - Home Assistant](https://www.home-assistant.io/docs/installation/docker/)
3. [Template Sensor - Home Assistant](https://www.home-assistant.io/components/sensor.template/)
4. [ha 接入设备汇总 -『HomeAssistant』智能硬件讨论区](https://bbs.hassbian.com/thread-4625-1-1.html)
5. [HomeAssistant联动HomeKit \| 某不科学的博客](https://mou.science/2018/07/22/homeassistant-2/)
6. [HomeKit - Home Assistant](https://www.home-assistant.io/components/homekit/)
7. [小度音箱接入HomeAssistant 采用自带OAuth访问控制](https://bbs.hassbian.com/thread-5417-1-1.html)
8. [GitHub - zhkufish/homeassistant-dueros](https://github.com/zhkufish/homeassistant-dueros)
9. [Memory leak - camera/synology](https://github.com/home-assistant/home-assistant/issues/9352)
10. [微改虫子DLNA，让小度上岗](https://bbs.hassbian.com/thread-4734-1-1.html)
11. [小度使用baidu的tts，输入文字后无法自动播放，必须点播放](https://bbs.hassbian.com/thread-6260-2-1.html)

