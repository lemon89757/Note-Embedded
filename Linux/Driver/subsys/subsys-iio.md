# IIO 子系统
Industrial I/O

## 作用和定位
## 驱动框架

---
《正点原子》：
- 当使用的传感器本质是 ADC 或 DAC 器件的时候，可以优先考虑使用 IIO 驱动框架。
- 数据结构：
    ```c
    // include/linux/iio/iio.h
    struct iio_dev {
        int id;
        
        int modes;
        int currentmode;
        struct device dev;

        struct iio_event_interface *event_interface;

        struct iio_buffer *buffer;
        struct list_head buffer_list;
        int scan_bytes;
        struct mutex mlock;

        const unsigned long *available_scan_masks;
        unsigned masklength;
        const unsigned long *active_scan_mask;
        bool scan_timestamp;
        unsigned scan_index_timestamp;
        struct iio_trigger *trig;
        struct iio_poll_func *pollfunc;

        struct iio_chan_spec const *channels;
        int num_channels;

        struct list_head channel_attr_list;
        struct attribute_group chan_attr_group;
        const char *name;
        const struct iio_info *info;
        struct mutex info_exist_lock;
        const struct iio_buffer_setup_ops  *setup_ops;
        struct cdev chrdev;
    #define IIO_MAX_GROUPS  6
        const struct attribute_group *groups[IIO_MAX_GROUPS + 1];
        int groupcounter;
        unsigned long flags;
    #if defined(CONFIG_DEBUG_FS)
        struct dentry   *debugfs_dentry;
        unsigned cached_reg_attr;
    #endif
    };
    ```
- API
    ```c
    // iio_dev 申请
    /** 
     * @sizeoof_priv: 私有数据内存空间大小， 一般将自定义的设备结构体变量作为 iio_dev 的私有数据
     * @return: 成功-iio_dev 首地址；失败-NULL
     */
    struct iio_dev *iio_device_alloc(int sizeof_priv);
    // devm: devm_iio_device_alloc()

    // 示例：
    struct icm20608_dev *dev;
    struct iio_dev *indio_dev;

    indio_dev = iio_device_alloc(sizeof(*dev));
    if (!indio_dev)
        return -ENOMEM;
    dev = iio_priv(indio_dev);

    // iio_dev 注册
    /* 0-成功；其他值-失败 */
    int iio_device_register(struct iio_dev *indio_dev);

    // iio_dev 注销
    void iio_device_unregister(struct iio_dev *indio_dev);

    // iio_dev 释放
    void iio_device_free(struct iio_dev *indio_dev);
    ```
- 驱动实现：
  1. `struct iio_info` 实现
    ```c
    // include/linux/iio/iio.h
    struct iio_info {
        struct module *driver_module;
        struct attribute_group *event_attrs;
        const struct attribute_group *attrs;

        int (*read_raw)(struct iio_dev *indio_dev, struct iio_chan_spec const *chan, int *val, int *val2, long mask);
        ...
        int (*write_raw)(struct iio_dev *indio_dev, struct iio_chan_spec const *chan, int val, int val2, long mask);
        int (*write_raw_get_fmt)(struct iio_dev *indio_dev, struct iio_chan_spec const *chan, long mask);
        ...
    }
    ```
    - `read_raw` 和 `write_raw` 读取和写入（配置时用到）原始数据；
    - `indio_dev`：需要读写的 IIO 设备；
    - `chan`：需要读取的通道（一个传感器可以输出多个数据，每个数据占用一个通道）；
    - `val` 和 `val2`：整数 + 小数（变换后的），数据组合形式：
      - `IIO_VAL_INT`：整数值，没有小数，不需要设置 `val2`；
      - `IIO_VAL_INT_PLUS_MICRO`：小数部分扩大 10^6 倍；
      - `IIO_VAL_INT_PLUS_NANO`：小数部分扩大 10^9 倍；
      - `IIO_VAL_INT_PLUS_MICRO_DB`：db 数据，和 `IIO_VAL_INT_PLUS_MICRO` 数据形式一样，只是在后面添加 db；
      - `IIO_VAL_INT_MULTIPLE`：多个整数值，比如一次要传回 6 个整数值，那么 `val` 和 `val2` 就不够用了。此宏主要用于 `iio_info` 的 `read_raw_multi()` 函数；
      - `IIO_VAL_FRACTIONAL`：分数值，也就是 val/val2；
      - `IIO_VAL_FRACTIONAL_LOG2`：值为 val >> val2。
    - `mask`：掩码，表示是什么数据，比如六轴传感器既有原始的测量数据（陀螺仪、加速度计，也有测量范围或者分辨率等）。至于每个通道可以采用哪几种掩码，在初始化通道的时候需要在驱动中设置好。
      - `IIO_CHAN_INFO_RAW`：原始数据；
      - `IIO_CHAN_INFO_SCALE`：分辨率。
    - `write_raw_get_fmt`：用于设置用户空间向内核空间写入的数据格式，它决定了 `val` 和 `val2` 的意义，即数据组合形式。
  2. 通道描述
    ```c
    // include/linux/iio/iio.h
    struct iio_chan_spec {
      enum iio_chan_type type;
      int channel;
      int channel2;
      unsigned long address;
      int scan_index;
      struct {
        char sign;
        u8 realbits;
        u8 storagebits;
        u8 shift;
        u8 repeat;
        enum iio_endian endianness;
      } scan_type;
      long info_mask_separate;
      long info_mask_shared_by_type;
      long info_mask_shared_by_dir;
      long info_mask_shared_by_all;
      const struct iio_event_spec *event_spec;
      unsigned int num_event_specs;
      const struct iio_chan_spec_ext_info *ext_info;
      const char *extend_name;
      const char *datasheet_name;
      unsigned modified:1;
      unsigned indexed:1;
      unsigned output:1;
      unsigned differential:1;
    };

    // 通道类型，type：
    // include/uapi/linux/iio/types.h
    enum iio_chan_type {
      IIO_VOLTAGE,    // 电压类型
      IIO_CURRENT,    // 电流类型
      IIO_POWER,      // 功率类型
      IIO_ACCEL,      // 加速度类型
      IIO_ANGL_VEL,   // 角速度类型（陀螺仪）
      IIO_MAGN,       // 电磁类型（磁力计）
      IIO_LIGHT,      // 灯光类型
      IIO_INTENSITY,  // 强度类型（光强传感器）
      IIO_PROXIMITY,  // 接近类型（接近传感器）
      IIO_TEMP,       // 温度类型
      IIO_INCLI,      // 倾角类型（倾角测量传感器）
      IIO_ROT,        // 旋转角度类型
      IIO_ANGL,       // 转动角度类型（电机旋转角度测量传感器）
      IIO_TIMESTAMP,  // 时间戳类型
      IIO_CAPACITANCE,  // 电容类型
      ...
    };

    // 当成员变量 indexed 为 1 时，channel 为通道索引
    // 当成员变量 modified 为 1 时，channel2 为通道修饰符：
    // 通道修饰符主要是影响 sysfs 下的通道文件名字
    // include/uapi/linux/iio/types.h
    enum iio_modifier {
      IIO_NO_MOD,
      IIO_MOD_X,    // x 轴
      IIO_MOD_Y,    // y 轴
      IIO_MOD_Z,    // z 轴
      ...
    };

    // address：一般会设置为通道对应的芯片数据寄存器地址。自定义，可用作其他功能，也可以不用。
    // scan_index: 当使用触发缓冲区时，sacn_index 是扫描索引。
    // scan_type: 描述扫描数据在缓冲区中的存储格式。sign: 'u'-无符号数据类型、's'-有符号；realbits: 数据有效位数；storagebits: 实际存储位数，有效位数 + 填充位；shift: 右移位数；repeat：实际或存储位的重复数量；endianness: 数据的大小端模式，IIO_CPU, IIO_BE, IIO_LE。

    // info_mask_separate 标记某些属性专属于此通道
    // include/linux/iio/types.h
    enum iio_chan_info_enum {
      IIO_CHAN_INFO_RAW = 0,    // 通道中的数据为原始数据
      IIO_CHAN_INFO_PROCESSED,
      IIO_CHAN_INFO_SCALE,
      IIO_CHAN_INFO_OFFSET,
      ...
      IIO_CHAN_INFO_DEBOUNCE_TIME,
    };

    // info_mask_shared_by_type: 标记导出的信息由相同类型的通道共享。
    // 例如，ICM20608 加速度计的 X、Y、Z 轴它们的 type 都是 IIO_ACCEL
    // 即类型相同。而这三个轴的分辨率（量程）是一样的，那么在配置这三个
    // 通道的时候就可以在 info_mask_shared_by_type 中使能 IIO_CHAN_INFO_SCALE 这个属性
    // 表示这三个通道的分辨率是共用的，这样在 sysfs 下就会只生成一个描述
    // 分辨率的文件，这三个通道都可以使用这个分辨率文件。

    // info_mask_shared_by_dir: 标记某些导出的信息由相同方向的通道共享
    // infO_mask_shared_by_all: 标记某些信息由所有的通道共享

    // modified 为 1 时，channel2 为通道修饰符
    // indexed 为 1 时，channel 为通道索引
    // output 表示为输出通道
    // differential 表示为差分通道
    ```
- 驱动代码
  ```c
  // IIO 设备申请和初始化
  struct xxx_dev {
    struct spi_device *spi;
    struct regmap *regmap;
    struct regmap_config regmap_config;
    struct mutex lock;
  }

  static const struct iio_chan_spec xxx_channels[] = {

  };

  // 当读取 sysfs 中的文件的时候，最终此函数会执行
  // 此函数读取数据然后传给应用
  static int xxx_read_raw(struct iio_dev *indio_dev, struct iio_chan_spec const *chan, int *val, int *val2, long mask) {
    // 借助 spi、i2c、regmap 读取数据
  }

  // 同上
  static int xxx_write_raw(struct iio_dev *indio_dev, struct iio_chan_spec const *chan, int *val, int *val2, long mask) {
    // 借助 spi、i2c、regmap 写入数据
  }

  // 同上
  static int xxx_write_raw_get_fmt(struct iio_dev *indio_dev, struct iio_chan_spec const *chan, long mask) {
    // 借助 spi、i2c、regmap 写入数据
  }

  static const struct iio_info xxx_info = {
    .read_raw = xxx_read_raw,             // 读文件时，此函数被调用
    .write_raw = xxx_write_raw,
    .write_raw_get_fmt = &xxx_write_raw_get_fmt,  // 用来指定用户空间写入的数据格式
  }

  // 以 spi 接口为例
  static int xxx_probe(struct spi_device *spi) {
    int ret;
    struct xxx_dev *data;
    struct iio_dev *indio_dev;

    indio_dev = devm_iio_device_alloc(&spi->dev, sizeof(*data));
    if (!indio_dev)
      return -ENOMEM;
    
    data = iio_priv(indio_dev);
    data->spi = spi;
    spi_set_drvdata(spi, indio_dev);
    mutex_init(&data->lock);

    indio_dev->dev.parent = &spi->dev;
    indio_dev->info = &xxx_info;
    indio_dev->name = "xxx";
    indio_dev->modes = INDIO_DIRECT_MODE;
    indio_dev->channels = xxx_channels;
    indio_dev->num_channels = ARRAY_SIZE(xxx_channels);

    iio_device_register(indio_dev);

    // regmap 相关设置

    // spi 相关设置

    // 芯片初始化

    return 0;
  }

  static int xxx_remove(struct spi_device *spi) {
    struct iio_dev *indio_dev = spi_get_drvdata(spi);
    struct xxx_dev *data;
    data = iio_priv(indio_dev);
    // 其他资源的注销及释放
    // 注销 IIO
    iio_device_unregister(indio_dev);   // 前面分配资源的时候使用了 devm，因此在这里不用手动释放 iio_dev
    return 0;
  }

  static const struct spi_device_id xxx_id[] = {
    {"xxx", 0},
    {/* sentinel */}
  };

  static const struct of_device_id xxx_of_match[] = {
    {.compatible = "xxx" },
    {/* sentinel */}
  };

  static struct spi_driver xxx_driver = {
    .probe = xxx_probe,
    .remove = xxx_remove,
    .driver = {
      .owner = THIS_MODULE,
      .name  = "xxx",
      .of_match_table = xxx_of_match,
    },
    .id_table = xxx_id,
  };

  static int __init xxx_init(void) {
    return spi_register_driver(&xxx_driver);
  }

  static void __exit xxx_exit(void) {
    spi_unregister_driver(&xxx_driver);
  }

  module_init(xxx_init);
  module_exit(xxx_exit);
  MODULE_LICENSE("GPL");
  ```
- 具体函数实现：
  - `xxx_read_raw` 函数：
    ```c
    #define ICM20608_TEMP_OFFSET    0
    #define ICM20608_TEMP_SCALE     326800000

    // icm20608 陀螺仪分辨率，对应 250、500、1000、2000，计算方法：
    // 以正负 250 度量程为例，500 / 2^16 = 0.007629，扩大 1000000 倍
    // 就是 7629
    static const int gyro_scale_icm20608[] = {
      7629,
      15258,
      30517,
      61035
    };

    // icm20608 加速度计分辨率，对应 2、4、8、16 计算方法：
    // 以正负 2g 量程为例，4 / 2^16 = 0.000061035，扩大 10^9，就是 61035
    static const int accel_scale_icm20608[] = {
      61035,
      122070,
      244140,
      488281,
    }

    static int icm20608_sensor_show(struct icm20608_dev *dev, int reg, int axis, int *val) {
      int ind, result,
      __be16 d;

      ind = (axis - IIO_MOD_X) * 2;
      result = regmap_bulk_read(dev->regmap, reg + ind, (u8 *)&d, 2);
      if (result)
        return -EINVAL;
      *val = (short)be16_to_cpup(&d);

      return IIO_VAL_INT;
    }

    static int icm20608_read_channel_data(struct iio_dev *indio_dev, struct iio_chan_spec const *chan, int *val) {
      struct icm20608_dev *dev = iio_priv(indio_dev);
      int ret = 0;
      switch (chan->type) {
      case IIO_ANGL_VEL:
        ret = icm20608_sensor_show(dev, ICM20_GYRO_XOUT_H, chan->channel2, val);
        break;
      case IIO_ACCEL:
        ret = icm20608_sensor_show(dev, ICM20_ACCEL_XOUT_H, chan->channel2, val);
        break;
      case IIO_TEMP:
        ret = icm20608_sensor_show(dev, ICM20_TEMP_OUT_H, IIO_MOD_X, val);
        break;
      default:
        ret = -EINVAL;
        break;
      }
      return ret;
    }

    static int icm20608_read_raw(struct iio_dev *indio_dev, struct iio_chan_spec const *chan, int *val, int *val2, long mask) {
      struct icm20608_dev *dev = iio_priv(indio_dev);
      int ret = 0;
      unsigned char regdata = 0;
      printk("icm20608_read_raw\n");

      switch (mask) {
      case IIO_CHAN_INFO_RAW:         // 读取加速度计、陀螺仪、温度传感器原始值
        iio_device_claim_direct_mode(indio_dev);  // 保持 direct 模式
        mutex_lock(&dev->lock);
        ret = icm20608_read_channel_data(indio_dev, chan, val);
        mutex_unlock(&dev->lock);
        iio_device_release_direct_mode(indio_dev);
        return ret;
      case IIO_CHAN_INFO_SCALE:
        switch (chan->type) {
        case IIO_ANGL_VEL:
          mutex_lock(&dev->lock);
          regdata = (icm20608_read_onereg(dev, ICM20_GYRO_CONFIG) & 0x18) >> 3;
          *val = 0;
          *val2 = gyro_scale_icm20608[regdata];
          mutex_unlock(&dev->lock);
          return IIO_VAL_INT_PLUS_MICRO; // 值为 val + val2 / 10^6
        case IIO_ACCEL:
          mutex_lock(&dev->lock);
          regdata = (icm20608_read_onereg(dev, ICM20_ACCEL_CONFIG) & 0x18) >> 3;
          *val = 0;
          *val2 = accel_scale_icm20608[regdata];
          mutex_lock(&dev->lock);
          return IIO_VAL_INT_PLUS_NANO; // 值为 val + val2 / 10^9
        case IIO_TEMP:
          *val = ICM20608_TEMP_SCALE / 1000000;
          *val2 = ICM20608_TEMP_SCALE % 1000000;
          return IIO_VAL_INT_PLUS_NICRO;  // 值为 val + val2 / 10^6
        default:
          return -INVAL;
        }
      case IIO_CHAN_INFO_OFFSET:      // ICM20608 温度传感器 offset 值
        switch (chan->type) {
        case IIO_TEMP:
          *val = ICM20608_TEMP_OFFSET;
          return IIO_VAL_INT;
        default:
          return -EINVAL;
        }
      case IIO_CHAN_INFO_CALIBBIAS:   // ICM20608 加速度计和陀螺仪校准值
        switch (chan->type) {
        case IIO_ANGL_VEL:
          mutex_lock(&dev->lock);
          ret = icm20608_sensor_show(dev, ICM20_XG_OFFS_USRH, chan->channel2, val);
          mutex_unlock(&dev->lock);
          return ret;
        case IIO_ACCEL:
          mutex_lock(&dev->lock);
          ret = icm20608_sensor_show(dev, ICM20_XA_OFFSET_H, chan->channel2, val);
          mutex_unlock(&dev->lock);
          return ret;
        default:
          return -EINVAL;
        }
      default:
        return -EINVAL;
      }
    }
    ```
  - `xxx_write_raw`
    ```c
    static int icm20608_sensor_set(struct imc20608_dev *dev, int reg, int axis, int val) {
      int ind, result;
      __be16 d = cpu_to_be16(val);

      ind = (axis - IIO_MOD_X) * 2;
      result = regmap_bulk_write(dev->regmap, reg + ind, (u8 *)&d, 2);
      if (result)
        return -EINVAL;
      
      return 0;
    }

    // val-量程/分辨率值
    static int icm20608_write_gyro_scale(struct icm20608_dev *dev, int val) {
      int result, i;
      u8 d;

      for (i = 0; i < ARRAY_SIZE(gyro_scale_icm20608); ++i) {
        if (gyro_scale_icm20608[i] == val) {
          d = (i << 3);
          result = regmap_write(dev->regmap, ICM20_GYRO_CONFIG, d);
          if (result)
            return result;
          return 0;
        }
      }
      
      return -EINVAL;
    }

    static int icm20608_write_accel_scale(struct icm20608_dev *dev, int val) {
      int result, i;
      u8 d;

      for (i = 0; i < ARRAY_SIZE(accel_scale_icm20608); ++i) {
        if (accel_scale_icm20608[i] == val) {
          d == (i << 3);
          result = regmap_write(dev->regmap, ICM20_ACCEL_CONFIG, d);
          if (result)
            return result;
          return 0;
        }
      }

      return -EINVAL;
    }

    static int icm20608_write_raw(struct iio_dev *indio_dev, struct iio_chan_spec const *chan, int val, int val2, long mask) {
      struct icm20608_dev *dev = iio_priv(indio_dev);
      int ret = 0;

      iio_device_claim_direct_mode(indio_dev);
      switch (mask) {
      case IIO_CHAN_INFO_SCALE: // 设置分辨率
        switch (chan->type) {
          case IIO_ANGL_VEL:    // 设置陀螺仪
            mutex_lock(&dev->lock);
            ret = icm20608_write_gyro_scale(dev, val2);
            mutex_unlock(&dev->lock);
            break;
          case IIO_ACCEL:
            mutex_lock(&dev->lock);
            ret = icm20608_write_accel_scale(dev, val2);
            mutex_unlock(&dev->lock);
            break;
          default:
            ret = -EINVAL;
            break;
        }
        break;
      case IIO_CHAN_INFO_CALIBBIAS: // 设置校准值
        switch (chan->type) {
        case IIO_ANGL_VEL:
          mutex_lock(&dev->lock);
          ret = icm20608_sensor_set(dev, ICM20_XG_OFFS_USRH, chan->channel2, val);
          mutex_unlock(&dev->lock);
          break;
        default:
          ret = -EINVAL;
          break;
        }
        break;
      default:
        ret = -EINVAL;
        break;
      }
      iio_device_release_direct_mode(indio_dev);
      return ret;
    }
    ```
  - `xxx_write_raw_get_fmt`:
    ```c
    static int icm20608_write_raw_get_fmt(struct iio_dev *indio_dev, struct iio_chan_spec const *chan, long mask) {
      switch (mask) {
      case IIO_CHAN_INFO_SCALE:
        switch (chan->type) {
        case IIO_ANGL_VEL:
          return IIO_VAL_INT_PLUS_MICRO;
        default:
          return IIO_VAL_INT_PLUS_NANO;
        }
      default:
        return IIO_VAL_INT_PLUS_MICRO;
      }
      return -EINVAL;
    }
    ```
    - 如果用户空间要设置是陀螺仪的比例，那么小数部分就要扩大 10^6 倍，其他就扩大 10^9 倍。比如，在用户空间要设置加速度计的量程为正负 4g，只需要向 in_accel_scale 写入 0.000122070，那么最终传入到驱动里面的就是 0.000122070 * 10^9 = 122070。
- APP：
  ```c
  #define SENSOR_FLOAT_DATA_GET(ret, index, str, member) \
    ret = file_data_read(file_path[index], str); \
    dev->member = atof(str);
  
  #define SENSOR_INT_DATA_GET(ret, index, str, member) \
    ret = file_data_read(file_path[index], str); \
    dev->member = atoi(str);
  
  static char file_path[] = {
    "/sys/bus/iio/devices/iio:device1/in_accel_scale",
    ...
  };

  enum path_index = {
    IN_ACCEL_SCALE = 0,
    ...
  };

  struct icm20608_dev {
    int accel_x_calibbias, ...
    ...
  }

  struct icm20608_dev icm20608;

  static int file_data_read(char *filename, char *str) {
    int ret = 0;
    FILE *data_stream;

    data_stream = fopen(filename, "r");
    if (data_stream == NULL) {
      printf("failed to open this file\n");
      return -1;
    }

    // 格式化读取数据，在遇到空格和换行符的时候就会结束
    // 返回：成功读取到的数据个数，如果读取到文件末尾或者读取错误就返回 EOF
    ret = fscanf(data_stream, "%s", str);
    if (!ret)
      printf("file read error\n");
    else if (ret == EOF)
      fseek(data_stream, 0, SEEK_SET);  // 读到文件末尾的话将文件指针重新调整到文件头
    fclose(data_stream);
    return 0;
  }

  static int sensor_read(struct icm20608_dev *dev) {
    int ret = 0;
    char str[50];

    // 1. 获取陀螺仪原始数据
    SENSOR_FLOAT_DATA_GET(ret, IN_ANGLVEL_SCALE, str, gyro_scale);
    ...
  }

  int main(int argc, char *argv[]) {
    int ret = 0;

    if (argc != 1) {
      return -1;
    }

    while (1) {
      ret = sensor_read(&icm20608);
      if (ret == 0) {
        // 读取数据成功，打印数据
      }
      usleep(100000); // 100ms
    }
    return 0;
  }
  ```
- 内核配置
  ```txt
  Device Drivers
    Industrial I/O support (IIO [=y])
      [*]Enable buffer support within IIO
      [*]Industrial I/O buffering based on kfifo
  ```
- 调试
  - 在 sysfs 中的文件：`sysfs/bus/iio/devices/xxx` 下
  - 文件名（通道属性）的命名模式：`[direction]_[type]_[index]_[modifier]_[info_mask]`
    ```c
    // direction
    static const char *const iio_direction[] = {
      [0] = "in",
      [1] = "out",
    };

    // type
    static const char *const iio_chan_type_name_spec[] = {
      [IIO_VOLTAGE] = "voltage",
      [IIO_CURRENT] = "current",
      [IIO_POWER]   = "power",
      [IIO_ACCEL]   = "accel",
      [IIO_ANGL_VEL] = "anglvel",
      [IIO_MAGN]    = "magn",
      ...
      [IIO_GRAVITY] = "gravity",
      [IIO_POSITIONRELATIVE] = "positionreletive",
      [IIO_PHASE] = "phase",
      [IIO_MASSCONCENTRATION] = "massconcentration",
    };

    // index
    // 如果配置通道的时候设置了 indexed = 1，那么就会使用通道的 channel
    // 成员变量来替代此部分命名。比如，有个 ADC 芯片支持 8 个通道，那么就
    // 可以使用 channel 来表示对应的通道，最终在用户空间呈现的每个通道文
    // 件名的 index 部分就是通道号。

    // modifier
    // 当通道的 modified 成员变量为 1 的时候，channel2 就是修饰符
    static const char * const iio_modifier_names[] = {
      [IIO_MOD_X] = "x",
      [IIO_MOD_Y] = "y",
      [IIO_MOD_Z] = "z",
      ...
      [IIO_MOD_PM4] = "pm4",
      [IIO_MOD_PM10] = "pm10",
    };

    // info_mask
    // 属性掩码，也即属性
    static const char * const iio_chan_info_postfix[] = {
      [IIO_CHAN_INFO_RAW] = "raw",
      [IIO_CHAN_INFO_PROCESSED] = "input",
      [IIO_CHAN_INFO_SCALE] = "scale",
      [IIO_CHAN_INFO_OFFSET] = "offset",
      [IIO_CHAN_INFO_CALIBSCALE] = "calibscale",
      [IIO_CHAN_INFO_CALIBBIAS] = "calibbias",
      ...
      [IIO_CHAN_INFO_DEBOUNCE_TIME] = "debounce_time",
      [IIO_CHAN_INFO_CALIBEMISSIVITY] = "calibemissivity",
      [IIO_CHAN_INFO_OVERSAMPLING_RATIO] = "oversampling_ratio",
    };
    ```
  - 通过 `cat` 命令查看文件数据

## 基于 ADC 的 IIO
- 驱动使能：
  ```txt
  Device Drivers
    Industrial I/O Support
      Analog to digital converters
        Freescale vf610 ADC driver
  ```
- 系统中的文件路径：`/sys/bus/iio/devices/iio:devicex`.
- 设备树绑定文件：`Documentation/devicetree/bindings/iio/adc/vf610-adc.txt`
  ```dts
  adc1: adc@02198000 {
    compatible = "fsl,imx6ul-adc", "fsl,vf610-adc";
    reg = <0x02198000 0x4000>;
    interrupts = <GIC_SPI 100 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&clks IMX6UL_CLK_ADC1>;
    num-channels = <2>;
    clock-names = "adc";
    status = "disable";
  };

  // pinctrl
  pinctrl_adc1: adc1grp {
    fsl,pins = <
      MX6UL_PAD_GPIO1_IO01__GPIO1_IO01 0xb0
    >;
  };

  // regulators
  reg_vref_adc: regulator@2 {
    compatible = "regulator-fixed";
    regulator-name = "VREF_3V3";
    regulator-min-microvolt = <3300000>;
    regulator-max-microvolt = <3300000>;
  }

  // 追加内容：
  &adc1 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_adc1>;
    num-channels = <2>;
    vref-supply = <&reg_vref_adc>;
    status = "okay";
  };
  ```
- 驱动源码文件：`drivers/iio/adc/vf610_adc.c`
  - platform + IIO 驱动框架；
  - ADC 外设设备结构体：
    ```c
    struct vf610_adc {
      struct device *dev;
      void __iomem *regs;
      struct clk *clk;

      u32 vref_uv;
      u32 value;
      struct regulator *vref;
      struct vf610_adc_feature adc_feature;

      u32 sample_freq_avail[5];

      struct completion completion;
    }
    ```
  - probe 函数（有省略）：
    ```c
    struct int vf610_adc_probe(struct platform_device *pdev) {
      struct vf610_adc *info;
      struct iio_dev *indio_dev;
      struct resource *mem;
      int irq;
      int ret;
      u32 channels;

      indio_dev = devm_iio_device_alloc(&pdev->dev, sizeof(struct vf610_adc));
      if (!indio_dev) {
        dev_err(&pdev->dev, "Failed allocating iio device\n");
        return -ENOMEM;
      }

      info = iio_priv(indio_dev);
      info->dev = &pdev->dev;

      mem = platform_get_resource(pdev, IORESOURCE_MEM, 0);
      info->regs = devm_ioremap_resource(&pdev->dev, mem);
      if (IS_ERR(info->regs))
        return PTR_ERRR(info->regs);

      irq = platform_get_irq(pdev, 0);
      if (irq < 0) {
        dev_err(&pdev->dev, "no irq resource?\n");
        return irq;
      }

      ret = devm_request_irq(info->dev, irq, 0, vf610_adc_irs, 0, dev_name(&pdev->dev), info);
      if (ret < 0) {
        dev_err(&pdev->dev, "failed requesting irq, irq = %d\n", irq);
      }

      info->clk = devm_clk_get(&pdev->dev, "adc");
      if (IS_ERR(info->clk)) {
        dev_err(&pdev->dev, "failed getting clock, err = %ld\n", PTR_ERR(info->clk));
        return PTR_ERR(info->clk);
      }

      info->vref = devm_regulator_get(&pdev->dev, "vref");
      if (IS_ERR(info->vref))
        return PTR_ERR(info->vref);

      ret = regulator_enable(info->vref);
      if (ret)
        return ret;
      
      info->vref_uv = regulator_get_voltage(info->vref);
      
      platfrom_set_drvdata(pdev, indio_dev);

      init_completion(&info->completion);

      ret = of_property_read_u32(pdev->dev.of_node, "num-channels", &channels);
      if (ret)
        channels = ARRAY_SIZE(vf610_adc_iio_channels);
      
      indio_dev->name = dev_name(&pdev->dev);
      indio_dev->dev.parent = &pdev->dev;
      indio_dev->dev.of_node = pdev->dev.of_node;
      indio_dev->info = &vf610_adc_iio_info;
      indio_dev->modes = INDIO_DIRECT_MODE;
      indio_dev->channels = vf610_adc_iio_channels;
      indio_dev->num_channels = (int)channels;

      ret = clk_prepare_enable(info->clk);
      if (ret) {
        dev_err(&pdev->dev, "Could not prepare or enable the clock\n");
        goto error_adc_clk_enable;
      }

      vf610_adc_cfg_init(info);
      vf610_adc_hw_init(info);

      ret = iio_device_register(indio_dev);
      if (ret) {
        dev_err(&pdev->dev, "Could not register the device.\n");
        goto error_iio_device_register;
      }

      return 0;
      ...
      return ret;
    }
    ```
    - 驱动分层，platform + IIO；对象分层，**上层的对象，其父对象是下层提供服务的对象**。
    - 通过 devm + pdev->dev + dev_err 可以方便地跳过错误处理中的资源释放部分？
  - `vf610_adc_iio_info`：
    ```c
    static int vf610_read_raw(struct iio_dev *indio_dev, struct iio_chan_spec const *chan, int *val, int *val2, long mask) {
      struct vf610_adc *info = iio_priv(indio_dev);
      unsigned int hc_cfg;
      long ret;

      switch (mask) {
        case IIO_CHAN_INFO_RAW:
        case IIO_CHAN_INFO_PROCESSED:
          mutex_lock(&indio_dev->mlock);
          reinit_completion(&info->completion);

          hc_cfg = VF610_ADC_ADCHC(chan->channel);
          hc_cfg |= VF610_ADC_AIEN;
          write1(hc_cfg, info->regs + VF610_REG_ADC_HC0);
          ret = wait_for_completion_interruptible_timeout(&info->completion, VF610_ADC_TIMEOUT);
          if (ret == 0) {
            mutex_unlock(&indio_dev->mlock);
            return -ETIMEDOUT;
          }
          if (ret < 0) {
            mutex_unlock(&indio_dev->mlock);
            return ret;
          }

          switch (chan->type) {
            case IIO_VOLTAGE:
              *val = info->value;
              break;
            case IIO_TEMP:
              // Calculate in degree Celsius times 1000
              // Using sensor slope of 1.84 mv/℃ and V at 25℃ of 696mv
              *val = 25000 - ((int)info->value - 864) * 1000000 / 1840;
              break;
            default:
              mutex_unlock(&indio_dev->mlock);
              return -EINVAL;
          }
          mutex_unlock(&indio_dev->mlock);
          return IIO_VAL_INT;
        case IIO_CHAN_INFO_SCALE:
          *val = info->vref_uv / 1000;
          *val = info->adc_feature.res_mode;
          return IIO_VAL_FRACTIONAL_LOG2;
        case IIO_CHAN_INFO_SAMP_FREQ:
          *val = info->sample_freq_avail[info->adc_feature.sample_rate];
          *val = 0;
          retur IIO_VAL_INT;
        default:
          break;
      }
      return -EINVAL;
    }

    static const struct iio_info vf610_adc_iio_info = {
      .driver_module = THIS_MODULE,
      .read_raw = &vf610_read_raw,
      .write_raw = &vf610_write_raw,
      .debugfs_reg_access = &vf610_adc_reg_access,
      .attrs = &vf610_attribute_group,
    };
    ```
  - `vf610_adc_isr`：
    ```c
    static irqreturn_t vf610_adc_isr(int irq, void *dev_id) {
      struct vf610_adc *info = (struct vf610_adc *)dev_id;
      int coco;
      
      coco = read1(info->regs + VF610_REG_ADC_HS);
      if (coco & VF610_ADC_HS_COCO0) {    // 检查 Convertion Complete 标志
        info->value = vf610_adc_read_data(info);  // 读取结果
        complete(&info->completion);              // 直接模式：唤醒等待的进程。对应的还有缓冲模式，它将读取到的数据推入 IIO 环形缓冲。
      }

      return IRQ_HANDLED;
    }
    ```

## todo
1. 数据类型 `__be16` 和函数 `be16_to_cpup()`、`cpu_to_be16()` 应该是和大小端数据相关，查找相关内容，并补充。
2. `iio_device_claim_direct_mode()` 和 `iio_device_release_direct_mode()` 函数的作用是什么？
3. `struct completion`
4. 在 vf610_adc.c 中并没有将 io 设置为 GPIO 输入、在设备树中并没有提供 interrupts 说明又是怎么得到 irq 的？、用户程序在读取文件时又是如何触发中断的呢？
   - 对于 i.MX6ULL 的 ADC，引脚不需要复用到某个数字外设功能。`MX6UL_PAD_GPIO1_IO01__GPIO1_IO01 0xb0` 中的 `0xb0` 是 pad 控制寄存器的配置值，主要设置：
     - 驱动强度
     - 速度
     - 上下拉
     - 最重要的是：关闭数字功能，使能模拟输入路径
     - 因此，不需要驱动代码干预。引脚配置完全由设备核心 + pinctrl 子系统完成（在 probe 之前）。驱动 vf610_adc.c 中没有任何 GPIO 相关代码，因为它不需要操作 GPIO。
   - 有 interrupts 属性（写的时候遗漏了）。
   - 并不是用户程序读取文件直接导致的中断，而是在 iio 驱动程序中的 `vf610_read_raw()` 函数启动了 ADC 转换，中断是 ADC 转换完成后又 ADC 设备发出的。开启 ADC 转换的命令：
      ```c
      hc_cfg = VF610_ADC_ADCHC(chan->channel);  // 选择通道号
      hc_cfg |= VF610_ADC_AIEN;                 // 使能转换完成中断
      write1(hc_cfg, info->regs + VF610_REG_ADC_HC0); // 写入 HC0 寄存器启动转换
      ```