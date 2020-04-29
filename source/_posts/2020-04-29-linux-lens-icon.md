---
title: 给 Kubernetes Lens AppImage 添加 Desktop Entry
excerpt: ...
categories: 笔记
toc: true
date: 2020-04-29 22:13:02
tags: [Linux]
---

<!-- toc -->

本文记录如何为 Kuberntes Lens 的 Linux 版本的 AppImage 手动创建一个 Desktop Entry 以及生成 Icon

## 手动创建 Desktop Entry
创建一个 `desktop` 文件，任意位置均可。假设名字为 `lens.desktop`:

```ini
[Desktop Entry]
Name=Lens
Comment=A full-featured Kubernetes IDE
Exec="/home/yayu/Soft/AppImage/Lens-3.3.1.AppImage" %U
Terminal=false
Type=Application
Icon=kube-lens
StartupWMClass=Lens
X-AppImage-Version=1.4.1.271
Categories=Network;
X-AppImage-BuildId=1N3OgzauYTeCvM55WzKgL7MIQe0
X-Desktop-File-Install-Version=0.24
X-AppImage-Comment=Create By HangYan
TryExec=/home/yayu/Soft/AppImage/Lens-3.3.1.AppImage
```

其中 `Exec` 以及 `TryExec` 是 AppImage 的位置，应相应更改

## 生成 ICON

这里我下载了一个黑色的 Kubernetes ICON, 背景透明，webp 格式

![](/images/kube/kube.png)

这里我已经转为 png 格式，转换命令为:

```bash
dwebp <origin>.webp -o kube.png
```

然后，生成相应的 Icon, 并安装到系统中

```bash
xdg-icon-resource install --theme Moka --size 48 kube.png kube-lens
```
安装的 icon 可以在 `~/.icons` 目录中看到。


其中:
* `theme`: 即将此 icon 安装到哪个 themes 中. Linux Desktop 基本上都有可以设置 icon themes 的地方
* `size`: 一般是 panel 上 icon 的 size, 这个在一般的 panel 的设置里也能看到
* `kube-lens`: 即最终生成的 icon 的名字，也就是上面 desktop 文件中引用的 icon 名字


## 安装 Desktop Entry

```bash
# 先做一下校验
desktop-file-validate lens.desktop

desktop-file-install lens.desktop --dir=~/.local/share/applications

# 更新 desktop entry
update-desktop-database ~/.local/share/applications
```


最终效果如图:

![](/images/shot/lens.png)






