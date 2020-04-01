# 关键结构体分析
相关链接：
[统一设备模型](https://www.cnblogs.com/wyk930511/p/7271462.html)
   
## 1. 内核对象kobject和sysfs
1. kref
   ```c
    struct kref {
	    atomic_t refcount;
    };
   ```
    1. kref只是包含一个原子量数据，初始化时会置成1，每调用一次，则加1；
    2. 通过对函数`kref_get`操作，可以获取kref被调用次数，如果小于2会发出警告；
    3. 通过对函数kref_put操作，可以判断资源是否有正确被释放；
   
2. 内核对象：kobject
   ```c
    struct kobject {
        const char		    *name;          // 该内核对象的名称
        struct list_head	entry;          // 链入kset的连接件
        struct kobject		*parent;        // 指向父对象，可以为空
        struct kset		    *kset;          // 指向的内核对象集合，可以为空
        struct kobj_type	*ktype;         // 该内核对象使用的操作集合
        struct kernfs_node	*sd;            /* sysfs directory entry */
        struct kref		    kref;           // 该内核对象的引用计数
    #ifdef CONFIG_DEBUG_KOBJECT_RELEASE
        struct delayed_work	release;
    #endif
        unsigned int        state_initialized:1;
        unsigned int        state_in_sysfs:1;
        unsigned int        state_add_uevent_sent:1;
        unsigned int        state_remove_uevent_sent:1;
        unsigned int        uevent_suppress:1;
    };
   ```
   1. kobject的功能：
      1. 实现结构的动态管理；
      2. 实现内核对象到sysfs的映射；
      3. 实现字定义属性的管理。
   2. 相关成员说明：
      1. name:该内核对象名字，想sysfs注册时，显示的目录名字；
      2. parent:指示了该内核对象在sysfs中的位置，如果有父类，则目录会创建在对应的父类下，如果没有，则创建在/sysfs下;
      3. ktype：包含了该内核对象的操作方法，包括前面提及的kref的自定义释放函数和自定义属性操作；
   3. 相关函数分析
      1. kobject_init：初始化内核对象，该函数通过调用kobject_init_internal对内核对象进行初始化，在函数kobject_init_internal里面，通过调用宏定义INIT_LIST_HEAD初始化一个链表头节点，并把链表前后都指向自己，但是该函数中，并没有对第二个参数kobj_type进行校验处理；
      2. kobject_put是内核对象的释放函数，对应于kref的kref_put，kobject_put仅仅是对kref_put的一个封装而已，向kref机制注册了kobject_release函数；
         1. container_of：获取kref内嵌的kobject对象
         2. release函数必须要实现
      3. kobject_add：在sysfs里创建层次化的目录，而这种层次都是parent带来的，逻辑上的父子概念，在sysfs内形成了目录的包含概念，其核心是kobject_add_internal
         1. create_dir：根据parent创建目录和之下的属性文件
      4. kobject_cleanup：最终会通过调用 t->release(kobj)来实现release函数的实现；
   
3. kset
   ```c
    struct kset {
        struct list_head list;
        spinlock_t list_lock;
        struct kobject kobj;
        const struct kset_uevent_ops *uevent_ops;
    };
   ```
    1. kset是kobject的一个再封装，从而使其具有kobject的功能以外，同时还添加了其他功能；
    2. kset_register的工作流程：
       1. kobject_init+kobject_add(kobject的基本流程)
       2. kobject_uevent: 通过环境变量发送一个uevent事件
    3. 总结：kobj通过parent的逻辑关系，形成了sysfs内的层级分明的目录。kset利用内嵌的kobj也形成了自己在sysfs内的层级分明的目录。但是这并不是kset本身的父子关系。kset的作用是对集合内的其他kobj有统一管理的能力。于是，kset就通过内嵌的kobj内的kset指针指向来确定自己的父亲。如果为NULL，表示自己不受管理。


## 2.统一设备模型  
> 说明：在统一设备模型里，内核利用kobj以及kset建立sysfs内的目录结构，并根据业务的不同，抽象出device、driver、bus、class等模型。几乎所有的内核驱动子系统都用到这一类模型。  
> 较低层次的模型：bus、subsys_interface、class、class_interface  
> 基于较低层次的模型：device、driver
1. bus结构体：bus_type
   1. bus_type
   ```c
    struct bus_type {
        const char		*name;                                  /* bus_name */
        const char		*dev_name;
        struct device		*dev_root;
        struct device_attribute	*dev_attrs;	/* use dev_groups instead */
        const struct attribute_group **bus_groups;
        const struct attribute_group **dev_groups;
        const struct attribute_group **drv_groups;

        int (*match)(struct device *dev, struct device_driver *drv);
        int (*uevent)(struct device *dev, struct kobj_uevent_env *env);
        int (*probe)(struct device *dev);
        int (*remove)(struct device *dev);
        void (*shutdown)(struct device *dev);

        /* main use is above. */
        int (*online)(struct device *dev);
        int (*offline)(struct device *dev);

        int (*suspend)(struct device *dev, pm_message_t state);
        int (*resume)(struct device *dev);

        const struct dev_pm_ops *pm;

        const struct iommu_ops *iommu_ops;

        struct subsys_private *p;
        struct lock_class_key lock_key;
    };
   ```
   1. 对于bus_type，其主要作用是注册（probe）、注销（remove）、匹配(match），以下分析关键操作函数：
      ```c
      bus_register
         priv->subsys.kobj.kset = bus_kset;
         retval = kset_register(&priv->subsys);             // 本质是kset_register
         priv->devices_kset = kset_create_and_add("devices", NULL, &priv->subsys.kobj);      // 生成/sys/bus/pci/devices文件夹
         priv->drivers_kset = kset_create_and_add("drivers", NULL, &priv->subsys.kobj);      // 生成/sys/bus/pci/devices文件夹

      kset_create_and_add        // kset的注册过程，归根到底还是对kset进行配置，比kset_register多了内存分配以及初始化(初始化parent)的过程
         kset_create             // 如果不加kset_create，该kset会创建在/sys目录下
            kzalloc
            kobject_set_name
         kset_register
      ```

2. subsys_private结构体
   1. subsys_private:用来记录在内核中关于bus的动态信息的，和bus_type结构体相对应。
   ```c
   struct subsys_private {
      struct kset subsys;                    // 子系统的内嵌kset
      struct kset *devices_kset;             // 该子系统管理的设备kset
      struct list_head interfaces;
      struct mutex mutex;

      struct kset *drivers_kset;             // 该子系统管理的驱动kset
      struct klist klist_devices;            // 该子系统下的所有device链表头
      struct klist klist_drivers;            // 该子系统下的所有driver链表头
      struct blocking_notifier_head bus_notifier;
      unsigned int drivers_autoprobe:1;
      struct bus_type *bus;                  // 与该子系统关联的bus

      struct kset glue_dirs;
      struct class *class;
   };
   ```
   2. interfaces：是一个链表头，作为管理`class/bus`
   
3. subsys_interface结构体
   1. subsys_interface：设备功能接口
   ```c
   struct subsys_interface {
      const char *name;
      struct bus_type *subsys;
      struct list_head node;
      int (*add_dev)(struct device *dev, struct subsys_interface *sif);
      void (*remove_dev)(struct device *dev, struct subsys_interface *sif);
   };
   ```
   2. `subsys_interface_register`通过`bus_get`获得`bus`类型，再通过`list_add_tail`把该对象加到对应的总线上
   3. `subsys_system_register`通过调用`subsys_register`注册subsystem到目录`/sys/devices/system/`中，同时在`bus_register`之后，也把总线生成一个设备，从而可以支持注册到别的总线上
4. class
   1. class与bus_type作用基本相同，但在新版内核中并没有看见struct class结构体的踪影

5. device和driver:整理从注册到probe成功的过程

