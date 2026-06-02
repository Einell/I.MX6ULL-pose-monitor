
---

# 基于 i.MX6ULL 的姿态监测与可视化系统 (Posture Monitor)

本项目是一个运行在正点原子 i.MX6ULL 开发板的姿态监测与可视化终端。项目通过 SPI 总线高效采集 ICM20608 传感器的 6 轴原始数据，在应用层使用 **POSIX 多线程** 与 **互补滤波算法** 进行精准的姿态解算（Pitch/Roll），并通过**LVGL v9** 图形引擎在 LCD 屏幕上显示仪表盘及数据看板。

---

##  环境说明

### 硬件环境

* **开发板：** 正点原子 i.MX6ULL 开发板 (Alpha EMMC 版本)
* **核心传感器：** ICM20608 (6轴 IMU 芯片)，采用 SPI 总线接口通信
* **显示设备：** 7 寸 RGB LCD 液晶屏 (默认分辨率 `1024x600`)

### 软件环境

* **操作系统内核：** 移植版 Linux Kernel (版本：`4.1.15`)
* **根文件系统：** 基于 BusyBox 构建 Rootfs
* **应用层图形库：** LVGL v9
* **工程构建工具：** CMake

---

##  系统移植与设备树配置

本项目的底层系统基础主要参考《正点原子 i.MX6U  嵌入式系统驱动开发指南系统移植篇》相关文档进行内核裁剪与基础移植。

### 1. 修改板级设备树

打开内核源码路径下的设备树文件：`arch/arm/boot/dts/imx6ull-alientek-emmc.dts`。

#### ① 添加 7 寸 LCD 节点配置

找到或追加 `&lcdif` 节点，修改为以下适配 7 寸 RGB LCD (1024x600) 的标准硬件时序参数：

```dts
&lcdif {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_lcdif_dat
                 &pinctrl_lcdif_ctrl>;
    display = <&display0>;
    status = "okay";

    display0: display {
        bits-per-pixel = <16>;
        bus-width = <24>;

        display-timings {
            native-mode = <&timing0>;
            timing0: timing0 {
                clock-frequency = <51200000>; /* 像素时钟 51.2 MHz */
                hactive = <1024>;             /* 水平显示区域 1024 */
                vactive = <600>;              /* 垂直显示区域 600 */
                hfront-porch = <160>;         /* HFP */
                hback-porch = <140>;          /* HBP */
                hsync-len = <20>;             /* HSPW */
                vback-porch = <20>;           /* VBP */
                vfront-porch = <12>;          /* VFP */
                vsync-len = <3>;              /* VSPW */

                hsync-active = <0>;
                vsync-active = <0>;
                de-active = <1>;
                pixelclk-active = <0>;
            };
        };
    };
};

```

#### ② 添加 SPI 引脚复用与子节点配置

在 `&iomuxc` 节点和外设总线中，配置 `ECSPI3` 对应的引脚复用，并将 `icm20608` 作为子外设进行挂载。

* **引脚复用配置 (`&iomuxc`)：**

```dts
&iomuxc {
    imx6ul-evk {
        pinctrl_ecspi3: icm20608 {
            fsl,pins = <
                MX6UL_PAD_UART2_TX_DATA__GPIO1_IO20    0x10b0 /* CS 片选，作为普通GPIO */
                MX6UL_PAD_UART2_RX_DATA__ECSPI3_SCLK   0x10b1 /* SCLK 时钟 */
                MX6UL_PAD_UART2_RTS_B__ECSPI3_MISO     0x10b1 /* MISO 输入 */
                MX6UL_PAD_UART2_CTS_B__ECSPI3_MOSI     0x10b1 /* MOSI 输出 */
            >;
        };
    };
};

```

* **SPI控制器配置 (`&ecspi3`)：**

```dts
&ecspi3 {
    fsl,spi-num-chipselects = <1>;
    cs-gpios = <&gpio1 20 GPIO_ACTIVE_LOW>; 
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_ecspi3>;
    status = "okay";

    icm20608@0 {
        compatible = "alientek,icm20608";
        spi-max-frequency = <8000000>;
        reg = <0>;
    };
};

```

### 2. 编译设备树

在内核源码根目录下执行命令编译生成新的设备树二进制 `.dtb` 文件：

```bash
make dtbs

```

### 3. 固件部署

* **烧录部署：** 生产或固化阶段可使用 NXP 官方工具 `mfgtools` 将 U-Boot、zImage、`.dtb` 以及根文件系统一键烧录至板载 EMMC。
* **开发联调：** 通过 Ubuntu 宿主机搭建环境，使用 `TFTP` 引导挂载 `zImage` 与 `dtb` 设备树，并通过 `NFS` 挂载根文件系统，具体可参考《【正点原子】I.MX6U网络环境TFTP&NFS搭建手册V1.3.2》。

---

##  运行与测试指南

### 1. 克隆本仓库至根目录
```git clone https://github.com/Einell/I.MX6ULL-pose-monitor.git```

### 2. 克隆lvgl仓库至项目目录pose_monitor下
```git clone https://github.com/lvgl/lvgl.git```

### 3. 加载底层驱动模块

将本项目源码中 `driver/` 目录下编译生成的驱动文件 `icm20608.ko`拷贝至开发板根文件系统/lib/modules/4.1.15/下：

```bash
# 在开发板终端执行
cd /lib/modules/4.1.15/
depmod
modprobe icm20608.ko
```

*执行 `ls /dev/icm20608`，若生成对应字符设备节点，则说明驱动匹配并加载成功。*

### 4. 交叉编译应用层工程

进入本项目pose_monitor/build/，清理旧的缓存并利用 CMake 构建：

```bash
cd "pose monitoring"
rm -rf *
cmake .. -DCMAKE_TOOLCHAIN_FILE=../arm-linux.cmake
make -j4

```

编译成功后，将在当前 `build` 目录下生成可执行二进制目标文件：`pose_demo`。

### 5. 开发板启动与可视化

将生成的 `pose_demo` 程序发送至开发板用户目录下，添加运行权限，清除显存，启动：

```bash
chmod +x pose_demo
echo 0 > /sys/class/vtconsole/vtcon1/bind 2>/dev/null
echo -e "\033[?25l" > /dev/tty0 2>/dev/null
echo -e "\033[?25l" > /dev/tty1 2>/dev/null
dd if=/dev/zero of=/dev/fb0 bs=1024 count=1200
./pose_demo
```

**预期运行效果：** 
1. **lcd屏效果：** 7 寸 LCD 显示屏会显示数字姿态仪表盘。当用手拿起开发板进行倾斜、摇晃等三维动作时，屏幕左侧的 Pitch（俯仰角）柱状指示条会平移，右侧的 Roll（横滚角）金色圆弧表盘会旋转，中央的地平线会随之移动。
![lcd显示](img/pose.gif)
2. **串口终端输出：** 控制台终端在程序进入大循环后，会在最底端原地非滚屏式刷新物理角度值。
![串口输出](img/image.png)
---

##  屏幕适配与尺寸变更指南 (针对 4.3 寸 LCD)

本项目默认针对 **7 寸 RGB LCD (1024x600)** 。如果您手里使用的是正点原子 **4.3 寸屏幕 (分辨率通常为 800x480 或 480x272)**，请按照以下三个步骤对源码进行修改：

### 1. 内核设备树修改

在 `imx6ull-alientek-emmc.dts` 中找到上述的 `&lcdif` 节点配置。将其内部 `timing0` 的像素时钟、显示区域参数修改为 4.3 寸屏对应的官方硬件时序参数（如 `hactive = <800>; vactive = <480>;`），保存后重新通过命令 `make dtbs` 编译并替换开发板所加载的 `.dtb` 文件。

### 2. 应用程序分辨率修改

用 VS Code 打开应用层工程中的主文件 `pose_dashboard.c`。在代码顶部找到物理画布分辨率的宏定义区，将 7 寸的分辨率修改为你使用的 4.3 寸实际分辨率：

```c
/*以 800x480 为例 */
#define SCREEN_WIDTH  800
#define SCREEN_HEIGHT 480

```

### 3. UI 布局局部微调

由于屏幕尺寸和物理像素降低，如果对显示比例有更高要求，可以对应用代码中创建控件的尺寸进行等比例缩小。例如找到构建圆弧的函数：

```c
/* 如果在小屏下圆弧偏大，可将原先 300x300 的尺寸局部缩小（例如改为 220x220） */
lv_obj_set_size(roll_arc, 220, 220);

```

### 4. 重新编译下发

完成上述调整后，重新在 Ubuntu 构建目录下依次调用 `cmake .. -DCMAKE_TOOLCHAIN_FILE=../arm-linux.cmake` 和 `make -j4` 编译生成新的 `pose_demo`。重新传输至板子上运行。

---