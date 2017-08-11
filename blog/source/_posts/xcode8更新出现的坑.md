---
title: xcode8更新出现的坑
categories: iOS
tags: 奇巧淫技
date: 2016-09-14 11:31:49

---

# xcode8注释快捷键失效的问题
1、命令行里输入：

sudo /usr/libexec/xpccachectl

2、重启电脑

# xcode8插件失效问题
## 解决方案

### update：
 [这个命令更简单的样子](https://github.com/inket/update_xcode_plugins)
 
 去签名版本可能无限菊花闪退，解决办法
 系统偏好设置-安全性与隐私-隐私-通讯录  去掉xcode前面的勾
 然后把插件都拿出来，一个个添加排除闪退的插件。
 
  <!--more-->
  
----------------以前的-------------------------------- 

- 1.下载[这个插件](https://github.com/fpg1503/MakeXcodeGr8Again)导出 product（product-archive-选择最后一个）
- 2.退出 Xcode8，同时运行刚刚导出的 MakeXcodeGr8Again，将 Xcode8 拖入其中，等待一段时间(3~10分钟)。
- 3.等菊花转完后，应用程序文件夹下会生成一个 XcodeGr8 的应用，运行命令 sudo xcode-select -s /Applications/XcodeGr8.app/Contents/Developer 将 Xcode 开发路径指向刚生成的 XcodeGr8。
- 4.命令行里输入：
find ~/Library/Application\ Support/Developer/Shared/Xcode/Plug-ins -name Info.plist -maxdepth 3 | xargs -I{} defaults write {} DVTPlugInCompatibilityUUIDs -array-add `defaults read /Applications/XcodeGr8.app/Contents/Info.plist DVTPlugInCompatibilityUUID`

## 可能遇到的问题
- 1.生成了 XcodeGr8 之后，打不开。 解决方法：重启。
- 2.如果之前对其它版本的 Xcode-beat 也有使用这种方式，再对 Xcode8 GM 也是用该方式可能 MakeXcodeGr8Again 这个 APP 会一直闪退。 解决方法：卸载之前生成的 XcodeGr8，再重试。卸载后记得将开发路径重新指回原来的路径，即 sudo xcode-select -s /Applications/Xcode.app/Contents/Developer。如果这种方式还不行，卸载所有版本的 Xcode，然后再安装 GM 版，重复上述步骤。

## 注意
- 1.如果要卸载 XcodeGr8，记得将重新开发路径置回初始状态。

 不要使用 XcodeGr8 打包上传 Appstore，最好使用服务器打包，保证服务器 Xcode 是 Appstore 下载的！！！
