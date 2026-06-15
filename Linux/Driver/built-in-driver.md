# Linux 内核自带驱动
## LED
Linux 内核的 LED 驱动采用 platform 框架。

1. 使能 LED 驱动：
```bash
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig

# 配置路径：
Device Drivers
    LED Support (NEW_LEDS [=y])
        LED Support for GPIO connected LEDs

#.config
CONFIG_LEDS_GPIO=y
```

2. 源码解析
- Makefile
    ```c
    // drivers/leds/Makefile
    #LED Core
    obj-$(CONFIG_NEW_LEDS)              += led-core.o
    ...
    obj-$(CONFIG_LEDS_GPIO_REGISTER)    += leds-gpio-register.o
    obj-$(CONFIG_LEDS_GPIO)             += leds-gpio.o
    ...
    ```
    - 当 `CONFIG_LEDS_GPIO=y` 时，leds-gpio.c 被编译到内核；
- .c 文件
    ```c
    // drivers/leds/leds-gpio.c
    static struct gpio_leds_priv *gpio_leds_create(struct platform_device *pdev) {
        struct device *dev = &pdev->dev;
        struct fwnode_handle *child;
        struct gpio_leds_priv *priv;
        int count, ret;
        struct device_node *np;

        count = device_get_child_node_count(dev);
        if (!count)
            return ERR_PTR(-ENODEV);
        
        priv = devm_kzalloc(dev, sizeof_gpio_leds_priv(count), GFP_KERNEL);
        if (!priv)
            return ERR_PTR(-ENOMEM);

        device_for_each_child_node(dev, child) {
            struct gpio_led led = {};
            const char *state = NULL;

            led.gpiod = devm_get_gpiod_from_child(dev, NULL, child);
            if (IS_ERR(led.gpio)) {
                fwnode_handle_put(child);
                ret = PTR_ERR(led.gpiod);
                goto err;
            }

            np = of_node(child);

            if (fwnode_property_present(child, "label")) {
                fwnode_property_read_string(child, "label", &led.name);
            } else {
                if (IS_ENABLED(CONFIG_OF) && !led.name && np)
                    led.name = np->name;
                if (!led.name)
                    return ERR_PTR(-EINVAL);
            }

            fwnode_property_read_string(child, "linux,default-trigger", &led.default_trigger);
            
            if (!fwnode_property_read_string(child, "default-state", &state)) {
                if (!strcmp(state, "keep"))
                    led.default_state = LEDS_GPIO_DEFSTATE_KEEP;
                else if (!strcmp(state, "on"))
                    led.default_state = LEDS_GPIO_DEFSTATE_ON;
                else
                    led.default_state = LEDS_GPIO_DEFSTATE_OFF;
            }

            if (fwnode_property_present(child, "retain-state-suspended"))
                led.retain_state_suspended = 1;
            
            ret = create_gpio_led(&led, &priv->leds[priv->num_leds++]),
            if (ret < 0) {
                fwnode_handle_put(child);
                goto err;
            }
        }

        return priv;
    err:
        for (count = priv->num_leds - 2; count >= 0; count--)
            delete_gpio_led(&priv-leds[count]);
        return ERR_PTR(ret);
    }

    static int gpio_led_probe(struct platform_device *pdev) {
        struct gpio_led_platform_data *pdata = dev_get_platdata(&pdev->dev);
        struct gpio_leds_priv *priv;
        int i, ret = 0;

        if (pdata && pdata->num_leds) {     // 非设备树方式
            // 获取 platform_device 信息
        } else {                            // 设备树方式
            priv = gpio_leds_create(pdev);
            if (IS_ERR(priv))
                return PTR_ERR(priv);
        }

        platform_set_drvdata(pdev, priv);

        return 0;
    }

    static const struct of_device_id of_gpio_leds_match[] = {
        {.compatilbe = "gpio-leds", },
        {},
    };

    ...

    static struct platform_driver gpio_led_driver = {
        .probe = gpio_led_probe,
        .remove = gpio_led_remove,
        .driver = {
            .name = "leds-gpio",
            .of_match_table = of_gpio_leds_match,
        },
    };

    module_platform_driver(gpio_led_driver);
    ```
    - 内核 LED 驱动兼容 `gpio-leds` 和 `leds-gpio` 的 LED 设备。
    - 驱动名为 `leds-gpio`，在 `/sys/bus/platform/drivers` 目录下会生成一个对应的文件。
    - 通过 `module_platform_driver` 宏完成 platform_driver 注册（__init）与注销（__exit）代码。

3. 设备树相关
- 参考设置：`Documentation/devicetree/bindings/leds/`：
  - `leds-gpio.txt`;
  - `common.txt`;
- 创建一个节点表示 LED 灯设备，如果存在多个 LED 灯则每个 LED 灯都作为该节点的子节点；
- 设置 `compatible` 属性为 `gpio-leds`；
- 设置 `label` 属性（可选），一般用来表示 LED 灯的名字，如 "red"、"green" 等；
- 设置 `gpios` 属性值，表示此 LED 所使用的 GPIO；
- 设置 `linux,default-trigger` 属性，LED 灯的默认功能：
  - backlight: LED 灯作为背光；
  - default-on: LED 打开；
  - heartbeat: LED 灯作为心跳指示灯，可以作为系统运行提示灯；
  - ide-disk: LED 作为硬盘活动指示灯；
  - timer: LED 周期性闪烁，由定时器驱动，闪烁频率可以修改。
- 可以设置 `default-state` 属性，为 on, off 或 keep，其中 keep 表示 LED 等保持当前模式。
- 示例：
    ```dts
    dtsleds {
        compatible = "gpio-leds"

        led0 {
            lable = "red";
            gpios = <&gpio1 3 GPIO_ACTIVE_LOW>;
            default-state = "off";
        };
        
        led1 {
            lable = "blue";
            gpios = <&gpio1 2 GPIO_ACTIVE_HIGH>;
            linux,default-trigger = "heartbeat";
            default-state = "on";
        };
    };
    ```

4. 调试
- `sys/bus/platform/devices/dtsleds` 目录是否存在；
- 控制 led0 的亮灭：
    ```bash
    echo 1 > /sys/class/leds/red/brightness
    echo 0 > /sys/class/leds/red/brightness
    ```

## 按键
1. 使能 KEY 驱动
```bash
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig

# 配置路径：
Device Drivers
    Input device support
        Generic input layer (needed for keyboard, mouse, ...)(INPUT [=y])
            Keyboards (INPUT_KEYBOARD [=y])
                GPIO Buttons
```

2. 源码解析
- 配置文件
    ```config
    // 在 .config 中出现“CONFIG_KEYBOARD_GPIO=y”
    ```

- .c 文件
    ```c
    // drivers/input/keyboard/gpio_keys.c
    static const struct of_device_id gpio_keys_of_match[] = {
        {.compatible = "gpio-keys", },
        {/* sentinel */}
    };

    static int gpio_keys_probe(struct platform_device *pdev) {
        struct device *dev = &pdev->dev;
        const struct gpio_keys_platform_data *pdata = dev_get_platdata(dev);
        struct gpio_keys_drvdata *ddata;
        struct input_dev *input;
        size_t size;
        int i, error;
        int wakeup = 0;

        if (!pdata) {
            pdata = gpio_keys_get_devtree_pdata(dev);
            if (IS_ERR(pdata))
                return PTR_ERR(pdata);
        }

        size = sizeof(struct gpio_keys_drvdata) +
                pdata->nbuttons * sizeof(struct gpio_button_data);
        ddata = devm_kzalloc(dev, size, GFP_KERNEL);
        if (!ddata) {
            dev_err(dev, "failed to allocate state\n");
            return -ENOMEM;
        }

        input = devm_input_allocate_device(dev);
        if (!input) {
            dev_err(dev, "failed to allocate input device\n");
            return -ENOMEM;
        }

        ddata->pdata = pdata;
        ddata->input = input;
        mutex_init(&ddata->disable_lock);

        platform_set_drvdata(pdev, ddata);
        input_set_drvdata(input, ddata);

        input->name = pdata->name ? : pdev->name;
        input->phys = "gpio-keys/input0";
        input->dev.parent = &pdev->dev;
        input->open = gpio_keys_open;
        input->close = gpio_keys_close;

        input->id.bustype = BUS_HOST;
        input->id.vendor = 0x0001;
        input->id.product = 0x0001;
        input->id.version = 0x0100;

        /* Enable auto repeat feature of Linux input subsystem */
        if (pdata->rep)
            __set_bit(EV_REP, input->evbit);

        for (i = 0; i < pdata->nbuttons; i++) {
            const struct gpio_keys_button *button = &pdata->buttons[i];
            struct gpio_button_data *bdata = &ddata->data[i];

            error = gpio_keys_setup_key(pdev, input, bdata, button);
            if (error)
                return error;

            if (button->wakeup)
                wakeup = 1;
        }

        error = sysfs_create_group(&pdev->dev.kobj, &gpio_keys_attr_group);
        if (error) {
            dev_err(dev, "Unable to export keys/switches, error: %d\n",
                error);
            return error;
        }

        error = input_register_device(input);
        if (error) {
            dev_err(dev, "Unable to register input device, error: %d\n",
                error);
            goto err_remove_group;
        }

        device_init_wakeup(&pdev->dev, wakeup);

        return 0;

    err_remove_group:
        sysfs_remove_group(&pdev->dev.kobj, &gpio_keys_attr_group);
        return error;
    }

    ...

    static struct platform_driver gpio_keys_device_driver = {
        .probe = gpio_keys_probe,
        .remove = gpio_keys_remove,
        .driver = {
            .name = "gpio-keys",
            .pm   = &gpio_keys_pm_ops,
            .of_match_table = of_match_ptr(gpio_keys_of_match),
        }
    };

    static int __init gpio_keys_init(void) {
        return platform_driver_register(&gpio_keys_device_driver);
    }

    static void __exit gpio_keys_exit(void) {
        platform_driver_unregister(&gpio_keys_device_driver);
    }
    ```
    - 采用 platform 驱动框架，在 KEY 驱动上使用了 input 子系统实现。

3. 设备树相关
- 参考设置：`Documentation/devicetree/bindings/input/gpio-keys.txt`。
- 节点要求：
  - 节点名字为 `"gpio-keys"`；
  - 节点的 `compatible` 属性值设置为 `"gpio-keys"`；
  - 所有的 KEY 都是 `"gpio-keys"` 节点的子节点，每个子节点可以用如下属性描述：
    - `gpios`：KEY 所连接的 GPIO 信息；
    - `interrupts`：KEY 所使用 GPIO 中断信息，不是必须的，可以不写；
    - `label`：KEY 名字；
    - `linux,code`：KEY 要模拟的按键。
  - 如果按键要支持连按的话要加入 autorepeat。
- 示例：
     ```dts
     gpio-keys {
         compatible = "gpio-keys";
         #address-cells = <1>;
         #size-celss = <0>;
         autorepeat;
         key0 {
             label = "GPIO Key Enter";
             linux,code = <KEY_ENTER>;
             gpios = <&gpio1 18 GPIO_ACTIVE_LOW>;
         };
     };
     ```

4. 调试
- 设备信息查看：
    ```bash
    cat /proc/bus/input/devices
    ```
- 输入检查-`hexdump`
    ```bash
    hexdump /dev/input/xxx
    ```