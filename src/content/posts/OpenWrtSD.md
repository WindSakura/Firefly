
---
title: OpenWrt - SDCard 分区扩容
published: 2024-09-12
pinned: false
description: 本篇文章将介绍扩容 OpenWrt root 分区的内容，本文所述内容适用于以 SD 卡存储介质的 ARM 设备 (树莓派、NanoPi R2S 等)。
tags: [OpenWrt, 扩容]
draft: false
---

# OpenWrt - SDCard 分区扩容

![F1AOijqaQAARoTH](https://cdn.jsdelivr.net/gh/WindSakura/imgs@main/imgsF1AOijqaQAARoTH.jpg)

https://web.archive.org/web/20230323181845/https://mlapp.cn/1011.html 怎么有网页似了啊

本篇文章将介绍扩容 OpenWrt root 分区的内容，**本文所述内容适用于以 SD 卡存储介质的 ARM 设备 (树莓派、NanoPi R2S 等)。**

将本项目 OpenWrt 固件刷入 SD 卡后，SD 卡内 boot 分区所占空间为 64M，root 分区所占空间为 960M，所以:

SD 卡总容量 - (64+960)M ≈ 空闲分区容量

虽然刷入固件的初始状态下空闲分区无法被利用，但在默认情况下，960M 的 root 分区完全能胜任日常使用。

若确有扩容 root 分区的需求，请按下文内容操作。

注意:

⚠️数据无价，在扩容操作前，**请务必备份好 SD 卡内的重要数据**。

⚠️如确有扩容需求，**请尽量在刷入固件后即进行分区扩容操作**，这样不仅可以避免文件丢失 (因为刚刷完固件什么重要文件都没有)，在一定程度上还可加快分区扩容速度。



## Squashfs
对于 squashfs 固件，我们可以在暂未使用的空闲空间上新建一个分区，之后将 overlay 分区中的内容拷贝到这个分区，然后将系统在 overlay 分区的挂载点修改为刚刚新建的分区来进行扩容。

⚠️由于 Squashfs 固件涉及到文件迁移，所以 请尽量在刷入固件后即进行分区扩容操作。

1. 在“系统 - 软件包”中查看 rootfs 剩余空间为 600M 

   ![](https://cdn.jsdelivr.net/gh/WindSakura/imgs@main/imgsimgsimage-20241113154531178.png)

2. 在“系统 - 磁盘管理”中找到 SD 卡设备 (/dev/mmcblk0)，点击“修改” »

   ![image-20241113154553211](https://cdn.jsdelivr.net/gh/WindSakura/imgs@main/imgsimage-20241113154553211.png)

   ![image-20241113154604654](https://cdn.jsdelivr.net/gh/WindSakura/imgs@main/imgsimage-20241113154604654.png)

   

3. 在分区信息中可以看出 SD 卡中有 14.83G 的空闲空间，点击右侧“新建”按钮新建分区 »

   ![image-20241113154614512](https://cdn.jsdelivr.net/gh/WindSakura/imgs@main/imgsimage-20241113154614512.png)

4. 分区新建完成，点击“格式化” »

   ![image-20241113154625315](https://cdn.jsdelivr.net/gh/WindSakura/imgs@main/imgsimage-20241113154625315.png)

5. 选择 ext4 分区作为新分区的文件系统 »

   ![image-20241113154637025](https://cdn.jsdelivr.net/gh/WindSakura/imgs@main/imgsimage-20241113154637025.png)

6. 分区已成功格式化为 ext4 格式 »

   ![image-20241113154647577](https://cdn.jsdelivr.net/gh/WindSakura/imgs@main/imgsimage-20241113154647577.png)

7. 进入 OpenWrt 的 TTYD 或 SSH，进行迁移文件操作 »

   ```
   # 将刚刚新建的 /dev/mmcblk0p3 分区挂载至 /mnt
   root@OpenWrt:/# mount /dev/mmcblk0p3 /mnt
   
   # 查看分区挂载情况
   root@OpenWrt:/# lsblk
   NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
   loop0         7:0    0 760.8M  0 loop /overlay
   mmcblk0     179:0    0  14.9G  0 disk 
   ├─mmcblk0p1 179:1    0    64M  0 part /boot
   ├─mmcblk0p2 179:2    0   960M  0 part /rom
   └─mmcblk0p3 179:3    0  13.8G  0 part /mnt
   
   # 将 /overlay 分区下的所有文件拷贝至刚刚建立好的分区内
   root@OpenWrt:/# cp -f -a /overlay/. /mnt
   
   # 查看是否拷贝成功
   root@OpenWrt:/# ls -a /mnt
   .           ..          .fs_state   lost+found  upper       work
   root@OpenWrt:/# ls -a /overlay
   .          ..         .fs_state  upper      work
   
   # 同步文件
   root@OpenWrt:/# sync
   
   # 卸载 /dev/mmcblk0p3 分区
   root@OpenWrt:/# umount /mnt
   ```

   

8. 前往“系统 - 挂载点”，点击“生成配置” »

   ![image-20241113154706143](https://cdn.jsdelivr.net/gh/WindSakura/imgs@main/imgsimage-20241113154706143.png)

9. 在“挂载点”中我们可以看到刚刚创建好的 ext4 分区 /dev/mmcblk0p3，点击右方“修改” »

   ![image-20241113154716766](https://cdn.jsdelivr.net/gh/WindSakura/imgs@main/imgsimage-20241113154716766.png)

10. 在接下来的界面中，“启用此挂载点”并选择“作为外部 overlay 使用”，点击“保存&应用” »

    ![image-20241113154726231](https://cdn.jsdelivr.net/gh/WindSakura/imgs@main/imgsimage-20241113154726231.png)

11. 在“系统 - 挂载点”页面下，确认挂载点已启用 (打钩)，并确认挂载点为 /overlay，点击下方“保存&应用”，之后重启 OpenWrt »

    ![image-20241113154740333](https://cdn.jsdelivr.net/gh/WindSakura/imgs@main/imgsimage-20241113154740333.png)

12. 验证分区扩容成功 »

    ![image-20241113154748805](https://cdn.jsdelivr.net/gh/WindSakura/imgs@main/imgsimage-20241113154748805.png)

    ```
    root@OpenWrt:/# lsblk
    NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
    loop0         7:0    0 760.8M  0 loop /mnt/loop0
    mmcblk0     179:0    0  14.9G  0 disk 
    ├─mmcblk0p1 179:1    0    64M  0 part /boot
    ├─mmcblk0p2 179:2    0   960M  0 part /rom
    └─mmcblk0p3 179:3    0  13.8G  0 part /overlay
    
    root@OpenWrt:/# mount | grep overlay
    /dev/mmcblk0p3 on /overlay type ext4 (rw,relatime)
    overlayfs:/overlay on / type overlay (rw,noatime,lowerdir=/,upperdir=/overlay/upper,workdir=/overlay/work)
    overlayfs:/overlay on /opt/docker type overlay (rw,noatime,lowerdir=/,upperdir=/overlay/upper,workdir=/overlay/work)
    
    root@OpenWrt:/# df -h | grep overlay
    /dev/mmcblk0p3           13.5G     42.2M     12.8G   0% /overlay
    overlayfs:/overlay       13.5G     42.2M     12.8G   0% /
    overlayfs:/overlay       13.5G     42.2M     12.8G   0% /opt/docker
    ```

    