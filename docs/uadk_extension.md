
## 背景

UADK 2.0的用户态框架整体分为算法抽象层，驱动层，libwd基础层。但是功能上还不完善，一个是第三方驱动
注册还没有考虑，第二个是指令实现作为一种第三方的实现，如何集成到UADK框架中。

第一个是顺其自然，现阶段要考虑补充设计。第二个支持指令这个需求，是否符合UADK的框架支持的范围?
反对的认为这不是硬件加速器，UADK不应该支持，UADK应该只聚焦到硬件加速器这类硬件。但是从整体上
看，CPU core也符合UADK设备的语义，即CPU与设备共享页表，越来越多的crypto extension指令加入到CPU
core里，可以看做一种免内核驱动的“设备对象”;另外从趋势上看，加速器的编程接口也朝着指令化方向发展。

考虑如何去做这两件事，实际上内核态crypto框架已经做了一个sample，它已经考虑到各种加速器，指令，
软算库等实现，各种算法实现通通注册到crypto，对外只提供统一的接口。

我们需要考虑的事情有哪些：
  1、算法层与驱动层面的函数接口定义，提供驱动注册接口。
  2、算法层是否足够抽象，而不与底层的各种硬件实现上存在耦合。
  3、算法层包含各个算法实体的管理，查询和选择。即同一个算法，用户可以在多个实现之间选择。
  4、原则上一个时间内，一个算法只会选择一种实现，是否需要考虑到两种实现（算力实体）之间的切换？

## 用户case

case1:单个加速器硬件注册到UADK。
  - 拿SM4算法举例，有支持SM4算法的加速器硬件A通过UACCE注册到框架，且只有这一种实现注册给算法层，
用户调用算法层API进行SM4加解密操作，这时候走的加速器硬件A。

case2:单个CPU专有指令注册到UADK。
  - 同样支持SM4算法，免内核驱动，bypass kernel和UACCE，用户态直接注册到UADK，且只有这一种实现。
用户调用算法层API，这时候走的CPU专有指令。

case3:加速器硬件和CPU专有指令均注册到UADK。支持SM4算法有两种实现，这里考虑3种场景：
  - 1、两种注册到UADK算法层时标注priority值，priority越大表示优先级越高，这时候用户调用
  算法层API，默认选择最高优先级的实现。一般硬件加速器的优先级为最高。
  - 2、用户调用API初始化，比如就只想用专有指令，需要传入priority的值，这时候算法层内部
  走对应priority的指令实现。但是有点麻烦，需要提供更多的接口比如用户获取算法的各种priority，
  还要把这个值传进来。与其这么麻烦，还不如卸载掉加速器硬件的驱动，这时候退化为case2，这会更简单。
  - 3、但还有一种场景，两种实现都会同时用到，比如openssl进程1将加速器硬件算力占满，
  再起一个进程2，这时候就需要使用指令算力。这个时候就存在异构算力协同。很明显，
  进程它不感知是谁在撑起它的业务，也不需要调用另外一套软算API接口。

  问题：算力切换需要上层或者适配层感知处理，还是丢给UADK来自行切换处理？
  - 在UADK处理一方观点：上层编程简单，完全无感，分布式存储等应用组件都有这种大带宽算力的需求。
  - 在上层处理一方观点：1、上层业务只想明确跑到硬件加速器，因为会更安全。2、大带宽可能是个伪需求。

权衡来看，这里提供一个设计思路，倾向在上层来选择，但是UADK提供机制。比如接口提供一个参数:工作模式，
work_mode {0， 1};0或者默认值，表示固定使用单一最高优先级算力；1表示按照优先级从高到低寻找可用的
算力切换，找到可用的算力，当前进程固定使用该算力直至进程结束。work_mode这里用降级fall_back表示
可能更明确。

但是这个参数，如何嵌入到已有的UADK init接口？可能需要考虑新增init接口。另外可以先预留，真正需要的
时候再考虑实现。

## API proposal

### ALG API
我们先梳理下原有的流程：
1. APP首先调用wd_request_ctx(uacce_dev)申请硬件设备的句柄资源，而uacce_dev需要提前调用
wd_get_accel_list(alg_name)来拿到dev_list。

这里即暴露了硬件ctx handle概念，又暴露了uacce_dev设备。且dev-list用户拿到了怎么选择设备？如果
用户绑numa，就从最近设备申请，如果不绑numa则只有均衡选择，因为线程会迁移到其他cpu上，但这样会浪费
ctx，所以不管绑不绑，都建议从最近numa设备申请，当然只跑一个设备，总带宽会上不去，所以调性能可以
使用环境变量去配置从哪几个numa申请。如果这点看，其实完全没必要对APP暴露uacce_dev，用户只要看着
算法名申请即可，剩下的就交给算法层内部完成。

2. 原有的流程拿到ctx后，就需要申请和实例化调度器。之所以要调度器，初衷是APP初始化看不到硬件资源，
进程它只有看到ctx handle（会发现这里ctx handle没有必要暴露给用户，用户后面是基于session handle
去下发）， 它只是下发request，而进程申请了一组ctx资源（有多个类型：numa域，同步的，异步的，压缩的，
解压缩），同类型的又有很多ctx，这时候进程或者线程的req需要调度器来下发到不同类型的ctx上去执行任务。
  对于用户需要调用的接口如下： 
  wd_sched_alloc(sched_type, op_type_num, numa_num, poll_cb)用来申请一个调度器结构体的内存初始化。  
  wd_sched_instance(sched, param)接着用来实例化填充一个调度器各个调度参数。  
  wd_sched_release(sched)释放接口。  

  以上函数的实现均在wd_sched.c，同时内部提供了一个sample rr调度器，用户调用wd_sched_alloc指定定
sched_type类型为RR type即可使用内部默认的调度器。在这个文件里可以不断拓充新的调度器的实现。
  新增一个调度器，需要内部实现3个回调：  
```c
     .sched_init(sched_ctx_h, sched_param)  
     .pick_next_ctx(sched_ctx_h, sched_key, sched_mode)  
     .poll_policy(sched_ctx_h, expect, count)  
```

3. 上面两步做完还不够，做任务前，还需要具体的算法init，如压缩 wd_comp_init(ctx_config, sched)，传
入前面两个步骤拿到的ctx的属性配置及调度器，完成一系列初始化和bind，驱动，sched，msg pool等初始化。

4. 上面只是做进程资源初始化，实际做任务需要申请session，wd_comp_alloc_sess(setup)，拿到session的
handle，后续的做任务都基于这个handle进行。

5. 具体do_comp(session, req)的任务操作接口。这个比较直接。

考虑对上述初始化步骤1,2,3封装后的流程：
1. wd_comp_init2(ctx_setup);
```c
   struct ctx_setup {  
	alg_name;   /* 根据算法名去搜索可用对象，挂接driver ops */  
	direction;  /* 压缩还是解压缩，还是dual mode */  
	op_mode;    /* 同步还是异步 */  
	fall_back;  /* 是否切换，0：不降级切换。 1：算力资源耗尽，搜索切换下一级算力*/  
	sched_type; /* 调度器类型，默认RR调度 */  
   };
```
  
  需要生成和维护一个comp_alg list，所有的算力对象注册到本list，那list怎么区分uacce设备和cpu core？  

2. alloc_session(set_up)，基本不变。

3. do_comp()，也基本不变。

### udriver register API
实现动态注册，考虑两件事情，一个comp_alg定义，一个动态注册机制。

#### comp_alg定义

```c
  struct comp_alg {  
	alg_name;    /* udrv里定义支持所有的算法字符串, 如“zlib/ngzip/nzstd” */  
	driver_name;  
	priority;  
	device_list;  
	ops {.init .send .recv .uninit}  
  };  
```

#### 动态注册机制
目的是算法层的so部署了不会动，可以新增udriver的so，并且可以动态注册到算法层使用。
这样可以单方面升级和新增udriver。方便拓展和解耦部署。

主要两步：
1. 算法层增加dlopen操作，遍历open所有的udriver so。并提供register接口。
2. udriver 使用register接口注册到全局comp_alg list。

在udriver使用constructor attribute，在加载so时可以自动执行probe函数。  
```c
void __attrbute___((constructor)) xxdrv_probe()  
{  
  /* 如果本drv是硬件加速器，首先判断是否存在uacce设备，不存在直接返回  
   * 如果是cpu指令，最好也要判断下本平台是否支持才能往下走。  
   */  

  /* xxdrv_comp_alg 注册到算法层全局comp alg list*/  
  wd_comp_register();  

}
```

遍历所有可用的driver so，注册到comp alg list，并选择最高priority drv_ops赋值。


## function flow
怎么把整个流程串起来？从上面设计看ctx和sched都收到内部。至于收到哪一层？算法层需不需要感知ctx handle
和sched？

放在驱动层的好处是算法层变的简单，各家硬件实现不一样，可能没法抽象出公共的调度模型。不好的是复杂
留给各个vedor驱动自己内部去管理调度。

放在算法层的话，驱动层就很薄，需要算法层要吃下所有的逻辑和抽象。

调度不是当前考虑拓展要优先解决的问题，目前还是把调度维持在算法层去看下大致的流程：

```c
 ctx_setup->alg_name = "zlib";   /* 算法大类?还是精确到算法zlib? */  
 ctx_setup->direction = COMPRESS;  
 ctx_setup->op_mode = SYNC;  
 ctx_setup->sched_type = RR;  
+wd_comp_init2(ctx_setup);  
	--> dlopen(xx_drv);注册到全局comp alg list;  
        --> sort_comp_alg_list(alg_name);  
	--> set_ops_handle(list_head);完成搜索挂接ops。  
	--> comp alg 是cpu算力，则无需后续的sched调度。如果是硬件加速器算力，继续。  
	--> dev_list = wd_get_accl_list(alg_name); 怎么区分支持同算法的两个不同加速器硬件？  
	--> ctx[] = wd_request_ctx(dev);  
	--> wd_sched_init(), cache_init();  
	--> ops->drv_int();  

 setup->alg_type = ZLIB;  
 setup->comp_lv = level9;  
 setup->win_sz = 32k;  
+wd_alloc_session(setup);  

+wd_do_comp_sync(sess_handle, req);

+wd_free_session(sess_handle);

+wd_comp_uninit();
	--> ops->drv_uninit();  
	--> wd_sched_free();  
	--> ctx_free();  
	--> comp_list_free();  
	--> dlclose();  
```
