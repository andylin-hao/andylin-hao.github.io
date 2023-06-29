---
title: VIRTIO-GPU Work Flow
date: 2020-07-15 17:51:04
permalink: /posts/virtio-gpu
tags: 
    - VIRTIO 
    - Emulator 
    - QEMU 
    - GPU
---

virtio-gpu属于virtio系列I/O虚拟化方案，采用类虚拟化设计，Guest端需要内置virtio驱动（>Linux 4.4均已配置，Android x86源码``kernel/drivers/virtio``中可查看）。virtio驱动的基本结构为前后端设计，前端为Guest OS驱动，后端为模拟器端逻辑，前后端通过virtqueue队列（由环形缓冲区vring实现）进行数据交换。

<img src="{{site.url}}/images/posts/virtio.gif">

## Guest端

* 操作系统启动时遍历PCI总线树，virtio-gpu作为一个PCI设备被OS检测到并注册

* ``virtio_pci_common.c``中的``virtio_pci_probe``函数会根据PCI设备号创建``virtio_pci_device``，并将其添加到``virtio_bus``

* 在``/sys/bus/virtio/drivers``中可以看到``virtio-gpu``设备

* 运行一个OpenGL应用，此时该应用会调用OpenGL库进行图形绘制

* OpenGL指令会送至内核中的DRM（Direct Rendering Management）图形驱动，DRM驱动向GPU下达相应的图形指令（着色、光栅化、贴图什么的），具体的OpenGL->DRM->GPU流程看这里https://www.cnblogs.com/shoemaker/p/linux_graphics01.html

* Guest端的virtio-gpu设备截获DRM的相关指令，然后转换分解为自己定义的一套指令（具体指令类型见后）

* 将指令放到virtqueue中，然后告知QEMU：

  ```c
  virtqueue_notify
      vq->notify(_vq) <-- vp_notify
      iowrite16(vq->index, vp_dev->ioaddr + VIRTIO_PCI_QUEUE_NOTIFY)
  ```

  方法是Guest写I/O寄存器，触发VM Exit进入到KVM中处理，进入到Host端流程

## Host端

* KVM无法处理而退出，处理退出原因发现是``KVM_EXIT_IO``，进而调用相应的Handle函数

  ```c
  int kvm_cpu_exec(CPUArchState *env)
  {
      do {
          run_ret = kvm_vcpu_ioctl(env, KVM_RUN, 0);
          switch (run->exit_reason) { /* 处理退出原因 */
          case KVM_EXIT_IO:
              kvm_handle_io();
              ...
  }
  ```

* ``kvm_handle_io``通过``write eventfd``将等待``poll``的QEMU主线程唤醒，然后一步步走到virtio-gpu的I/O handlers。其中与virtio-gpu有关的virtqueue有两个：``ctrl_vq``负责交换图形命令，``cursor_vq``负责交换光标信息；

  从而相应也有两个handlers：``virtio_gpu_handle_ctrl``和``virtio_gpu_handle_cursor``，两个函数在``hw/display/virtio-gpu.c``；

  handler的注册在GCC编译时设定为在main函数前执行，位于``virtio_gpu_device_realize``函数中，具体见QOM编程模型https://blog.csdn.net/u011364612/article/details/53581411和相关的GCC trickhttps://blog.csdn.net/u011364612/article/details/53581501

  ```c
  main  main_loop main_loop_wait
      os_host_main_loop_wait
          glib_pollfds_poll
              g_main_context_dispatch 
                  aio_ctx_dispatch    aio_dispatch
                      virtio_queue_host_notifier_read
                          virtio_queue_notify_vq 
                              virtio_gpu_handle_ctrl virtio_gpu_handle_cursor
  ```

* 以``virtio_gpu_handle_ctrl``继续向下，到``virtio_gpu_handle_ctrl``，其首先从virtqueue中取出Guest端传来的命令，然后进行处理

  ```c
  void virtio_gpu_process_cmdq(VirtIOGPU *g)
  {
      struct virtio_gpu_ctrl_command *cmd;
  
      while (!QTAILQ_EMPTY(&g->cmdq)) {
          /* 取命令 */
          cmd = QTAILQ_FIRST(&g->cmdq);
          
          ...
  
          /* 处理命令 */
          VIRGL(g, virtio_gpu_virgl_process_cmd, virtio_gpu_simple_process_cmd,
                g, cmd);
          QTAILQ_REMOVE(&g->cmdq, cmd, next);
          ...
      }
  }
  ```

  ``virtio_gpu_ctrl_command``为Guest端解析DRM驱动指令重新构成的virtio命令格式

  ```c
  struct virtio_gpu_ctrl_command {
      VirtQueueElement elem;                       // 取出的命令被映射到这里
      VirtQueue *vq;                               // Host/Guest共享的队列
      struct virtio_gpu_ctrl_hdr cmd_hdr;          // 命令指定的header，指定了命令类型
      uint32_t error;
      bool finished;
      QTAILQ_ENTRY(virtio_gpu_ctrl_command) next;
  };
  ```

* 处理命令的``VIRGL``宏如下，这里用了``libvirglrenderer``

  ```c
  #ifdef CONFIG_VIRGL
  #include <virglrenderer.h>
  #define VIRGL(_g, _virgl, _simple, ...)                     \
      do {                                                    \
          if (_g->parent_obj.use_virgl_renderer) {            \
              _virgl(__VA_ARGS__);                            \
          } else {                                            \
              _simple(__VA_ARGS__);                           \
          }                                                   \
      } while (0)
  #else
  #define VIRGL(_g, _virgl, _simple, ...)                 \
      do {                                                \
          _simple(__VA_ARGS__);                           \
      } while (0)
  #endif
  ```

  有``virgl``支持则能进行3D加速，否则只能进行2D加速，在Windows上目前直接使用``virtio-vga``会黑屏可能是已经没有所谓的2D加速这种东西了吧

* ``_simple``对应``virtio_gpu_simple_process_cmd``，3D加速对应的函数差不多

  ```c
  static void virtio_gpu_simple_process_cmd(VirtIOGPU *g,
                                            struct virtio_gpu_ctrl_command *cmd)
  {
      VIRTIO_GPU_FILL_CMD(cmd->cmd_hdr);
      virtio_gpu_ctrl_hdr_bswap(&cmd->cmd_hdr);
  
      switch (cmd->cmd_hdr.type) { //type就是命令类型
      case VIRTIO_GPU_CMD_GET_DISPLAY_INFO:
          virtio_gpu_get_display_info(g, cmd);
          break;
      case VIRTIO_GPU_CMD_GET_EDID:
          virtio_gpu_get_edid(g, cmd);
          break;
      case VIRTIO_GPU_CMD_RESOURCE_CREATE_2D:          // 创建2D资源，仅有长宽和格式，没有像素信息，像素信息单独传
          virtio_gpu_resource_create_2d(g, cmd);
          break;
      case VIRTIO_GPU_CMD_RESOURCE_UNREF:
          virtio_gpu_resource_unref(g, cmd);
          break;
      case VIRTIO_GPU_CMD_RESOURCE_FLUSH:             // 将处理好的图形送到显示器
          virtio_gpu_resource_flush(g, cmd);
          break;
      case VIRTIO_GPU_CMD_TRANSFER_TO_HOST_2D:        // 交给Host的pixman库进行图形处理，比如明暗、光栅化处理
          virtio_gpu_transfer_to_host_2d(g, cmd);
          break;
      case VIRTIO_GPU_CMD_SET_SCANOUT:
          virtio_gpu_set_scanout(g, cmd);
          break;
      case VIRTIO_GPU_CMD_RESOURCE_ATTACH_BACKING:    // 获取像素信息并和2D资源绑定
          virtio_gpu_resource_attach_backing(g, cmd);
          break;
      case VIRTIO_GPU_CMD_RESOURCE_DETACH_BACKING:
          virtio_gpu_resource_detach_backing(g, cmd);
          break;
      default:
          cmd->error = VIRTIO_GPU_RESP_ERR_UNSPEC;
          break;
      }
      if (!cmd->finished) {
          virtio_gpu_ctrl_response_nodata(g, cmd, cmd->error ? cmd->error :
                                          VIRTIO_GPU_RESP_OK_NODATA);
      }
  }
  ```

  具体的各流程和结构体解释见https://blog.csdn.net/huang987246510/article/details/106254294/

* ``pixman``库处理完后由``dpy_gfx_update``刷新显示器