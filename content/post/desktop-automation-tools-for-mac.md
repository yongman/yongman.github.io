---
title: Mac osx让工作更有效率的桌面控制管理工具
date: 2018-07-18 10:05:46
categories: ["生产力"]
---

## 1. Hammerspoon-高效操作窗口和鼠标
自己在工作中一直在使用`mac`，并且外接显示器多屏幕，一直在找可以快速将鼠标焦点移动到另外屏幕的工具，之前搜了半天没找到就放弃了。
但是用`trackpad`将鼠标在两个屏幕中拖来拖去，手指都酸了。今天发现了一个`Hammerspoon`工具，通过自定义lua脚本可以实现
- 窗口复杂的移动，可以指定移动的坐标
- 窗口大小调整
- 多窗口排列
- 监控响应多种事件
- 鼠标控制
- ...其他暂时没有需求

下面是我使用的lua配置，实现了窗口移动和大小控制，鼠标快速切换
```
-- 一般组合键为 Shift + Command + ?
local hyper = {'shift', 'cmd'}

-- 最大化窗口
-- 快捷键为 Shift + Command + ↑
hs.hotkey.bind(hyper, 'up', function()
    hs.grid.maximizeWindow()
end)

-- 让窗口占据左半边（Windows 下面的向左贴边停靠）
-- 快捷键为 Shift + Command + ←
hs.hotkey.bind(hyper, "Left", function()
  local win = hs.window.focusedWindow()
    local f = win:frame()
    local screen = win:screen()
    local max = screen:frame()

    f.x = max.x
    f.y = max.y
    f.w = max.w / 2
    f.h = max.h
    win:setFrame(f)
end)

-- 让窗口占据右半边（Windows 下面的向左贴边停靠）
-- 快捷键为 Shift + Command + ->
hs.hotkey.bind(hyper, "Right", function()
  local win = hs.window.focusedWindow()
    local f = win:frame()
    local screen = win:screen()
    local max = screen:frame()

    f.x = max.x + max.w / 2
    f.y = max.y
    f.w = max.w / 2
    f.h = max.h
    win:setFrame(f)
end)

-- 移动鼠标第n个屏幕
-- 快捷键为 Shift + Command + `
hs.hotkey.bind(hyper, '`', function()
    local screen = hs.mouse.getCurrentScreen()
    local nextScreen = screen:next()
    local rect = nextScreen:fullFrame()
    local center = hs.geometry.rectMidPoint(rect)
 
    hs.mouse.setAbsolutePosition(center)
end)

-- 将当前窗口移动到第 n 个屏幕
-- 并最大化窗口
-- 快捷键为 Ctrl + Command + 屏幕数字
local hyper2 = {'ctrl', 'cmd'}
moveto = function(win, n)
  local screens = hs.screen.allScreens()
  if n > #screens then
    hs.alert.show("No enough screens " .. #screens)
  else
    local toWin = hs.screen.allScreens()[n]:name()
    hs.layout.apply({{nil, win:title(), toWin, hs.layout.maximized, nil, nil}})
  end
end

hs.hotkey.bind(hyper2, "1", function()
  local win = hs.window.focusedWindow()
  moveto(win, 1)
end)
hs.hotkey.bind(hyper2, "2", function()
  local win = hs.window.focusedWindow()
  moveto(win, 2)
end)

```

**更新**
发现一个更完整的配置，窗口管理更完善，代码风格更好。[https://github.com/S1ngS1ng/HammerSpoon](https://github.com/S1ngS1ng/HammerSpoon)

## 2. Manico-快速切换或打开app

>Manico 是一个非常容易使用的 App，您可以在无需任何配置和学习就可以立刻开始使用它。您所做的仅仅是，当您需要切换至一个 App 时，按住 Option 键和对应的目标数字或字母，您将被立马切换至对应的 App。如果这个 App 还没有启动，那么 Manico 会先启动它。
>
>Manico 的主意最初来自于 Ubuntu Unity 桌面，如果您曾经使用过 Unity 桌面并且习惯于 Super 键切换 App，并且想要在 macOS 里有同样的功能，那么 Manico 就是为您设计的。这也是我最初开发 Manico 的目标，但是它不仅仅是如此。
>
>在我们的日常电脑使用中，我们每个人都有一些每天都会使用的 App，比如：Finder，Safari（或其他浏览器）或 Terminal。切换并使用他们应该是很直接的，传统的「CMD + Tab」的切换方式应该只用于切换那些使用频率不高的 App。



## 参考
[Hammerspoon - Getting Started](http://www.hammerspoon.org/go/)
[Mac 多显示器快速移动鼠标 - 简书](https://www.jianshu.com/p/3d62c18c0c78)
[有没有快速把鼠标移到下一个屏幕窗口的方法? - V2EX](https://www.v2ex.com/t/187697)
[Manico - macOS 下的快速 App 切换器](https://manico.im/)
