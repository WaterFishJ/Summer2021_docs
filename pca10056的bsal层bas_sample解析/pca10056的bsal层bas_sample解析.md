# pca10056的bsal层bas_sample分析——bas&blufi

### 一、简介

​	此sample包含了bas服务和blufi服务，分别在bsal_srv_bas.c文件和bsal_srv_blufi.c文件实现，在ble_bas_blufi_app.c文件被一同启动。因此我们需要逐步分析以上三个文件。

​	参考资料有：[SPEC](https://www.bluetooth.com/specifications/specs/)中的BAS，[assigned number](https://www.bluetooth.com/specifications/assigned-numbers/)，还有bsal_app_api.h等



### 二、bas_blufi_app.c

bsal_bas_blufi_app函数：

```c
int bsal_bas_blufi_app(void)
{
    void *stack_ptr = bsal_find_stack_ptr(BSAL_STACK_NAME);
    if (stack_ptr == NULL)
    {
        //print error;
        return 1;
    }
    //set iocapability


    bsal_stack_ptr  = stack_ptr;
    //1. init stack
    bsal_stack_init(stack_ptr, bsal_app_all_callback);  // init param not start stack
    // set device name
    char *device_name = "ble_rtt";
    bsal_set_device_name(stack_ptr, strlen(device_name), (uint8_t *)device_name);
    //2. bond type
    bsal_set_device_le_bond_type(stack_ptr, false, BSAL_NO_INPUT, BSAL_NO_OUTPUT, BSAL_GAP_AUTHEN_BIT_NO_BONDING, false);
    //set the bond flag:

    //3. service begin
    bsal_stack_le_srv_begin(stack_ptr, 2, bsal_app_profile_callback);  //will add 1 service
    //4. bas_init
    bsal_le_bas_svr_init(stack_ptr, bsal_app_profile_callback); //add battery servcie

    //5. blufi_init
    bsal_le_blufi_svr_init(stack_ptr, bsal_app_profile_callback);

    //6. srv_end
    bsal_stack_le_srv_end(stack_ptr);    //end srv add

    //start stack
    bsal_stack_startup(stack_ptr);    //start she

    return 0;
}
```

- **void *bsal_find_stack_ptr(char *stack_name)**：根据协议栈名，查找并返回相应协议栈指针。宏 BSAL_STACK_NAME 在menuconfig内配置为了 "nimble"。
- **int bsal_stack_init(void *stack_ptr, void *callback)**：初始化协议栈，并将bsal_app_all_callback函数绑定到协议栈，这样协议栈下的连接，断开等操作将会进入到回调函数。
- **int bsal_set_device_name(void *stack_ptr, uint8_t length, uint8_t *name)**：设置设备（指协议栈）名字。
- **uint16_t bsal_set_device_le_bond_type(void *stack_ptr, bool is_bond, uint8_t input, uint8_t output, uint8_t bond_type, bool oob_enable)**：<u>暂时不清楚</u>，但这条函数在 bsal_stack_init 中调用过，这里应该可以不用再调用。
- **void bsal_stack_le_srv_begin(void *stack_ptr, uint8_t num, void *p_fun_cb)**：表示开始注册 profile 到协议栈，num 表示 profile 的数量，此处用到回调函数的<u>原因不明</u>。
- **void bsal_le_bas_svr_init(void *stack_ptr, void *app_callback)；void bsal_le_blufi_svr_init(void *stack_ptr, void *app_callback)**这两天函数分别是初始化 bas service 和 blufi service ，并绑定了同一个回调函数。这两条函数分别在 bsal_srv_bas.c 和 bsal_srv_blufi.c 实现（后面再讲）。
- **void bsal_stack_le_srv_end(void *stack_ptr)**：表示停止注册 profile 到协议栈。
- **void bsal_stack_startup(void *stack_ptr)**：启动协议栈，此时，所有 service 随之启动

bsal_app_all_callback函数：

