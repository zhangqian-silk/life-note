# PotPlayer

PotPlayer 是一款免费的视频播放器，可以通过加载插件，提升画面质量。

## 安装 LAV Filters

> 项目地址：<https://github.com/Nevcairiel/LAVFilters/releases>

LAV Filter 是一个基于 FFMpeg 开发的开源的音视频解码器，相比 PotPlayer 自带的解码器，功能更加强大。

- 打开 LAV Video Configuration，按照如图所示进行修改：

    ![](images/2024-08-31-17-57-35.png)

## 安装 MADVR

> 官网链接：<https://madvrenvy.com/>
> 下载地址：<https://www.videohelp.com/software/madVR#reviews>

MADVR 是一款视频渲染器，可以用于增加视频画质。

- 下载并解压完成后，以管理员身份运行 `install.bat` 文件，执行安装逻辑

    ![](images/2024-08-31-17-30-35.png)

- 安装完成后，启动 madHcCtrl 程序

    ![](images/2024-08-31-18-17-27.png)

  - 启动后有可能不会弹出面板，可在缩略程序中点击并打开

    ![](images/2024-08-31-18-19-03.png)

- 修改色深为 10 bit

    ![](images/2024-08-31-18-20-19.png)

- 启用去色阶功能

    ![](images/2024-08-31-18-21-53.png)

- 修改刷新率，使其更加平滑

    ![](images/2024-08-31-18-22-38.png)

- 修改算法配置，根据显卡不同，采用不同的策略

    ![](images/2024-08-31-18-25-26.png)

## 安装 PotPlayer

> 官网链接：<http://potplayer.daum.net/>

- 在官网直接下载安装器进行安装即可，安装时，要勾选“安装额外的编解码器”：

    ![](images/2024-08-31-16-31-33.png)

- 安装解码器组件时，勾选所有组件：

    ![](images/2024-08-31-16-35-31.png)

- 打开 PotPlayer 滤镜配置：按照如图所示方式修改：

    ![](images/2024-08-31-17-59-48.png)

- 导入滤镜信息

    ![](images/2024-08-31-18-48-04.png)

- 将所有选项修改为 LAV，无法修改的保持现状

  - 修改源滤镜

    ![](images/2024-08-31-18-49-29.png)

  - 修改视频解码器

    ![](images/2024-08-31-18-50-37.png)

  - 修改音频解码器

    ![](images/2024-08-31-18-51-36.png)

- 导入 LAV Video Decoder、madVR，并设置为强制使用：

    ![](images/2024-08-31-18-33-44.png)

- 修改视频渲染器为 Madshi，即 MadVR

    ![](images/2024-08-31-18-07-50.png)

- 修改音频默认输出设备：

    ![](images/2024-08-31-18-06-12.png)

- 取消规格化

    ![](images/2024-08-31-18-46-09.png)

## 配置方案

- <https://vcb-s.com/archives/7228/comment-page-1>

## 参考

- <https://blog.minz.li/CS/madVR/>
- <https://cloud.tencent.com/developer/article/2134562>
