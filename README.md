# Faster GPIOs on ESP8266

Do you use standard GPIO_OUTPUT_SET() macros for your bit-banging and do you want make it faster? Well, then you're at the right place. Continue reading and you'll learn how to get 6x faster GPIOs.

The way I see it, GPIO_OUTPUT_SET() was/is intended to be just for first initialization.
For controlling the output GPIOs in the code it's much better to skip all the unnecessary calling underlying the GPIO_OUTPUT_SET() and set output GPIO pins directly using GPIO_OUT_W1TS_ADDRESS / GPIO_OUT_W1TC_ADDRESS registers. The resulting code will be smaller and faster.

## Test A - using GPIO_OUTPUT_SET()

    PIN_FUNC_SELECT(PERIPHS_IO_MUX_MTDO_U, FUNC_GPIO15);
    GPIO_OUTPUT_SET(15, 1);
    //
    while(1){
        uint8_t i;
        for(i=0;i<20;i++) GPIO_OUTPUT_SET(15, 1);
        GPIO_OUTPUT_SET(15, 0);
        //
        vTaskDelay(1);
    }

![Test A](https://raw.githubusercontent.com/wdim0/esp8266_direct_gpio/master/test_A.jpg)

## Test B - using GPIO_OUT_W1TS/C directly

    PIN_FUNC_SELECT(PERIPHS_IO_MUX_MTDO_U, FUNC_GPIO15);
    GPIO_OUTPUT_SET(15, 1);
    //
    while(1){
        uint8_t i;
        for(i=0;i<20;i++) GPIO_REG_WRITE(GPIO_OUT_W1TS_ADDRESS, 1<<15);
        GPIO_REG_WRITE(GPIO_OUT_W1TC_ADDRESS, 1<<15);
        //
        vTaskDelay(1);
    }

![Test B](https://raw.githubusercontent.com/wdim0/esp8266_direct_gpio/master/test_B.jpg)

The timebase of the scope is set to 2 us per div in both test cases.<br />
Test A - impulse length is around 12 us.<br />
Test B - impulse length is around 1.9 us<br />
=> code B is around <b>6x faster</b> than A

Tests configuration: ESP8266 running at default 80 MHz, RTOS (SDK 1.4.0)
	
## GPIO_OUT_W1TS/C details

Each register is for all GPIOs - every bit corresponds to one GPIO (GPIO 0 is bit0, GPIO 1 is bit1, ...). You don't need to mask anything. If you just want to set one GPIO H/L, set the proper bit to 1, leave other bits to 0. Easy, fancy, fast.

GPIO_OUT_W1TS_ADDRESS - write 1 to set GPIO (high)

GPIO_OUT_W1TC_ADDRESS - write 1 to clear GPIO (low)

Reading these registers, will give you 0 - they are just for writing.

## Conclusion

It's definitely better to use direct GPIO control of output pins by writing to GPIO_OUT_W1TS_ADDRESS / GPIO_OUT_W1TC_ADDRESS registers. I've briefly looked into Espressif's GPIO driver (gpio.c, gpio.h) but there are no macros dedicated to this exactly. But anyway, it's so easy that we can do our own macros. To give you some hints:

    #define GPIO4           GPIO_INPUT_GET(4)
    
    #define GPIO2_H         GPIO_REG_WRITE(GPIO_OUT_W1TS_ADDRESS, 1<<2)
    #define GPIO2_L         GPIO_REG_WRITE(GPIO_OUT_W1TC_ADDRESS, 1<<2)
    #define GPIO2(x)        ((x)?GPIO2_H:GPIO2_L)
    
    ... init
    
    PIN_FUNC_SELECT(PERIPHS_IO_MUX_GPIO2_U, FUNC_GPIO2);
    GPIO_OUTPUT_SET(2, 0); //GPIO2 as output low
    //
    PIN_FUNC_SELECT(PERIPHS_IO_MUX_GPIO4_U, FUNC_GPIO4);
    GPIO_DIS_OUTPUT(4); //GPIO4 as input
    
    ... work with GPIOs
    
    GPIO2_H; //set GPIO2 high
    GPIO2_L; //set GPIO2 low
    GPIO2(1); //set GPIO2 high
    GPIO2(GPIO4); //set on GPIO2 the same level as is on GPIO4 input
