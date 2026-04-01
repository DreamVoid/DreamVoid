---
title: 使用 postmarketOS 打造便携服务器（4）：限制充电电量，保护电池
date: 2026-03-31 10:42:25
tags: 
- Linux
- postmarketOS
categories: 技术
---

> 当使用旧智能手机作为永久服务器且始终供电时，保持电池电压最高并不理想。
> 
> 本指南展示了如何调节电池的最大电压。这包括反编译设备树，找到并改变最大电压，然后重新编译。
> 
> 这听起来很吓人，但其实非常简单，而且可以直接在运行PostmarketOS的设备上完成

我觉得[官方指南](https://wiki.postmarketos.org/wiki/Battery_Voltage_Reduction)说的很不错，就直接复制了。为了防止电池鼓包，我也需要限制一下一加 6 的充电。

# 官方指南方式
以下是参考官方指南写的，先说结论：**对一加 6 无效**。但我不想白写，因此留在这里。

## 尝试直接写入电压值
首先找到电压设置，然后试着读取值：
```bash
OnePlus-6:~# ls -l /sys/class/power_supply/*/voltage*
-r--r--r--    1 root     root          4096 Mar 30 13:45 /sys/class/power_supply/bq27411-0/voltage_max_design
-r--r--r--    1 root     root          4096 Mar 30 13:45 /sys/class/power_supply/bq27411-0/voltage_min_design
-r--r--r--    1 root     root          4096 Mar 30 13:45 /sys/class/power_supply/bq27411-0/voltage_now
-r--r--r--    1 root     root          4096 Mar 31 10:38 /sys/class/power_supply/pmi8998-charger/voltage_now
OnePlus-6:~# cat /sys/class/power_supply/bq27411-0/voltage_now
4394000
OnePlus-6:~# cat /sys/class/power_supply/pmi8998-charger/voltage_now
4785156
OnePlus-6:~# cat /sys/class/power_supply/bq27411-0/voltage_max_design
4400000
```

很明显 `/sys/class/power_supply/bq27411-0/voltage_max_design` 就是要修改的，但真的能修改吗？试一下：
```bash
OnePlus-6:~# echo 4400000 > /sys/class/power_supply/bq27411-0/voltage_max_design
-bash: /sys/class/power_supply/bq27411-0/voltage_max_design: Permission denied
```

不行。那只能和官方指南一样反编译设备树了。

## 修改设备树
### 反编译
使用软件包管理器安装设备树编译器：
```bash
apk add dtc
```

一加 6 的设备树文件是 `/boot/sdm845-oneplus-enchilada.dtb`，备份一下：
```bash
cp /boot/sdm845-oneplus-enchilada.dtb /boot/sdm845-oneplus-enchilada.dtb.bak
```

反编译到用户目录：
```bash
dtc -I dtb -O dts /boot/sdm845-oneplus-enchilada.dtb -o ~/enchilada.dts
```

反编译的时候出现了一堆警告，不要害怕不要慌，继续看我们要的设置在哪：
```bash
OnePlus-6:~# grep -n "voltage-max-design" ~/enchilada.dts 
6800:		voltage-max-design-microvolt = <0x432380>;
```

6800 是需要修改的行号，官方指南使用纯命令来修改，不过我决定使用文本编辑器来修改，这样也能看到我修改了什么。

### 修改文件
使用 Visual Studio Code 打开设备树文件，转到 6800 行。我看到 6796 行到 6803 行是这样的：
```
	battery {
		compatible = "simple-battery";
		charge-full-design-microamp-hours = <0x325aa0>;
		voltage-min-design-microvolt = <0x33e140>;
		voltage-max-design-microvolt = <0x432380>;
		constant-charge-current-max-microamp = <0x1b7740>;
		phandle = <0x67>;
	};
```

接下来直接把 `0x432380` 改为 `0x39f880`，即官方指南的 3.8 伏。如果要其他电压值，官方指南提供了转换教程，手动转换即可。

用命令行验证一下修改：
```bash
OnePlus-6:~# grep -n -A 2 -B 2 "voltage-max-design-microvolt" ~/enchilada.dts 
6798-		charge-full-design-microamp-hours = <0x325aa0>;
6799-		voltage-min-design-microvolt = <0x33e140>;
6800:		voltage-max-design-microvolt = <0x39f880>;
6801-		constant-charge-current-max-microamp = <0x1b7740>;
6802-		phandle = <0x67>;
```

### 重新编译并安装

用 dtc 重新编译：
```bash
dtc -I dts -O dtb ~/enchilada.dts -o ~/enchilada-modified.dtb
```

也是出现了一堆警告，不要害怕不要慌，继续安装设备树：
```bash
cp ~/enchilada-modified.dtb /boot/sdm845-oneplus-enchilada.dtb
```

如果没有问题，直接 `reboot` 重启，然后等一会连接上后，验证修改是否成功：
```
OnePlus-6:~# cat /sys/class/power_supply/bq27411-0/voltage_max_design
4400000
```

噔噔咚。没有修改。我非常确定我的操作没有问题，只能说明是这玩意对一加 6 无效。

# 手动设置充电电流

既然设备树行不通，那就尝试手动限制充电。我在 `power_supply` 目录翻看文件时，发现了几个有意思的东西：
```bash
OnePlus-6:~# ls -l  /sys/class/power_supply/pmi8998-charger/
total 0
-rw-r--r--    1 root     root          4096 Mar 31 11:03 current_max
-r--r--r--    1 root     root          4096 Mar 31 11:03 current_now
lrwxrwxrwx    1 root     root             0 Mar 31 11:03 device -> ../../../c440000.spmi:pmic@2:charger@1000
drwxr-xr-x    2 root     root             0 Mar 31 10:59 extensions
-r--r--r--    1 root     root          4096 Mar 31 11:03 health
drwxr-xr-x    3 root     root             0 Jan  3  1970 hwmon23
-r--r--r--    1 root     root          4096 Mar 31 11:03 manufacturer
-r--r--r--    1 root     root          4096 Mar 31 11:03 model_name
lrwxrwxrwx    1 root     root             0 Mar 31 11:03 of_node -> ../../../../../../../../../firmware/devicetree/base/soc@0/spmi@c440000/pmic@2/charger@1000
-r--r--r--    1 root     root          4096 Mar 31 10:59 online
drwxr-xr-x    2 root     root             0 Mar 31 10:59 power
-rw-r--r--    1 root     root          4096 Mar 31 11:03 status
lrwxrwxrwx    1 root     root             0 Jan  3  1970 subsystem -> ../../../../../../../../../class/power_supply
-r--r--r--    1 root     root          4096 Mar 31 10:59 type
-rw-r--r--    1 root     root          4096 Jan  3  1970 uevent
-r--r--r--    1 root     root          4096 Mar 31 10:59 usb_type
-r--r--r--    1 root     root          4096 Mar 31 11:03 voltage_now
drwxr-xr-x    2 root     root             0 Jan  3  1970 wakeup9
OnePlus-6:~# ls -l  /sys/class/power_supply/bq27411-0/
total 0
-r--r--r--    1 root     root          4096 Mar 31 10:59 capacity
-r--r--r--    1 root     root          4096 Mar 31 10:59 capacity_level
-r--r--r--    1 root     root          4096 Mar 31 10:59 charge_full
-r--r--r--    1 root     root          4096 Mar 31 10:59 charge_full_design
-r--r--r--    1 root     root          4096 Mar 31 10:59 charge_now
-r--r--r--    1 root     root          4096 Mar 31 11:03 constant_charge_current_max
-r--r--r--    1 root     root          4096 Mar 31 10:59 current_now
lrwxrwxrwx    1 root     root             0 Mar 31 11:03 device -> ../../../10-0055
drwxr-xr-x    2 root     root             0 Mar 31 11:03 extensions
drwxr-xr-x    3 root     root             0 Jan  3  1970 hwmon22
-r--r--r--    1 root     root          4096 Mar 31 10:59 manufacturer
lrwxrwxrwx    1 root     root             0 Mar 31 11:03 of_node -> ../../../../../../../../../firmware/devicetree/base/soc@0/geniqup@ac0000/i2c@a88000/bq27441-battery@55
drwxr-xr-x    2 root     root             0 Mar 31 11:03 power
-r--r--r--    1 root     root          4096 Mar 31 10:59 present
-r--r--r--    1 root     root          4096 Mar 31 10:59 status
lrwxrwxrwx    1 root     root             0 Jan  3  1970 subsystem -> ../../../../../../../../../class/power_supply
-r--r--r--    1 root     root          4096 Mar 31 10:59 technology
-r--r--r--    1 root     root          4096 Mar 31 10:59 temp
-r--r--r--    1 root     root          4096 Mar 31 10:59 type
-rw-r--r--    1 root     root          4096 Jan  3  1970 uevent
-r--r--r--    1 root     root          4096 Mar 31 10:59 voltage_max_design
-r--r--r--    1 root     root          4096 Mar 31 10:59 voltage_min_design
-r--r--r--    1 root     root          4096 Mar 31 10:59 voltage_now
```

在 `pmi8998-charger` 目录下有一个可写的文件 `current_max`，查阅资料得知，这个值定义充电电流，默认值是 `125000`：
```bash
OnePlus-6:~# cat /sys/class/power_supply/pmi8998-charger/current_max
125000
```

如果我手动设置为 `0`，然后等待一段时间查看电量：
```bash
OnePlus-6:~# cat /sys/class/power_supply/bq27411-0/capacity  # 设置之前，查看当前电量
100
OnePlus-6:~# echo 0 > /sys/class/power_supply/pmi8998-charger/current_max  # 设置充电电流为 0
OnePlus-6:~# cat /sys/class/power_supply/pmi8998-charger/current_max  # 检查是否生效
0

OnePlus-6:~# cat /sys/class/power_supply/bq27411-0/capacity  # 等待一段时间后，查看当前电量
97
```

电量在下降，证明当前没有在充电。那思路就很清晰了，造一个服务，当电量降到 45% 时开始充电，电量达到 50% 时停止充电。`/sys/class/power_supply/bq27411-0/capacity` 文件提供当前电量百分比数值，让 AI 写一个脚本，脚本文件存放在 `/usr/local/bin/battery-limit.sh`：
```sh
#!/bin/sh

# 定义路径变量
BAT_CAP="/sys/class/power_supply/bq27411-0/capacity"
CHG_LIMIT="/sys/class/power_supply/pmi8998-charger/current_max"

# 定义限制（单位：%）
UPPER_LIMIT=50
LOWER_LIMIT=45

# 定义恢复充电时的电流（单位通常为微安 uA）
RESTORE_CURRENT=125000

while true; do
    # 读取当前电量
    CURRENT_CAP=$(cat "$BAT_CAP")
    
    # 读取当前限制状态（为了减少不必要的写入）
    CURRENT_SETTING=$(cat "$CHG_LIMIT")

    if [ "$CURRENT_CAP" -ge "$UPPER_LIMIT" ]; then
        # 达到或超过 50%，停止充电
        if [ "$CURRENT_SETTING" -ne 0 ]; then
            echo 0 > "$CHG_LIMIT"
            echo "[$(date '+%Y-%m-%d %H:%M:%S')] 当前电量 $CURRENT_CAP%，停止充电"
        fi
    elif [ "$CURRENT_CAP" -le "$LOWER_LIMIT" ]; then
        # 低于或等于 45%，恢复充电
        if [ "$CURRENT_SETTING" -eq 0 ]; then
            echo "$RESTORE_CURRENT" > "$CHG_LIMIT"
            echo "[$(date '+%Y-%m-%d %H:%M:%S')] 当前电量 $CURRENT_CAP%，恢复充电"
        fi
    fi

    # 每 60 秒检查一次，平衡响应速度和功耗
    sleep 60
done
```

然后加到服务里，建一个新服务，文件位于 `/etc/systemd/system/battery-limit.service`
```
[Unit]
Description=OnePlus 6 Battery Charge Limit Service
After=multi-user.target

[Service]
Type=simple
KillMode=control-group
ExecStart=/usr/local/bin/battery-limit.sh
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

然后设置脚本运行权限，重载并启动服务：
```bash
chmod +x /usr/local/bin/battery-limit.sh
systemctl daemon-reload
systemctl restart battery-limit.service
```

看看有没有生效：
```bash
OnePlus-6:/usr/local/bin# systemctl status battery-limit.service
● battery-limit.service - OnePlus 6 Battery Charge Limit Service
     Loaded: loaded (/etc/systemd/system/battery-limit.service; disabled; preset: disabled)
     Active: active (running) since Tue 2026-03-31 16:23:23 CST; 42min ago
 Invocation: fadb62c4dbf347f18f37621eac2f36c1
   Main PID: 20519 (battery-limit.s)
      Tasks: 2 (limit: 9108)
     Memory: 284K (peak: 1.9M)
        CPU: 336ms
     CGroup: /system.slice/battery-limit.service
             ├─20519 /bin/sh /usr/local/bin/battery-limit.sh
             └─22903 sleep 60

Mar 31 16:23:23 OnePlus-6 systemd[1]: Started OnePlus 6 Battery Charge Limit Service.
Mar 31 16:23:24 OnePlus-6 battery-limit.sh[20519]: [2026-03-31 16:23:24] 当前电量 97%，停止充电
```

一切正常，但愿如此设置后电池不会太快鼓包。

后续我注意到 125000 似乎不能让电池恢复充电，原因是这个值是我在接近充满电的情况下获取的，不足以向电池充电。将其设为 500000（500毫安）之后查看电池状态，可看到电池为正在充电：
```
OnePlus-6:~# cat /sys/class/power_supply/bq27411-0/status 
Charging
```

并且充电电流也是根据电池状态动态变化的，并不是定死在 500000。