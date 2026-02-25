# input 子系统
## 概览
- [ ] 添加示例图
图中：
  - 驱动层：输入设备的具体驱动程序，如按键驱动程序，向内核层报告输入内容；
  - 核心层：承上启下，为驱动层提供输入设备注册和操作接口，通知事件层对输入事件进行处理；
    ```c
    struct class input_class = {
        .name = "input",
        .devnode = input_devnode,
    };

    static int __init input_init(void) {
        int err;
        err = class_register(&input_class);
        ...
        err = register_chrdev_region(MKDEV(INPUT_MAJOR, 0), INPUT_MAX_CHAR_DEVICES, "input");
        ...
        return 0;
    }
    ```
      - `drivers/input/input.c`：核心层会向 Linux 内核注册一个字符设备；
      - 通过 `class_register` 在用户空间生成 `sys/class/input` 目录接口；
      - input 子系统所有设备的主设备号都是 `INPUT_MAJOR(13, include/uapi/linux/major.h)`，通过 `register_chrdev_region` 函数完成设备号注册；
      - **在使用 input 子系统处理输入设备时不需要去注册字符设备了，只需要向子系统注册一个 `input_device` 即可**；
  - 事件层：主要和用户空间进行交流；

## 使用
### 注册
在子系统中，使用 `input_dev` 结构体表示 input 设备。

1. `input_dev` 结构体：
   ```c
   struct input_dev {
    const char *name;
    const char *phys;
    const char *uniq;
    struct input_id id;

    unsigned long propbit[BITS_TO_LONGS(INPUT_PROP_CNT)];
    unsigned long evbit[BITS_TO_LONGS(EV_CNT)];
    unsigned long keybit[BITS_TO_LONGS(KEY_CNT)];
    unsigned long relbit[BITS_TO_LONGS(REL_CNT)];
    unsigned long absbit[BITS_TO_LONGS(ABS_CNT)];
    unsigned long ledbit[BITS_TO_LONGS(LED_CNT)];
    unsigned long sndbit[BITS_TO_LONGS(SND_CNT)];
    unsigned long ffbit[BITS_TO_LONGS(FF_CNT)];
    unsigned long swbit[BITS_TO_LONGS(SW_CNT)];
    ...
    bool devres_managed;
   }
   ```
   - `evbit`：表示输入事件的类型，可选内容位于 `include/uapi/linux/input.h`，例如 `EV_SYN`（同步事件）、`EV_KEY`（按键事件）、`EV_REL`（相对坐标事件）等；
   - `keybit`：按键值被定义在 `include/uapi/linux/input.h`；

2. 流程
   1. 申请一个 `input_dev` 结构体标量： `struct input_dev *input_allocate_device(void);`
   2. 申请后进行初始化，主要是初始化事件类型（`evbit`）和事件值（`keybit`）这两种。初始化方式：
        ```c
        // 方式 1
        __set_bit(EV_KEY, inputDev->evbit); // 设置产生按键事件；
        __set_bit(EV_REP, inputDev->evbit); // 重复事件
        __set_bit(KEY_0, inputDev->keybit); // 设置产生哪些按键值
        // 方式 2，3：略
        ```
   3. 初始化后向内核注册：`int input_register_device(struct input_dev *dev);`;
   4. 取消注册：`void input_unregister_device(struct input_dev *dev);`；
   5. 释放一个不再使用的 `input_dev` 结构体 `void input_free_device(struct input_dev *dev);`；

### 事件上报
不同的事件，其上报事件的 API 函数不同。产生事件时，在相应的处理函数（如中断服务程序）中上报事件。
- 通用 API:
    ```c
    /* 
     * type: 上报的事件类型，比如 EV_KEY; 
     * code: 事件码，比如 KEY_0, KEY_1 等；
     * value: 事件值，比如 1 表示按键按下，0 表示按键松开
     * 此函数可以上报所有的事件类型和事件值
     */
    void input_event(struct input_dev *dev, unsigned int type, unsigned int code, int value);
    ```
- 对 `input_event` 的进一步封装：
  ```c
  input_report_key(struct input_dev *dev, unsigned int code, unsigned int value);
  input_report_rel(...);
  input_report_abs(...);
  ```

### 事件同步
上报完事件后，需使用 `input_sync` 以通知 Linux 内核 input 子系统上报结束，它的本质是上报一个同步事件。
```c
void input_sync(struct input_dev *dev);
```

### input_event 结构体
Linux 内核使用 `input_event` 结构体来表示所有的输入事件，位于 `include/uapi/linux/input.h`：
```c
typedef long                __kernel_long_t;
typedef __kernel_long_t     __kernel_time_t;
typedef __kernel_long_t     __kernel_suseconds_t;

struct timeval {
    __kernel_time_t tv_sec;         // 秒
    __kernel_suseconds_t tv_usec;   // 微秒
}

struct input_event {
    struct timeval time; // 事件发生的时间；
    _u16 type;
    _u16 code;
    _s32 value;
}
```
所有的输入设备最终都是按照 `input_event` 结构体呈现给用户，用户应用程序可以通过 `input_event` 来获取到具体的输入事件或相关的值；

## 示例
如下是按键 input 驱动程序示例：
```c
#include <linux/types.h>
#include <linux/kernel.h>
#include <linux/delay.h>
#include <linux/ide.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/errno.h>
#include <linux/gpio.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/of.h>
#include <linux/of_address.h>
#include <linux/of_gpio.h>
#include <linux/input.h>
#include <linux/semaphore.h>
#include <linux/timer.h>
#include <linux/irq.h>
#include <linux/of_irq.h>
#include <asm/mach/map.h>
#include <asm/uaccess.h>
#include <asm/io.h>


#define KEYINPUT_CNT        1           // 设备号个数
#define KEYINPUT_NAM        "keyinput"
#define KEY0VALUE           0x01        // KEY0 按键值
#define INVAKEY             0xff        // 无效按键值
#define KEY_NUM             1           // 按键数量

// 中断 io 描述结构体
struct irq_keydesc {
    int gpio;
    int irqnum;
    unsigned char value;        // 按键对应的按键
    char name[10];
    irqreturn_t (*handler)(int, void *);        // 中断服务函数
};

// keyinput 设备结构体
struct keyinput_dev {
    dev_t devid;
    struct cdev cdev;
    struct class *class;
    struct device *device;
    struct device_node *nd;
    struct timer_list timer;        // 定义一个定时器
    struct irq_keydesc irqkeydesc[KEY_NUM];
    unsigned char curkeynum;        // 当前的按键号
    struct input_dev *inputdev;
};

struct keyinput_dev keinputdev;

/* 中断服务函数，开启定时器，延迟 10ms，定时器用于按键消抖 */
static irqreturn_t key0_handler(int irq, void *dev_id) {
    struct keyinput_dev *dev = (struct keyinput_dev *)dev_id;

    dev->curkeynum = 0;
    dev->timer.data = (volatile long)dev_id;
    mod_timer(&dev->timer, jiffies + msecs_to_jiffies(10));
    return IRQ_RETVAL(IRQ_HANDLED);
};

/* 定时器服务函数，延迟消抖后再次读取按键值，如果还是处于按下状态就表示按键有效 */
void tiemr_function(unsigned long arg) {
    unsigned char value;
    unsigned char num;
    struct irq_keydesc *keydecs;
    struct keyinput_dev *dev = (struct keyinput_dev *)arg;

    num = dev->curkeynum;
    keydesc = &dev->irqkeydesc[num];
    value = gpio_get_value(keydesc->gpio);      // 获取对应 gpio 的值
    if (value == 0) {
        // 按下按键
        // 上报按键值 
        // input_event(dev->inputdev, EV_KEY, keydesc->value, 1);
        input_report_key(dev->inputdev, keydesc->value, 1); // 1，按下
        input_sync(dev->inputdev);
    } else {
        // 松开按键
        // input_event(dev->inputdev, EV_KEY, keydesc->value, 0);
        input_report_key(dev->inputdev, keydesc->value, 0);
        input_sync(dev->inputdev);
    }
}

// 按键 io 初始化
static int keyio_init(void) {
    unsigned char i = 0;
    char name[10];
    int ret = 0;

    keyinputdev.nd = of_find_node_by_path("/key");
    if (keyinputdev.nd == NULL) {
        printk("key node not find!\r\n");
        return 0;
    }

    // 提取 gpio
    for (i = 0; i < KEY_NUM; i++) {
        keyinputdev.irqkeydesc[i].gpio = of_get_named_gpio(keyinputdev.nd, "key-gpio", i);
        if (keyinputdev.irqkeydesc[i].gpio < 0) {
            printk("can't get key%d\r\n", i);
        }
    }

    // 初始化 key 所使用的 io，并且设置成中断模式
    for (i = 0; i < KEY_NUM; ++i) {
        memset(keyinputdev.irqkeydesc[i].name, 0, sizeof(name));
        sprintf(keyinputdev,irqkeydesc[i].name, "KEY%d", i);
        gpio_request(keyinputdev.irqkeydesc[i].gpio, keyinputdev.irqkeydesc[i].name);
        gpio_direction_input(keyinputdev.irqkeydesc[i].gpio);
        keyinputdev.irqkeydesc[i].irqnum = irq_of_parse_and_map(keyinputdev.nd, i);
    }

    // 申请中断
    keyinputdev.irqkeydesc[0].handler = key0_handler;
    keyinputdev.irqkeydesc[0].value = KEY_0;

    for (i = 0; i < KEY_NUM; ++i) {
        ret = request_irq(keyinputdev.irqkeydesc[i].irqnum, keyinputdev.irqkeydesc[i].handler, IRQF_TRIGER_FALLING | IRQF_TRIGGER_RISING, keyinputdev.irqkeydesc[i].name, &keyinputdev);
        if (ret < 0) {
            printk("irq %d request failed!\r\n", keyinputdev.irqkeydesc[i].irqnum);
            return -EFAULT;
        }
    }

    // 创建定时器
    init_timer(&keyinputdev.timer);
    keyinputdev.tiemr.fucntion = timer_function;

    // 申请 input_dev
    keyinputdev.inputdev = input_allocate_device();
    keyinputdev.inputdev->name = KEYINPUT_NAME;
#if 0
    // 初始化 input_dev，设置产生哪些事件
    __set_bit(EV_KEY, keyinputdev.inputdev->evbit);
    __set_bit(EV_REP, keyinputdev.inputdev->evbit);
    // 初始化 input_dev，设置产生哪些按键
    __set_bit(KEY_0, keyinputdev.inputdev->keybit);
#endif

#if 0
    keyinputdev.inputdev->evbit[0] = BIT_MASK(EV_KEY) | BIT_MASK(EV_REP);
    keyinputdev.inputdev->keybit[BIT_WORD(KEY_0)] |= BIT_MASK(KEY_0);
#endif
    keyinputdev.inputdev->evbit[0] = BIT_MASK(EV_KEY) | BIT_MASK(EV_REP);
    input_set_capability(keyinputdev.inputdev, EV_KEY, KEY_0);

    // 注册输入设备
    ret = input_register_device(keyinputdev.inputdev);
    if (ret) {
    printk("register input device failed!\r\n");
    return ret;
    }
    
    return 0;
}

static int __init keyinput_init(void) {
    keyio_init();
    return 0;
}

static void __exit keyinput_exit(void) {
    unsigned int i = 0;
    // 删除定时器
    del_timer_sync(&keyinputdev.timer);
    // 释放中断
    for (i = 0; i < KEY_NUM; ++i) {
        free_irq(keyinputdev.irqkeydesc[i].irqnum, &keyinputdev);
    }
    // 释放 io
    for (i = 0; i < KEY_NUM, ++i) {
        gpio_free(keyinputdev.irqkeydesc[i].gpio);
    }
    // 释放 input_dev
    input_unregister_device(keyinputdev.inputdev);
    input_free_device(keyinputdevice.inputdev);
}

module_init(keyinput_init);
module_exit(keyinput_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("xxx");
```
