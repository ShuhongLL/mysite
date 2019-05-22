---
title: 如何用VS Code免翻墙听网易云音乐
date: 2019-05-22 16:27:00
tags: [VS Code, NeteaseMusic, 网易云]
photos: ["../images/neteaseMusic.JPG"]
---
在墙外打开网易云音乐发现全是灰色的？ 抱歉您所在区域无法播放？ 该资源暂无版权？ 需要vip？
VS Code**一行script**全搞定～～ <!-- more -->

- 打开VS Code

- 左侧 Extensions 搜索 Netease Music (VSC Netease Music) 或者点[这里](https://marketplace.visualstudio.com/items?itemName=nondanee.vsc-netease-music)

- 点击 Install 后重启 VS Code

- 作者提供了基于 VS Code 自身插件工具 Webview 实现的通过替换 electron 动态链接库翻墙的插件, 有详细的[中文文档](https://marketplace.visualstudio.com/items?itemName=nondanee.vsc-netease-music), 这里 **Unix Shell** 用户（包括 **MacOS**）可以直接在 **Terminal** （任意dir）输入以下script:
```
curl https://gist.githubusercontent.com/nondanee/f157bbbccecfe29e48d87273cd02e213/raw | python
```
  script 输出结果为:
```
% Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  3060  100  3060    0     0   8496      0 --:--:-- --:--:-- --:--:--  8476  
vscode 1.34.0 x64
electron 3.1.8
download well
replace done
remove temp
```

- 替换完毕后打开 VS Code, 上方工具栏 Go -> Go to file... 或者 `Command(⌘) P`, 输入：
```
>NeteaseMusic: Start
```
  等待 editor 跳出 **Please preserve this webview tab** 后就可以使用所有网易云的功能了, 注意这个tab页面必须要保留（不能关闭）

- 使用时直接`Command(⌘) P**`后输入命令即可, 命令都是以 `>NeteaseMusic`起头, 输入 `>ne` 后VS Code会自动跳出并补齐可行指令, 甚至还可以登陆收藏查看评论