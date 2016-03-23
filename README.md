## Faster GPIOs on ESP8266 ##

Do you use standard GPIO_OUTPUT_SET() macros for your bit-banging and do you want make it faster? Well, then you're at the right place. Continue reading and you'll learn how to get 6x faster GPIOs.

The way I see it, GPIO_OUTPUT_SET() was/is intended to be just for first initialization.
For controlling the output GPIOs in the code it's much better to skip all the unnecessary calling underlying the GPIO_OUTPUT_SET() and set output GPIO pins directly using GPIO_OUT_W1TS/GPIO_OUT_W1TC registers. The resulting code will be smaller and faster.

## Test A - using GPIO_OUTPUT_SET() ##

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

## Test B - using GPIO_OUT_W1TS/GPIO_OUT_W1TC ##

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

The timebase of the scope is set to 2 us per div in both test cases.

Test A - impulse length is around 12 us.

Test B - impulse length is around 1.8 us

=> code B is around 6.5x faster than A

Tests configuration: ESP8266 running at default 80 MHz, RTOS (SDK 1.4.0)
	
## GPIO_OUT_W1TS/C details ##

Each register is for all GPIOs - every bit corresponds to one GPIO (GPIO 0 is bit0, GPIO 1 is bit1, ...)

GPIO_OUT_W1TS - write 1 to set GPIO (high)

GPIO_OUT_W1TC - write 1 to clear GPIO (low)

(when you read these registers, you'll get value 0) 


## Conclusion ##

It's definitely better to use direct GPIO control of output pins by writing to GPIO_OUT_W1TS/GPIO_OUT_W1TC registers. I've briefly looked into Espressif's GPIO driver (gpio.c, gpio.h) but there are no macros dedicated to this exactly. Anyway, it's so easy that it doesn't need any macro.
