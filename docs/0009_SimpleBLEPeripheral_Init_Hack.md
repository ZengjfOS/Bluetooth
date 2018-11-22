# SimpleBLEPeripheral Init Hack

```C
void SimpleBLEPeripheral_Init( uint8 task_id )
{
    simpleBLEPeripheral_TaskID = task_id;

    // Setup the GAP
    /**
     * 1. TGAP_CONN_PAUSE_PERIPHERAL到底是什么意思？
     *   https://e2echina.ti.com/question_answer/wireless_connectivity/bluetooth/f/103/t/70278
     * 2. 这个和从机向主机发送参数更新请求有关系，就是说从建立连接开始，到发送参数更新请求至少要5s的时间。这个是保证连接的稳定性的一个参数。
     */
    VOID GAP_SetParamValue( TGAP_CONN_PAUSE_PERIPHERAL, DEFAULT_CONN_PAUSE_PERIPHERAL );

    // Setup the GAP Peripheral Role Profile
    {
#if defined( CC2540_MINIDK )
        // For the CC2540DK-MINI keyfob, device doesn't start advertising until button is pressed
        // 默认关闭广播功能
        uint8 initial_advertising_enable = FALSE;
#else
        // For other hardware platforms, device starts advertising upon initialization
        uint8 initial_advertising_enable = TRUE;
#endif

        // By setting this to zero, the device will go into the waiting state after
        // being discoverable for 30.72 second, and will not being advertising again
        // until the enabler is set back to TRUE
        // 开机32.72秒内没有被连接，后面就不进行广播，也就无法被发现了
        uint16 gapRole_AdvertOffTime = 0;

        uint8 enable_update_request = DEFAULT_ENABLE_UPDATE_REQUEST;            // TRUE
        uint16 desired_min_interval = DEFAULT_DESIRED_MIN_CONN_INTERVAL;        // 80
        uint16 desired_max_interval = DEFAULT_DESIRED_MAX_CONN_INTERVAL;        // 800
        uint16 desired_slave_latency = DEFAULT_DESIRED_SLAVE_LATENCY;           // 0
        uint16 desired_conn_timeout = DEFAULT_DESIRED_CONN_TIMEOUT;             // 1000

        // Set the GAP Role Parameters
        // 默认关闭广播功能
        GAPRole_SetParameter( GAPROLE_ADVERT_ENABLED, sizeof( uint8 ), &initial_advertising_enable );
        // 开机32.72秒内没有被连接，后面就不进行广播，也就无法被发现了
        GAPRole_SetParameter( GAPROLE_ADVERT_OFF_TIME, sizeof( uint16 ), &gapRole_AdvertOffTime );

        // 回应扫描请求的响应数据
        /**
         * // GAP - SCAN RSP data (max size = 31 bytes)
         * static uint8 scanRspData[] =
         * {
         *     // complete name
         *     0x14,   // length of this data
         *     GAP_ADTYPE_LOCAL_NAME_COMPLETE,                                  // 0x09
         *     0x53,   // 'S'
         *     0x69,   // 'i'
         *     0x6d,   // 'm'
         *     0x70,   // 'p'
         *     0x6c,   // 'l'
         *     0x65,   // 'e'
         *     0x42,   // 'B'
         *     0x4c,   // 'L'
         *     0x45,   // 'E'
         *     0x50,   // 'P'
         *     0x65,   // 'e'
         *     0x72,   // 'r'
         *     0x69,   // 'i'
         *     0x70,   // 'p'
         *     0x68,   // 'h'
         *     0x65,   // 'e'
         *     0x72,   // 'r'
         *     0x61,   // 'a'
         *     0x6c,   // 'l'
         *
         *     // connection interval range
         *     0x05,   // length of this data
         *     GAP_ADTYPE_SLAVE_CONN_INTERVAL_RANGE,                            // 0x12
         *     LO_UINT16( DEFAULT_DESIRED_MIN_CONN_INTERVAL ),   // 100ms       // 0x50
         *     HI_UINT16( DEFAULT_DESIRED_MIN_CONN_INTERVAL ),                  // 0x00
         *     LO_UINT16( DEFAULT_DESIRED_MAX_CONN_INTERVAL ),   // 1s          // 0x20
         *     HI_UINT16( DEFAULT_DESIRED_MAX_CONN_INTERVAL ),                  // 0x03
         *
         *     // Tx power level
         *     0x02,   // length of this data
         *     GAP_ADTYPE_POWER_LEVEL,                                          // 0x0A
         *     0       // 0dBm
         * };
         */
        GAPRole_SetParameter( GAPROLE_SCAN_RSP_DATA, sizeof ( scanRspData ), scanRspData );
        // 向外广播的数据
        /**
         * // GAP - Advertisement data (max size = 31 bytes, though this is
         * // best kept short to conserve power while advertisting)
         * static uint8 advertData[] =
         * {
         *     // Flags; this sets the device to use limited discoverable
         *     // mode (advertises for 30 seconds at a time) instead of general
         *     // discoverable mode (advertises indefinitely)
         *     0x02,   // length of this data
         *     GAP_ADTYPE_FLAGS,                                                        // 0x01
         *     DEFAULT_DISCOVERABLE_MODE | GAP_ADTYPE_FLAGS_BREDR_NOT_SUPPORTED,        // 0x01 | 0x04 = 0x05
         * 
         *     // service UUID, to notify central devices what services are included
         *     // in this peripheral
         *     0x03,   // length of this data
         *     GAP_ADTYPE_16BIT_MORE,      // some of the UUID's, but not all           // 0x02 
         *     LO_UINT16( SIMPLEPROFILE_SERV_UUID ),                                    // 0xF0
         *     HI_UINT16( SIMPLEPROFILE_SERV_UUID ),                                    // 0xFF
         * };
         */
        GAPRole_SetParameter( GAPROLE_ADVERT_DATA, sizeof( advertData ), advertData );

        // 连接失败自动调整连接参数
        GAPRole_SetParameter( GAPROLE_PARAM_UPDATE_ENABLE, sizeof( uint8 ), &enable_update_request );
        // 连接间隔，在BLE的两个设备的连接中使用跳频机制。
        GAPRole_SetParameter( GAPROLE_MIN_CONN_INTERVAL, sizeof( uint16 ), &desired_min_interval );
        GAPRole_SetParameter( GAPROLE_MAX_CONN_INTERVAL, sizeof( uint16 ), &desired_max_interval );
        // Slave latency to use for a param update
        GAPRole_SetParameter( GAPROLE_SLAVE_LATENCY, sizeof( uint16 ), &desired_slave_latency );
        // 超时管理——这个事两个成功连接事件之间的最大允许间隔。如果超过了这个事件而没有连接成功的连接事件，设备认为连接已经丢失。
        GAPRole_SetParameter( GAPROLE_TIMEOUT_MULTIPLIER, sizeof( uint16 ), &desired_conn_timeout );
    }

    // Set the GAP Characteristics
    // 修改设备名称：GENERIC ACCESS
    // The application now has the capability to change the permissions of the device name in the GAP service by calling GGS_SetParameter and changing the value of the parameter GGS_W_PERMIT_DEVICE_NAME_ATT.
    // GAP GATT服务器参数设置这一句修改的是service的名称
    GGS_SetParameter( GGS_DEVICE_NAME_ATT, GAP_DEVICE_NAME_LEN, attDeviceName );

    // Set advertising interval 
    // What is the advertising interval when device is discoverable
    {
        uint16 advInt = DEFAULT_ADVERTISING_INTERVAL;                           // 160

        GAP_SetParamValue( TGAP_LIM_DISC_ADV_INT_MIN, advInt );
        GAP_SetParamValue( TGAP_LIM_DISC_ADV_INT_MAX, advInt );
        GAP_SetParamValue( TGAP_GEN_DISC_ADV_INT_MIN, advInt );
        GAP_SetParamValue( TGAP_GEN_DISC_ADV_INT_MAX, advInt );
    }

    // Setup the GAP Bond Manager
    {
        uint32 passkey = 0; // passkey "000000"
        // Wait for a pairing request or slave security request
        uint8 pairMode = GAPBOND_PAIRING_MODE_WAIT_FOR_REQ;
        uint8 mitm = TRUE;
        uint8 ioCap = GAPBOND_IO_CAP_DISPLAY_ONLY;
        uint8 bonding = TRUE;
        //密钥,范围是0-999999,默认值为0
        GAPBondMgr_SetParameter( GAPBOND_DEFAULT_PASSCODE, sizeof ( uint32 ), &passkey );
        //告诉绑定管理器是否配对通过,不论它等待一个请求从控制设备或者是自己发起配对.默认的设置是等待一个请求从控制设备. 配对模式:配置成等待主机的配对请求
        GAPBondMgr_SetParameter( GAPBOND_PAIRING_MODE, sizeof ( uint8 ), &pairMode );
        // 设置中间人保护是否使能.如果使能了,配对请求将鉴定连接在从和主之间.profile默认的值为FALSE,即使应用设置它为TRUE在初始化的时候. 打开密钥保护的配对算法
        GAPBondMgr_SetParameter( GAPBOND_MITM_PROTECTION, sizeof ( uint8 ), &mitm );
        //告诉绑定管理器设备的输入和输出的能力.为了判断设备是否有显示屏或者输入键盘这个参数是需要的.然而,默认的值为GAPBOND_IO_CAP_DISPLAY_ONLY,表明设备有一个显示屏但没有键盘.即使设备没有物理意义上的显示器,一个展示的钥匙(在用户指导中)被认为是一个显示器.默认的万能钥匙是一个六位数字字符串”000000”.
        GAPBondMgr_SetParameter( GAPBOND_IO_CAPABILITIES, sizeof ( uint8 ), &ioCap );
        //使能绑定.profile默认的值为FALSE,即使SimpleBLEPeripheral应用设置它为TRUE在初始化时.
        GAPBondMgr_SetParameter( GAPBOND_BONDING_ENABLED, sizeof ( uint8 ), &bonding );
    }

    // 接下来就是注册5个Service了
    // Initialize GATT attributes
    GGS_AddService( GATT_ALL_SERVICES );            // GAP
    GATTServApp_AddService( GATT_ALL_SERVICES );    // GATT attributes
    DevInfo_AddService();                           // Device Information Service
    SimpleProfile_AddService( GATT_ALL_SERVICES );  // Simple GATT Profile
#if defined FEATURE_OAD
    VOID OADTarget_AddService();                    // OAD Profile
#endif

    // Setup the SimpleProfile Characteristic Values
    {
        uint8 charValue1 = 1;
        uint8 charValue2 = 2;
        uint8 charValue3 = 3;
        uint8 charValue4 = 4;
        uint8 charValue5[SIMPLEPROFILE_CHAR5_LEN] = { 1, 2, 3, 4, 5 };
        SimpleProfile_SetParameter( SIMPLEPROFILE_CHAR1, sizeof ( uint8 ), &charValue1 );
        SimpleProfile_SetParameter( SIMPLEPROFILE_CHAR2, sizeof ( uint8 ), &charValue2 );
        SimpleProfile_SetParameter( SIMPLEPROFILE_CHAR3, sizeof ( uint8 ), &charValue3 );
        SimpleProfile_SetParameter( SIMPLEPROFILE_CHAR4, sizeof ( uint8 ), &charValue4 );
        SimpleProfile_SetParameter( SIMPLEPROFILE_CHAR5, SIMPLEPROFILE_CHAR5_LEN, charValue5 );
    }


#if defined( CC2540_MINIDK )

    SK_AddService( GATT_ALL_SERVICES ); // Simple Keys Profile

    // Register for all key events - This app will handle all key events
    RegisterForKeys( simpleBLEPeripheral_TaskID );

    // makes sure LEDs are off
    HalLedSet( (HAL_LED_1 | HAL_LED_2), HAL_LED_MODE_OFF );

    // For keyfob board set GPIO pins into a power-optimized state
    // Note that there is still some leakage current from the buzzer,
    // accelerometer, LEDs, and buttons on the PCB.

    P0SEL = 0; // Configure Port 0 as GPIO
    P1SEL = 0; // Configure Port 1 as GPIO
    P2SEL = 0; // Configure Port 2 as GPIO

    P0DIR = 0xFC; // Port 0 pins P0.0 and P0.1 as input (buttons),
    // all others (P0.2-P0.7) as output
    P1DIR = 0xFF; // All port 1 pins (P1.0-P1.7) as output
    P2DIR = 0x1F; // All port 1 pins (P2.0-P2.4) as output

    P0 = 0x03; // All pins on port 0 to low except for P0.0 and P0.1 (buttons)
    P1 = 0;   // All pins on port 1 to low
    P2 = 0;   // All pins on port 2 to low

#endif // #if defined( CC2540_MINIDK )

#if (defined HAL_LCD) && (HAL_LCD == TRUE)

#if defined FEATURE_OAD
#if defined (HAL_IMAGE_A)
    HalLcdWriteStringValue( "BLE Peri-A", OAD_VER_NUM( _imgHdr.ver ), 16, HAL_LCD_LINE_1 );
#else
    HalLcdWriteStringValue( "BLE Peri-B", OAD_VER_NUM( _imgHdr.ver ), 16, HAL_LCD_LINE_1 );
#endif // HAL_IMAGE_A
#else
    HalLcdWriteString( "BLE Peripheral", HAL_LCD_LINE_1 );
#endif // FEATURE_OAD

#endif // (defined HAL_LCD) && (HAL_LCD == TRUE)

    // Register callback with SimpleGATTprofile
    // 作为GATT的server和client,主要通过Attribute来进行交互,当client请求server 读取数据时,通过如下注册的回调函数来进行访问。
    VOID SimpleProfile_RegisterAppCBs( &simpleBLEPeripheral_SimpleProfileCBs );

    // Enable clock divide on halt
    // This reduces active current while radio is active and CC254x MCU
    // is halted
    // BLE 协议栈里面还有一个关32MHz clock地方，需要透过下面API 去开关。 默认会在协议栈里面关闭32MHz，你的uart，timer1，3，4都可能间歇工作不正常.  调试PWM 信号的时候示波器出来总是不对。
    HCI_EXT_ClkDivOnHaltCmd( HCI_EXT_ENABLE_CLK_DIVIDE_ON_HALT );

#if defined ( DC_DC_P0_7 )

    // Enable stack to toggle bypass control on TPS62730 (DC/DC converter)
    HCI_EXT_MapPmIoPortCmd( HCI_EXT_PM_IO_PORT_P0, HCI_EXT_PM_IO_PORT_PIN7 );

#endif // defined ( DC_DC_P0_7 )

    // Setup a delayed profile startup， 发送事件
    osal_set_event( simpleBLEPeripheral_TaskID, SBP_START_DEVICE_EVT );

}
```
