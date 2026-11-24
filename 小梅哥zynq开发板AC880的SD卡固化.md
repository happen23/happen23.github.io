# 背景
AC880是[小梅哥](https://fpga.cn/)生产的一款zynq开发板，我司买来用于评估其MIPI功能，在用JTAG跑通MIPI示例工程后，需要将示例工程固化到SD卡上。

# 典型步骤

小梅哥配套教程的步骤，简单来说就是
1. 在vivado的BD（block design）里添加SD外设
2. 重新generate output products并update HDL wrapper
3. 重新generate bitstream并export hardware，生成xsa文件
4. 点击vivado的tools菜单里的launch Vitis IDE，在vivado工程根目录下新建vitis目录做workspace，点确定
5. 基于导出的xsa文件更新platform工程
6. 点击platform.spr文件，在右侧`standalone on ps7_cortex9_0`下`Board Support Package`，点击`Modify BSP Settings...`
7. 在弹出的对话框里勾选`xilffs`，这样FSBL才具备访问SD上FAT分区的能力，才能将bit文件下载到PL
8. 点击ok按钮，编译platform工程，再重编app工程
9. 右键system工程，选择`Create Boot Image`，在弹出的对话框里检查fsbl.elf的路径是否正确，然后确定
10. 在`_ide\bootimage`目录下找到BOOT.BIN，将其拷贝到SD卡的FAT分区
11. 将拨码开关拨成`上下上`，将SD卡插到卡槽，将开发板上电，应该能看到核心板上青色的LED灯亮起来了。

# 青色的LED灯不亮！

这个可能是因为SD卡接触不良，尝试多插拔几次，或者用眼镜布擦拭几下金手指（成功概率很高），就能解决。

## 还是不亮

如果还是启动不了，可以用vitis的attach机制：
1. 在debug configuration界面点击Single Application Debug，新建启动项
2. 在main选项卡的`debug type`下拉框里选中`Attach to running target`
3. 点击apply和debug，就连接上zynq了

连上zynq后，点击core0，将其暂停，然后在memory选项卡里添加一个`Monitor memory`条目，地址是0xf800025c，看其值是否等于5，是的话说明拨码开关配对了

再按一下PS端的reset按键（位号S4，就在音频接口的旁边），看能否亮起来，因为AC880的SD卡和eMMC是共用一套SDIO控制器，通过管脚复用芯片TXS02612RTWR实现，因此可能BootROM读取的其实是eMMC

还不行的话就试试在vitis的attach界面复位下core0甚至整个PS

还不行就咨询小梅哥团队或安富利、AMD的FAE吧。

# 小梅哥原理图的一点设计缺陷

SDIO复用芯片的VCC是接的PS的VCCIO_BANK1，也就是说zynq上电时SDIO复用芯片才上电，那bootrom读取SD卡时，SD卡本身可能还未初始化完毕，正确的做法是SD卡在zynq上电前就上电。

小梅哥默认是从QSPI falsh启动，QSPI的VCC接到VCCIO_BANK0，虽然也有时序风险，但是一来QSPI flash的初始化应该比MMC快，二来中间没有隔一个复用芯片，所以风险较小

更严谨的判定方法是用示波器抓取波形，如果上电后SD卡的MISO信号线上无波形，就能确定是时序问题。
