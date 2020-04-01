# USB总线驱动
1. 刚插进USB时，系统会自动搜索USB设备，并提示“usb 1-1: new full-speed USB device number 3 using s3c2410-ohci”，通过在内核中查找该字符串位置确定输出该语句的函数，在反推调用该函数的上层函数，直到找到发现USB设备的函数即可以确定整个USB总线驱动流程。

```txt
hub_irq
    --检测插入中断，而后kick_khubd(hub);
    kick_khubd
        --wake_up(&khubd_wait);
        hub_thread(平时是休眠态，khubd_wait)
            --wait_event_freezable(khubd_wait,...);
            hub_events
                hub_port_connect_change
                    hub_port_init
```