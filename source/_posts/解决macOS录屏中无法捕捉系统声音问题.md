---
title: 解决macOS录屏中无法捕捉系统声音问题
mathjax: false
date: 2022-05-27 15:02:38
categories: MacOS
---

本篇文章是关于解决 macOS Monterey 中 QuickTime Player 录屏无法捕捉系统声音问题的解决方案。 

所需工具：[BlackHole](https://github.com/ExistentialAudio/BlackHole)

<!-- more -->

## 录屏软件 

参考[官方链接](https://support.apple.com/zh-cn/HT208721)使用系统内置的 QuickTime Player 为录屏软件，可以同时按下 Shift、Command 和 5 呼出录屏界面。

![录屏演示一](https://pico-1258741719.cos.ap-shanghai.myqcloud.com/blog/解决macOS录屏中无法捕捉系统声音问题/%E5%BD%95%E5%B1%8F%E6%BC%94%E7%A4%BA%E4%B8%80.jpeg)

点击录制按钮后，即可开始录制，退出录制则在上方菜单栏按下停止按钮或同时按下 Command、Control、Esc。

## 声音插件 

### BlackHole 的下载

直接使用 QuickTime Player 录制的内容是不包括系统声音的，只可以捕捉到麦克风的声音。也有方案选择声音插件 [SoundFlower](https://github.com/mattingalls/Soundflower)，但此插件不支持 m1 芯片的 macOS。因此寻找替代方案 [BlackHole](https://github.com/ExistentialAudio/BlackHole)。

参照 [BlackHole](https://github.com/ExistentialAudio/BlackHole)，在终端中输入命令 `brew install blackhole-16ch` 或者 `brew install blackhole-2ch` 即可完成插件 BlackHole 的下载，下载途中可能要输入用户密码。

```shell
brew install blackhole-16ch
brew install blackhole-2ch
```

注：16ch 和 2ch 指的是声道的多少，一般场景下 2ch 已经足够使用，选择上述命令中的一个即可使用，笔者使用的是 2ch。

![BlackHole的安装](https://pico-1258741719.cos.ap-shanghai.myqcloud.com/blog/解决macOS录屏中无法捕捉系统声音问题/BlackHole%E7%9A%84%E5%AE%89%E8%A3%85.png)

### BlackHole 的设置

Step 1：按住 command 和 空格，搜索 midi，打开“音频 midi 设置”。

![音频midi设置](https://pico-1258741719.cos.ap-shanghai.myqcloud.com/blog/解决macOS录屏中无法捕捉系统声音问题/%E9%9F%B3%E9%A2%91midi%E8%AE%BE%E7%BD%AE.png)

Step 2：点击左下角“+”，新建一个“聚集设备”，先后勾选右边的“内置麦克风”和“BlackHole 2ch”，并重命名该聚集设备为 “QuickTime Player Input”。

![新建聚集设备](https://pico-1258741719.cos.ap-shanghai.myqcloud.com/blog/解决macOS录屏中无法捕捉系统声音问题/%E6%96%B0%E5%BB%BA%E8%81%9A%E9%9B%86%E8%AE%BE%E5%A4%87.png)

Step3：点击左下角“+”，新建一个“多输出设备”，先后勾选右边的“内建输出”和“BlackHole 2ch”，并重命名该多输出设备为 “Screen Record w/ Audio”，将“内建输出”的右侧选项勾选。

注：如果这里用户使用的耳机，那么将 “内建输出”替换为耳机选项。

![新建多输出设备](https://pico-1258741719.cos.ap-shanghai.myqcloud.com/blog/解决macOS录屏中无法捕捉系统声音问题/%E6%96%B0%E5%BB%BA%E5%A4%9A%E8%BE%93%E5%87%BA%E8%AE%BE%E5%A4%87.png)

Step 4： 按住 command 和 空格，搜索 sound，打开“声音设置”。

![声音设置1](https://pico-1258741719.cos.ap-shanghai.myqcloud.com/blog/解决macOS录屏中无法捕捉系统声音问题/%E5%A3%B0%E9%9F%B3%E8%AE%BE%E7%BD%AE1.png)

Step 5：在输出中选择刚刚新建立的“多输出设备”。

![声音设置2](https://pico-1258741719.cos.ap-shanghai.myqcloud.com/blog/解决macOS录屏中无法捕捉系统声音问题/%E5%A3%B0%E9%9F%B3%E8%AE%BE%E7%BD%AE2.png)

Step 6：同时按下 Shift、Command 和 5 呼出录屏界面，在“设置”中，“麦克风”选项下选择之前新建的“聚集设备”。

![录屏设置](https://pico-1258741719.cos.ap-shanghai.myqcloud.com/blog/解决macOS录屏中无法捕捉系统声音问题/%E5%BD%95%E5%B1%8F%E8%AE%BE%E7%BD%AE.jpeg)

至此，安装过程已经完成。此后用 QuickTime Player 录制的视频文件将会带有系统内录的声音。

### BlackHole 的卸载

Step 1：按住 command 和 空格，搜索 midi，打开“音频 midi 设置”。

Step 2：选择之前新建的“聚集设备”，点击左下角“-”；选择之前新建的“多输出设备”，点击左下角“-”。

Step 3：根据之前下载的 BlackHole 版本，在终端输入命令 `sudo rm -R /Library/Audio/Plug-Ins/HAL/BlackHole2ch.driver` 或者 `sudo rm -R /Library/Audio/Plug-Ins/HAL/BlackHole16ch.driver`。需要输入用户密码。

```shell
sudo rm -R /Library/Audio/Plug-Ins/HAL/BlackHole2ch.driver
sudo rm -R /Library/Audio/Plug-Ins/HAL/BlackHole16ch.driver
```

Step 4：在终端输入命令 `sudo launchctl kickstart -kp system/com.apple.audio.coreaudiod` 重启系统音频服务。

```shell
sudo launchctl kickstart -kp system/com.apple.audio.coreaudiod
```

## 一些问题

安装完 BlackHole，使用多输出设备作为音频输出后，无法调节音量。建议是在非录屏情况下，去“声音设置”中“输出”依旧选择默认输出设备，录屏时再使用多输出设备。

## Reference

1. https://zhuanlan.zhihu.com/p/343145329
2. https://zhuanlan.zhihu.com/p/87715078
3. https://github.com/ExistentialAudio/BlackHole