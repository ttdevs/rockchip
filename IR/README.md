# RK3399平台增加红外接收功能

[TOC]

## 0x00 概述

项目中需要处理红外键盘操作，研究了一下，这里做个记录。

开发板：`Firefly AIO-3399J`
源码Tag：`AIO-3399J_Android7.1.2_AIO_180613`
Android版本：`Android 7.1.2`
修改文件：
    
- `kernel/arch/arm64/boot/dts/rockchip/rk3399-firefly-port.dtsi`
- `ff420030_pwm.kl`，二选一：
    
    1. 源码：`device/rockchip/common/ff420030_pwm.kl`，修改后需要编译android部分
    2. Android：`/system/usr/keylayout/ff420030_pwm.kl`，修改后重启即可

参考文件：

- `kernel/include/dt-bindings/input/rk-input.h`
- `frameworks/native/include/input/InputEventLabels.h`


## 0x01 准备工作

### IR接收头

    将准备的IR接收头接到主板上，三个脚都接，也可以只接信号和地两只脚。淘宝价格几毛钱一个，种类很多，但是差别很大。开始准备了一个带铁壳的接收头，效果特别差，必须对准才能接收。后来换了一个，效果好很多。
    
### 遥控器

    这个看具体要求，研究目的，我准备了一个乐视的39键红外遥控器，淘宝价20左右，比官方的遥控器按键更丰富。
    
### 查看遥控码

    1. 打开红外调试的log(debug模式直接执行，`adb shell` 中需先执行 `su`)
        
        `echo 1 > sys/module/rockchip_pwm_remotectl/parameters/code_print`
        
    2. 按下遥控器记录下键值信息(debug模式会直接显示，`adb shell` 中可以执行 `cat /proc/kmsg`)，看到信息如下：

        ``` shell
rk3399_firefly_aio_box:/ # cat /proc/kmsg
<6>[24135.793939] USERCODE=0x654c
<6>[24135.820808] RMC_GETDATA=ea
<6>[24274.495380] USERCODE=0x654c
<6>[24274.522295] RMC_GETDATA=e0
<6>[24275.393934] USERCODE=0x654c
<6>[24275.420833] RMC_GETDATA=e1
...
        ```
        
        `USERCODE` 遥控器码，这里是 `0x654c`；`RMC_GETDATA` 每个按键的值。后面会用到这些信息。
        
    > PS：如果红外接收正常，遥控器按下时主板的两个蓝色指示灯有一个会闪烁。
                

## 0x02 定制

定制主要有这样几个步骤：修改内核，编译内核，将内核刷入开发板，修改Android部分，重启等。

### 修改内核

- `kernel/arch/arm64/boot/dts/rockchip/rk3399-firefly-port.dtsi`

    找到 `&pwm3` ，之前的代码如下：
    
    ``` shell
    ...
    &pwm3 {
    	status = "okay";
    	interrupts = <GIC_SPI 61 IRQ_TYPE_LEVEL_HIGH 0>;
    	compatible = "rockchip,remotectl-pwm";
    	remote_pwm_id = <3>;
    	handle_cpu_id = <1>;
    	remote_support_psci = <1>;
    
    	ir_key1{
    		rockchip,usercode = <0xff00>;
    		rockchip,key_table =
    			<0xeb  KEY_POWER>,
    			<0xec  KEY_MENU>,
    			<0xfe  KEY_BACK>,
    			<0xb7  KEY_HOME>,
    			<0xa3  KEY_WWW>,
    			<0xf4  KEY_VOLUMEUP>,
    			<0xa7  KEY_VOLUMEDOWN>,
    			<0xf8  KEY_REPLY>,
    			<0xfc  KEY_UP>,
    			<0xfd  KEY_DOWN>,
    			<0xf1  KEY_LEFT>,
    			<0xe5  KEY_RIGHT>;
    	};
    };
    ...
    ```
    
    现在将我们自己的红外按键值添加进去，修改之后的代码如下：
    
    ``` shell
    ...
    &pwm3 {
    	status = "okay";
    	interrupts = <GIC_SPI 61 IRQ_TYPE_LEVEL_HIGH 0>;
    	compatible = "rockchip,remotectl-pwm";
    	remote_pwm_id = <3>;
    	handle_cpu_id = <1>;
    	remote_support_psci = <1>;
    
    	ir_key1{
    		rockchip,usercode = <0xff00>;
    		rockchip,key_table =
    			<0xeb  KEY_POWER>,
    			<0xec  KEY_MENU>,
    			<0xfe  KEY_BACK>,
    			<0xb7  KEY_HOME>,
    			<0xa3  KEY_WWW>,
    			<0xf4  KEY_VOLUMEUP>,
    			<0xa7  KEY_VOLUMEDOWN>,
    			<0xf8  KEY_REPLY>,
    			<0xfc  KEY_UP>,
    			<0xfd  KEY_DOWN>,
    			<0xf1  KEY_LEFT>,
    			<0xe5  KEY_RIGHT>;
    	};
    
    	ir_key2{
    		rockchip,usercode = <0x654c>;
    		rockchip,key_table = 
    			<0xf5  KEY_POWER>,
    			<0xfe  KEY_1>,
    			<0xfd  KEY_2>,
    			<0xfc  KEY_3>,
    			<0xfb  KEY_4>,
    			<0xfa  KEY_5>,
    			<0xf9  KEY_6>,
    			<0xf8  KEY_7>,
    			<0xf7  KEY_8>,
    			<0xf6  KEY_9>,
    			<0xba  KEY_0>,
    			...
    			<0xeb  KEY_STOP>;
    	};
    };
    ...
    ``` 

    说明：
    
    1. dts的文件格式比较严格，请确保格式正确，否则会编译错误
    2. `handle_cpu_id`： CPU哪个核处理红外中断，建议避开第0个
    3. `status`、`interrupts`、`compatible`、`remote_pwm_id`、`remote_support_psci` 用默认，不需修改
    4. 增加了一个 `ir_key2`，如果你有更多，可以用`ir_key3`、`ir_key4`，以此类推
    5. `usercode`： 填入上面检测遥控器时记录的 USERCODE，我的是 `0x654c`
    6. `key_table`： 有两列，第一列填入遥控器每个按键的键值；第二列填入你希望给这个键的定义。第二列可选的定义在这里查看： `kernel/include/dt-bindings/input/rk-input.h`

### 将修改编译进内核

这部分官方文档提到的。在源码Tag `AIO-3399J_Android7.1.2_AIO_180613` 中，默认是选中的，可以跳过直接编译内核烧录即可。

- 向配置文件 `drivers/input/remotectl/Kconfig` 中添加配置：

    ``` shell
config ROCKCHIP_REMOTECTL_PWM
    bool "rockchip remoctrl pwm capture"
default n
    ```
    
- 修改 `drivers/input/remotectl` 路径下的 `Makefile`，添加如下编译选项：
    
    `obj-$(CONFIG_ROCKCHIP_REMOTECTL_PWM)+= rockchip_pwm_remotectl.o`
    
- 在 kernel 路径下使用 `make menuconfig`，选中IR驱动：

    ``` shell
Device Drivers --->
    Input device support --->
        rockchip remotectl --->
            [*]   rockchip remoctrl pwm capture
    ```

保存之后，编译 `kernel`：

``` shell
cd kernel
make ARCH=arm64 firefly_defconfig
make -j8 ARCH=arm64 rk3399-firefly-aio.img
```

烧录 `kernel.img` 即可（不确定要不要烧录 `resource.img`，我是全量烧录的）。

### Android部分修改

这部分主要修改 `/system/usr/keylayout/ff420030_pwm.kl`，我的修改之后为：

``` shell
rk3399_firefly_aio_box:/ # cat /system/usr/keylayout/ff420030_pwm.kl
#$_FOR_ROCKCHIP_RBOX_$
#$_rbox_$_modify_$_chenzhi_20120220: add for IR remote

# number:10
key 2		1
key 3		2
key 4		3
key 5		4
key 6		5
key 7		6
key 8		7
key 9		8
key 10		9
key 11		0

# function: 4 + 回看+列表+屏显+信号源
key 377		TV
key 376		TV_INPUT_VGA_1
key 240		TV_INPUT_HDMI_1
key 379		TV_INPUT_COMPOSITE_1

# system: 13
key 116   	POWER
key 102		HOME
key 158   	BACK
key 139   	MENU
key 352    	DPAD_CENTER
key 115   	VOLUME_UP
key 114   	VOLUME_DOWN
key 113		VOLUME_MUTE
key 103		DPAD_UP
key 108		DPAD_DOWN
key 105		DPAD_LEFT
key 106		DPAD_RIGHT
key 141		SETTINGS

# media: 8
key 402   	CHANNEL_UP
key 403   	CHANNEL_DOWN
key 164		MEDIA_PLAY_PAUSE
key 128		MEDIA_STOP
key 407		MEDIA_NEXT
key 412		MEDIA_PREVIOUS
key 242		MEDIA_REWIND
key 241		MEDIA_FAST_FORWARD
rk3399_firefly_aio_box:/ #
```

这个文件有三列：

- 第一列：固定值 `key`
- 第二列：红外键值对应按键的按键码，如果不知道可以通过 `getevent` 命令查看，如
    
    ``` shell
rk3399_firefly_aio_box:/ # getevent
add device 1: /dev/input/event2
  name:     "rk29-keypad"
add device 2: /dev/input/event0
  name:     "ff420030.pwm"
add device 3: /dev/input/event1
  name:     "rockchip,rt5640-codec Headphone Jack"
/dev/input/event0: 0001 0073 00000001
/dev/input/event0: 0000 0000 00000000
/dev/input/event0: 0001 0073 00000000
/dev/input/event0: 0000 0000 00000000
/dev/input/event0: 0001 0072 00000001
/dev/input/event0: 0000 0000 00000000
/dev/input/event0: 0001 0072 00000000
/dev/input/event0: 0000 0000 00000000
    ```
    这里有两个按键，第一个为0x73，第二个为0x72，都是16进制。
    
- 第三列：Android层按键定义。

比如，我们遥控器电源键的定义如下：

    `<0xeb  KEY_POWER> <key 116 POWER>`

0xeb：遥控器的红外码
KEY_POWER：`kernel/include/dt-bindings/input/rk-input.h` 中定义的 `KEY_POWER`
116：`kernel/include/dt-bindings/input/rk-input.h` 中定义的 `KEY_POWER` 的值
POWER：`frameworks/native/include/input/InputEventLabels.h` 中定义的 `POWER`

修改之后重启设备即可生效。 如果希望编译进ROM，可以直接修改 `device/rockchip/common/ff420030_pwm.kl`。

## 0x03 增加按键

上面描述的是将遥控器的按键映射到系统中已经定义的按键上，有时候我们还希望增加系统中不存在的按键。下面介绍如何增加自己的按键。

TODO

## 0xFF

第一次尝试编译源码，中间遇到各种问题，但是最终还是成功了。

参考
---
1. http://www.t-firefly.com/doc/product/info/id/329.html
2. 

