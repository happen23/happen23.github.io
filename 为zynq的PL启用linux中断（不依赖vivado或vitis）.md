# 背景
zynq是Xilinx的arm+FPGA架构芯片，arm端称作PS，FPGA端称作PL，PS和PL间的同步比较复杂，特别是PL->PS。

petalinux自带的zynq-7000.dtsi里只有PS端的设备树节点，没有PL的

一般用Xilinx的vivado和vitis等工具导出设备树，但我不想安装和使用那些庞大的工具，想试试自己手搓。

幸运的是，我成功了！

# 为PL创建设备树节点

下文提到的目录都以petalinux工程根目录为base
修改`project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi`，添加以下内容：
```
&amba {
    pl: pl@ffff0000 {
        compatible = "nebula,pl";
        reg = <0xffff0000 0x1000>;
        interrupt-parent = <&intc>;
        interrupts = <0 29 1>;
        #address-cells = <1>;
        #size-cells = <0>;
        status = "okay";
    };
};

```
一些说明：
1. `pl@ffff0000`表明pl的配置寄存器地址位于0xffff0000，这实际上是OCM的地址，因为之前PS是通过mmap到OCM的方式与PL通信的。
2. `interrupts = <0 29 1>;`三元组中的
3. 0表示SPI外设（相对global timer、scu timer等PPI外设）
4. 29表示在SPI外设中断编号（全局中断编号-32）
5. 1表示上升沿触发（参考dt-binding头文件）

# 为PL创建platform驱动

```c
static irqreturn_t dab_irq(int irq, void *lp)
{
    struct dab_ctx *ctx = (struct dab_ctx *)lp;

    printk("dab interrupt\n");

    return IRQ_HANDLED;
}

static const struct of_device_id dab_dt_ids[] = {
    { .compatible = "nebula,pl" },
    { /* sentinel */ }
};

static int dab_probe(struct platform_device *pdev)
{
    int ret = 0;
    uint32_t virq = 0;
    struct dab_ctx *ctx = NULL;

    pr_info("enter\n");

    ctx = (struct dab_ctx *) kzalloc(sizeof(struct dab_ctx), GFP_KERNEL);
    if (!ctx) {
        pr_err("Cound not allocate dab device\n");
        ret = -ENOMEM;
        goto exit;
    }
    platform_set_drvdata(pdev, ctx);
    ctx->irq = platform_get_irq(pdev, 0);
    if (ctx->irq < 0) {
        pr_err("Cound not map hwirq to virq, err %d\n", ctx->irq);
        ret = -ENOMEM;
        goto failed_get_virq;
    }
    ret = devm_request_irq(&pdev->dev, ctx->irq, dab_irq, IRQF_SHARED, "dab", ctx);
    if (ret) {
        dev_err(&pdev->dev, "Unable to request IRQ %d (error %d)\n", ctx->irq, ret);
        goto failed_get_virq;
    }
    pr_info("exit\n");

    return 0;

failed_get_virq:
    kfree(ctx);
exit:
    return ret;
}

static int dab_remove(struct platform_device *pdev)
{
    struct dab_ctx *ctx = NULL;

    pr_info("enter\n");
    ctx = platform_get_drvdata(pdev);
    if (ctx) {

    }
    pr_info("exit\n");
}

static struct platform_driver dab_driver = {
    .probe      = dab_probe,
    .remove     = dab_remove,
    .driver     = {
        .name       = "dab",
        .of_match_table = of_match_ptr(dab_dt_ids),
    },
};

module_platform_driver(dab_driver);

```
# 运行效果
```
root@DAB:~# cat /proc/interrupts
           CPU0       CPU1
 16:          0          0     GIC-0  27 Edge      gt
 17:    1079329     870344     GIC-0  29 Edge      twd
 18:          0          0     GIC-0  37 Level     arm-pmu
 19:          0          0     GIC-0  38 Level     arm-pmu
 20:         43          0     GIC-0  39 Level     f8007100.adc
 22:          2          0     GIC-0  57 Level     cdns-i2c
 23:       5097          0     GIC-0  80 Level     cdns-i2c
 25:          0          0     GIC-0  35 Level     f800c000.ocmc
 26:        288          0     GIC-0  59 Level     xuartps
 27:         11          0     GIC-0  82 Level     xuartps
 28:          3          0     GIC-0  51 Level     e000d000.spi
 29:       1339          0     GIC-0  54 Level     eth0
 30:       2791          0     GIC-0  56 Level     mmc0
 31:        538          0     GIC-0  79 Level     mmc1
 32:          0          0     GIC-0  45 Level     f8003000.dmac
 33:          0          0     GIC-0  46 Level     f8003000.dmac
 34:          0          0     GIC-0  47 Level     f8003000.dmac
 35:          0          0     GIC-0  48 Level     f8003000.dmac
 36:          0          0     GIC-0  49 Level     f8003000.dmac
 37:          0          0     GIC-0  72 Level     f8003000.dmac
 38:          0          0     GIC-0  73 Level     f8003000.dmac
 39:          0          0     GIC-0  74 Level     f8003000.dmac
 40:          0          0     GIC-0  75 Level     f8003000.dmac
 41:          0          0     GIC-0  40 Level     f8007000.devcfg
 43:          0          0     GIC-0  43 Level     ttc_clockevent
 49:         24          0     GIC-0  61 Edge      dab
IPI1:          0          0  Timer broadcast interrupts
IPI2:        780       6543  Rescheduling interrupts
IPI3:          8          6  Function call interrupts
IPI4:          0          0  CPU stop interrupts
IPI5:          0          0  IRQ work interrupts
IPI6:          0          0  completion interrupts
Err:          0

```
可以看到，Linux为pl分配的虚拟中断号是49
