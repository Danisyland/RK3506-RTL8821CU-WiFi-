RK3506 平台 RTL8821CU WiFi 驱动移植与自动联网指南

本教程将指导你完成 RTL8821CU USB WiFi 驱动在 RK3506 (Linux 6.1 内核) 上的交叉编译、版本适配、部署及开机自动联网配置。

---

## 第一阶段：驱动移植与编译 (PC端 - Ubuntu)

**核心目标：** 生成一个能被开发板内核“认可”的 `8821cu.ko` 驱动文件。

### 1. 准备环境 (SDK)
1. **下载 SDK**：从官方渠道获取万象奥科提供的 `RK3506 Linux 6.1 SDK`。
2. **初始化环境**：进入 SDK 根目录，执行以下命令以生成必要的内核头文件：
   ```bash
   ./build.sh lunch  # 选择对应的配置，如：RK3506 EVM NAND
   ./build.sh kernel # 编译内核，这是编译外置驱动的前提
   ```

### 2. 解决“版本魔术数”不匹配 (Version Magic)
若 SDK 内核版本（如 `6.1.118`）与板子实际运行版本（如 `6.1.99`）不一致，直接加载驱动会报错 `invalid module format`。
1. 进入 SDK 的内核目录：`cd kernel/` (或 `kernel-6.1`)。
2. **修改 Makefile**：打开根目录下的 `Makefile`，找到 `SUBLEVEL` 变量，将其值修改为与板子 `uname -r` 一致（例如改为 `99`）。
3. **更新配置**：执行 `make modules_prepare` 更新版本定义。

### 3. 修改驱动 Makefile
进入 RTL8821CU 驱动源码目录，打开 `Makefile` 进行以下关键修改：

#### 第一步：修改平台开关 (Platform Selection)
找到平台配置区域，关闭默认的 PC 平台，开启自定义的 RK3506 平台。
```makefile
# ... 其他平台 ...
CONFIG_PLATFORM_I386_PC = n         # 将 y 改为 n
CONFIG_PLATFORM_ARM_RK3506 = y      # 新增此行并设为 y
CONFIG_PLATFORM_ANDROID_X86 = n
# ... 其他平台 ...
```

#### 第二步：添加编译参数 (Platform Related)
在 `Platform Related` 区域（或搜索 `CONFIG_PLATFORM_I386_PC` 出现第二次的地方）插入以下代码：

```makefile
###################### RK3506 Configuration #######################
ifeq ($(CONFIG_PLATFORM_ARM_RK3506), y)

# 1. 基础宏定义 (必须包含 CFG80211 以支持 wpa_supplicant)
EXTRA_CFLAGS += -DCONFIG_LITTLE_ENDIAN
EXTRA_CFLAGS += -DCONFIG_IOCTL_CFG80211 -DRTW_USE_CFG80211_STA_EVENT
ARCH := arm

# 2. 交叉编译器路径 (请修改为你电脑上的绝对路径，末尾带横杠 -)
CROSS_COMPILE := /home/huhaoxi/RK3506_KEY/RK3506_sdk/rk3506_linux6.1_sdk_v1.2.0/prebuilts/gcc/linux-x86/arm/gcc-arm-10.3-2021.07-x86_64-arm-none-linux-gnueabihf/bin/arm-none-linux-gnueabihf-

# 3. 内核源码路径 (指向刚刚编译过的 SDK 内核目录)
KSRC := /home/huhaoxi/RK3506_KEY/RK3506_sdk/rk3506_linux6.1_sdk_v1.2.0/kernel

endif
###################################################################
```

### 4. 执行编译
1. 在驱动源码目录执行：
   ```bash
   make clean
   make
   ```
2. **产物检查**：只要看到终端出现 `CC [M] ...` 且没有报错，最终目录下生成的 `8821cu.ko` 即为我们要的文件。

---

## 第二阶段：部署与系统配置 (板端 - RK3506)

**核心目标：** 将驱动放入开发板并配置 WiFi 接入点信息。

### 1. 传输与归档
1. 使用 ADB、U盘或 MobaXterm 将 `8821cu.ko` 拷贝到板子上。
2. 建议存放到系统标准模块路径：
   ```bash
   cp 8821cu.ko /lib/modules/$(uname -r)/kernel/drivers/net/wireless/
   depmod -a  # 更新模块依赖，之后可用 modprobe 直接加载
   ```

### 2. 创建 WiFi 配置文件
创建并编辑 `/etc/wpa_supplicant.conf`，写入以下内容：
```conf
ctrl_interface=/var/run/wpa_supplicant
ap_scan=1
update_config=1

network={
    ssid="你的WiFi名称"
    psk="你的WiFi密码"
}
```

---

## 第三阶段：编写自动化脚本 (板端)

**核心目标：** 解决 USB 网卡被误识别为“光驱”的问题，并实现开机自动联网。

### 1. 创建开机启动脚本
创建文件 `/etc/init.d/S99wifi`（`S99` 确保它在系统服务最后阶段启动）：
```bash
#!/bin/sh

# 1. 屏蔽内核繁杂的日志打印 (防止 Turbo EDCA 日志刷屏)
dmesg -n 1

case "$1" in
  start)
    echo "--- Starting Auto WiFi ---"

    # 2. 模式切换：利用 eject 弹出“伪光驱”以触发网卡模式
    if [ -e /dev/sr0 ]; then
        eject /dev/sr0
        sleep 2
    fi

    # 3. 加载驱动
    modprobe 8821cu
    sleep 3

    # 4. 启动网卡
    ifconfig wlan0 up
    
    # 5. 连接 WiFi (清理旧进程并后台运行)
    killall wpa_supplicant 2>/dev/null
    wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant.conf
    
    # 6. 获取 IP (后台获取，不阻塞开机)
    sleep 2
    udhcpc -i wlan0 -b
    ;;
    
  stop)
    killall wpa_supplicant
    ifconfig wlan0 down
    ;;
esac
exit 0
```

### 2. 赋予执行权限
**关键点：** 必须执行此步，脚本才会生效。
```bash
chmod +x /etc/init.d/S99wifi
```

---

## 第四阶段：最终验证

1. **测试**：拨掉开发板网线，输入 `reboot` 重启板子。
2. **观察**：
   - 系统启动后，串口终端应非常干净（无大量 WiFi 调试日志）。
   - 等待约 15-20 秒。
3. **检查**：
   - 输入 `ifconfig wlan0`，确认是否已获得 IP 地址。
   - 输入 `ping www.baidu.com`，验证网络是否连通。

---

## 💡 关键避坑指南回顾
*   **编译器路径**：`CROSS_COMPILE` 必须包含前缀（例如 `arm-none-linux-gnueabihf-`），不能只写到文件夹。
*   **版本对齐**：内核 Makefile 的 `SUBLEVEL` 必须与板子 `uname -r` 的版本号完全对应。
*   **光驱模式**：若没有 `usb_modeswitch` 工具，`eject /dev/sr0` 是处理 RTL8821CU 初始模式的最简有效方案。
*   **日志控制**：`dmesg -n 1` 能极大改善操作体验，否则大量的驱动日志会让你无法在串口输入命令。
