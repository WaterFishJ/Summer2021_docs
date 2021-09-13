# pca10056的bsal层bas_sample分析

### 一、简介

​	此sample包含了bas服务，在bsal_srv_bas.c文件实现，在ble_bas_only_app.c文件被启动。因此我们需要逐步分析以上两个文件。

​	参考资料有：[SPEC](https://www.bluetooth.com/specifications/specs/)中的BAS，[assigned number](https://www.bluetooth.com/specifications/assigned-numbers/)，还有bsal_app_api.h等



### 二、ble_bas_only_app.c

**bsal_bas_blufi_app函数：**

```c
int bsal_bas_blufi_app(void)
{
    void *stack_ptr = bsal_find_stack_ptr(BSAL_STACK_NAME);
    void *bsal_test_app_task;
    if (stack_ptr == NULL)
    {
        //print error;
        return 1;
    }
    //set iocapability


    bsal_stack_ptr  = stack_ptr;

    //1. stack init
    bsal_stack_init(stack_ptr, bsal_app_all_callback);  // init param not start stack
    char *device_name = "ble_rtt";
    bsal_set_device_name(stack_ptr, strlen(device_name), (uint8_t *)device_name);
    //2. set bond type
    bsal_set_device_le_bond_type(stack_ptr, false, BSAL_NO_INPUT, BSAL_NO_OUTPUT, BSAL_GAP_AUTHEN_BIT_NO_BONDING, false);
    //set the bond flag:
    //3. start add profile service
    bsal_stack_le_srv_begin(stack_ptr, 1, bsal_app_profile_callback); //will add 1 service

    //4. add profile service
    bsal_le_bas_svr_init(stack_ptr, bsal_app_profile_callback); //add battery servcie

    //5. end add profile service
    bsal_stack_le_srv_end(stack_ptr);    //end srv add

    //6. start ble stack
    bsal_stack_startup(stack_ptr);    //start she

    //bsal_ble_loop(NULL);

    bsal_test_app_task =  rt_thread_create("bsal_app", bsal_ble_loop, NULL, 2 * 256, 5, 10);
    if (bsal_test_app_task != RT_NULL)
    {

        rt_thread_startup(bsal_test_app_task);
    }
    return 0;
}
```

- **void *bsal_find_stack_ptr(char *stack_name)**：根据协议栈名，查找并返回相应协议栈指针。宏 BSAL_STACK_NAME 在menuconfig内配置为了 "nimble"。
- **int bsal_stack_init(void *stack_ptr, void *callback)**：初始化协议栈，并将bsal_app_all_callback函数绑定到协议栈，这样协议栈下的连接，断开等操作将会进入到回调函数。
- **int bsal_set_device_name(void *stack_ptr, uint8_t length, uint8_t *name)**：设置设备（指协议栈）名字。
- **uint16_t bsal_set_device_le_bond_type(void *stack_ptr, bool is_bond, uint8_t input, uint8_t output, uint8_t bond_type, bool oob_enable)**：<u>暂时不清楚</u>，但这条函数在 bsal_stack_init 中调用过，这里应该可以不用再调用。
- **void bsal_stack_le_srv_begin(void *stack_ptr, uint8_t num, void *p_fun_cb)**：表示开始注册 profile 到协议栈，num 表示 profile 的数量，此处用到回调函数的<u>原因不明</u>。
- **void bsal_le_bas_svr_init(void *stack_ptr, void *app_callback)**：这条函数是初始化 bas service ，并绑定了一个回调函数。这条函数在 bsal_srv_bas.c 实现（后面再讲）。
- **void bsal_stack_le_srv_end(void *stack_ptr)**：表示停止注册 profile 到协议栈。
- **void bsal_stack_startup(void *stack_ptr)**：启动协议栈，此时，所有 service 随之启动



**bsal_app_all_callback函数：**

```c
void bsal_app_all_callback(void *stack_ptr, uint8_t cb_layer, uint16_t cb_sub_event, uint8_t value_length , void *value)
```

函数原型如上所示。此条回调函数是协议栈相关的，用户不会主动调用，我们只需要关注其中 cb_layer 和 cb_sub_event 这两个参数。

- cb_layer：

  ```c
  switch (cb_layer)
  {
      case BSAL_CB_LAYER_GAP:
          break;
      case BSAL_CB_LAYER_GATT_PROFILE:
          break;
      case BSAL_CB_LAYER_SM:
          break;
      case BSAL_CB_LAYER_COMMON:
          break;
    case BSAL_CB_LAYER_UNKNOWN:
          break;
      default:
          break;
  }
  ```
  
  其中，除了 BSAL_CB_LAYER_GAP 以外，其他 case 均未被调用（可能是还没实现），因此我们仅需关注这个 case。
  
- cb_sub_event：

  ```c
  switch (cb_layer)
  {
      case BSAL_CB_LAYER_GAP:
          switch (cb_sub_event)
          {
               case BSAL_CB_STACK_READY:
                  bsa_app_set_adv_data(stack_ptr);
                  bsal_stack_start_adv(stack_ptr);
                  break;
               case BSAL_CB_CONNECT_STATUS:
                  if (bsal_gap_msg_data->gap_conn_state_change.new_state == BSAL_GAP_CONN_STATE_CONNECTED)
                  {
                      bsal_app_conn_handle = bsal_gap_msg_data->gap_conn_state_change.conn_id;
                  }
                  else if (bsal_gap_msg_data->gap_conn_state_change.new_state == BSAL_GAP_CONN_STATE_DISCONNECTED)
                  {
                      bsal_stack_start_adv(stack_ptr);
                  }
                  break;
               default:
                  break;
          }
      default:
  }
  ```

  可以看到，cb_sub_event 有两个 case ，一个是 BSAL_CB_STACK_READY 时，则会进行配置广播包，然后开始打广播；另一个是 BSAL_CB_CONNECT_STATUS ，当 ble 连接成功后，记录下连接设备的 id ，当 ble 断开后，重新开始打广播。



**bsal_app_profile_callback函数：**

```c
void bsal_app_profile_callback(void *p)
```

函数原型如上所示。这条callback函数我认为更多是为 app 服务，因为 profile 的功能其实在各自 profile_callback 里已经可以实现。

这里的回调函数主要功能部分是对 battery service 的 CCCD 标志位做了一个记录，给 app 判断是否需要 notify。

```c
if (bsal_param->msg_type == BSAL_CALLBACK_TYPE_INDIFICATION_NOTIFICATION)
{
    uint16_t  cccbits = (uint16_t)bsal_param->value;
    if (bsal_param->srv_uuid.u16.value == BSAL_GATT_SERVICE_BATTERY_SERVICE)
    {
        if (cccbits & BSAL_GATT_CCC_NOTIFY)
        {
            battery_flag = 1;
        }
        else
        {
            battery_flag = 0;
        }
    }
}
```



**bsal_ble_loop函数：**

这条函数就是属于 app 的函数了，计算电量值，并调用 bsal_srv_bas.c 的 api 发送电量数据。



### 三、bsal_srv_bas.c

**bsal_le_bas_svr_init函数：**

```c
void bsal_le_bas_svr_init(void *stack_ptr, void *app_callback)
```

这条函数主要做了两个操作，一个是构造一个 profile 的结构体，另一个就是将结构体注册进协议栈，并绑定回调函数。

```c
struct bsal_gatt_app_srv_def ble_svc_bas_defs[] =
{
    //profile的所有参数
}
bsal_stack_le_srv_reg_func(stack_ptr, &ble_svc_bas_defs, (P_SRV_GENERAL_CB *)profile_callback);
pfn_bas_cb = (P_SRV_GENERAL_CB)app_callback;
```

profile 的结构体参考 BAS 规格书做配置，因此不展开细说。

pfn_bas_cb 将会被 profile_callback 调用。也就是说，bsal_app_profile_callback 函数会被这里的 profile_callback 调用。



**profile_callback函数：**

这里分别处理了四种协议栈传递过来的消息类型：

```c
typedef enum
{
    BSAL_CALLBACK_TYPE_READ_CHAR_VALUE = 1,              /**< client read event */
    BSAL_CALLBACK_TYPE_WRITE_CHAR_VALUE = 2,             /**< client write event */
    BSAL_CALLBACK_TYPE_INDIFICATION_NOTIFICATION = 3,    /**< CCCD update event */
    BSAL_CALLBACK_TYPE_HANDLE_TABLE = 4,                 /**< handle event */
} T_BSAL_SRV_CB_TYPE;
```

当消息类型是读数值时，判断 handle 是否为 battery level 的 handle（注意这里的 off_handle 是一个偏移量，因此 GATT_SVC_BAS_BATTERY_LEVEL_INDEX + 1 为实际 handle 值），是则发送电池电量数据：

```c
if (p_param->msg_type == BSAL_CALLBACK_TYPE_READ_CHAR_VALUE)
{
    if (GATT_SVC_BAS_BATTERY_LEVEL_INDEX == p_param->off_handle)
    {
        is_app_cb = true;
        //write db the battery_level
        bsal_srv_write_data(p_param->stack_ptr, p_param->start_handle, p_param->off_handle, sizeof(uint8_t), &bsal_bas_battery_level);
    }
}
```

当消息类型是 CCCD 更新时，同理，判断 handle ，并做相应操作（这里没有什么特别操作，<u>消息长度为啥为2暂时不清楚，反正抓包是看到为2</u>）：

```c
else if (p_param->msg_type == BSAL_CALLBACK_TYPE_INDIFICATION_NOTIFICATION)
{
    if (GATT_SVC_HRS_MEASUREMENT_CHAR_CCCD_INDEX == p_param->off_handle)
    {
        if (p_param->length == 2)
        {
            is_app_cb = true;
        }
    }
}
```

其他两个消息类型，BAS 没有用到（写特性可有可无，具体看规格书描述）。



**bsal_bas_send_notify_level函数：**

```c
void bsal_bas_send_notify_level(void *stack_ptr, uint16_t conn_id,  uint8_t battery_level)
{
    /*TODO*/
    if (battery_level > 100)
    {
        bsal_osif_printf_err("The battery is invalid:%d", battery_level);
        return;
    }
    bsal_uuid_any_t uuid_srv;
    uuid_srv.u_type = 16;
    uuid_srv.u16.value = GATT_UUID_BATTERY;
    uint16_t start_handle = bsal_srv_get_start_handle(stack_ptr, uuid_srv);
    bsal_bas_battery_level = battery_level;
    bsal_srv_send_notify_data(stack_ptr, conn_id, start_handle, GATT_SVC_BAS_BATTERY_LEVEL_INDEX, sizeof(uint8_t), &battery_level);
}
```

- **uint16_t bsal_srv_get_start_handle(void *stack_ptr, bsal_uuid_any_t uuid)**：在协议栈中寻找对应的 uuid ，并返回该 uuid 在协议栈的绝对 handle。
- **int bsal_srv_send_notify_data(void *stack_ptr, uint16_t conn_id, uint16_t start_handle, uint16_t offset_handle, uint16_t value_length, uint8_t *value)**：向设备发送数据，这一步有点类似向 spi 通信的传感器发送数据，修改寄存器配置一样，首先选择设备片选（conn_id），然后发送寄存器起始地址（start_handle）和偏移量（off_handle），最后发送数据（value_length 和 *value）



### 四、总结

bas_sample是一个比较简单，很典型的例程。例程中展示了在 bsal 层中，如何初始化协议栈，如何构造 profile ，如何注册 profile，如何处理协议栈回调等。参考此 sample ，便能很容易实现 HRS（Heart Rate Service），NUS（Nordic Uart Service）（均已实现）。同时，例程全程没有使用到协议栈相关 api 或宏，体现了 bsal 层向下屏蔽协议栈，可兼容多家MCU的协议栈的特点，使应用更具有可移植性。