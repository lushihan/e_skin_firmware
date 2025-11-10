# E-Skin Firmware Code Documentation
---


A comprehensive guide to the E-Skin firmware code, detailing its structure, functionality, and usage.

## Overview
The E-Skin firmware is designed to run on a PIC32 MZ EF microcontroller, enabling high-speed data acquisition from analog sensors and transmitting this data over USB. The firmware utilizes the built-in ADC capabilities of the microcontroller to sample data at a rate of 6.25 Msps, leveraging DMA for efficient data handling. The firmware uses MPLAB Harmony, the hardware abstraction layer for PIC32 microcontrollers. Harmony provides a graphical configurator to setup the peripherals, and generates the code to set the correct values into the registers, and provides functions to interact with the peripherals. This greatly simplifies development, as you do not have to search through the datasheet to figure out which registers need to be set to configure a peripheral. Harmony also provides a USB library, which greatly simplifys the USB code. 

## Code Structure
The firmware for this project follows the standard MPLAB Harmony project structure. The code is structured as a state machine, with the main application logic residing in the `app.c` file. The ```APP_Initialize``` function is called once at the start of the program, and the ```APP_Tasks``` function is called repeatedly in a loop. The ```appData``` structure is used to store the state of the application, as well as any data that needs to be shared between different parts of the code. The ```appData``` structure is defined in the `app.h` file.


## ADC (Analog to Digital converter)

### How it works
The purpose of an ADC is to convert an analog voltage to a digital value (basicaly measure the voltage). The circuitry required for this is somewhat complcated, and there are multiple architectures for ADCs. 

The PIC32 MZ EF has a Successive-approximation-register (SAR) ADC. SAR ADCs can be capable of up to 5Msps sampling rate and up to 18 bits of resolution, but the ADCs on the PIC32 have a 12 bit resolution with a 3.125Msps sampling rate in 12 bit mode. However, multiple ADCs can be interleaved by reading from different modules in succession. The SAR ADC works by conduncting a binary search to converge find the closest approximation of the input voltage. First, a sample-and-hold circuit captures the input voltage at one instant. Then a DAC (Digital to Analog converter) creates a reference voltage in the middle of the voltage range, and these 2 voltages are fed into a comparator. If the sample voltage is higher, the first result bit is set, and the reference voltage is increased. If the sample voltage is lower, the first bit is cleared and reference voltage is decreases. This process is repeated for each bit of resolution. This process takes some time, as the DAC and comparator have some settling time, and there is some logic required for each step. The time for each result to be ready determines the sampling rate of the ADC [More info on SAR ADCs Here](https://www.analog.com/en/resources/technical-articles/successive-approximation-registers-sar-and-flash-adcs.html)


 The PIC32MZ EF has 5 dedicated ADC modules (ADC0-4), and 1 shared module (ADC7). The dedicated modules are for high-speed and precise sampling, but can only be configured a primary or alternate pin. (alternate pins are not availalbe on the 64 pin version of the chip we will use in the final device). The shared ADC has a multiplexer and can be configured to many inputs, but is slower. Details of the PIC32MZ EF ADC can found in the ADC section of the [datasheet](https://ww1.microchip.com/downloads/aemDocuments/documents/MCU32/ProductDocuments/DataSheets/PIC32MZ-Embedded-Connectivity-with-Floating-Point-Unit-Family-Data-Sheet-DS60001320H.pdf)

In this project, we will use ADC2 and ADC4, since these are broken out on the dev board. We will run these 2 channels in interleaved mode at the full 12 bit resolution for a combined sampling rate of 6.25Msps. 

In order to collect this incoming data, we will use DMA (Direct Memory Access) to automatically move the data from the ADC result register into a buffer after every sample is collected. DMA allows peripherals to move data in and out of memory without the CPU doing anything, which is faster and frees up CPU time. When the ADC data is ready, it will trigger and interupt, and this interupt will trigger the DMA to transfer the ADC_DATA register into a buffer. In this case, the ADC interupt does not trigger an ISR (Interupt Service Routine), it just causes the DMA transfer. Once the DMA transfers a set amount of data, (ie, enough to fill up the buffer) another interupt will trigger to notify the program that the data is ready. 

Both ADCs require a trigger source, and must be in sync with each other. To do this we will use 2 more peripherals, a timer (TMR) and an output compare (OCMP). The timer will just be configured to have a frequency of 3.125Mhz, and this will trigger ADC2. We want ADC4 to trigger halfway between ADC2 trigger, so we can use a OCPM module to compare the value of the timer to a set value, and trigger ADC4 when they are equal. 



### Configuring the ADC in Harmony  
**Step 1:** Open the MCC (MPLAB Content Configurator) in MPLAB X IDE.

**Step 2:** In the "Device Resouces" tab, under "Harmony" and "Peripherals", select Drag ADCHS, TMR3, and OCMP3 into project graph.

**Step 3:** Configure the Pins: 
- ADC2: Set pin RB2 as AN2
- ADC4: Set pin RB4 as AN4

**Step 4:** Configure the ADCs:
- Set the clock source to "System Clock"
- For ADC2, set the following:
  - Enable: Yes
  - Resolution: 12 bits
  - Trigger Source: TMR3 Match

- For ADC4, set the following:
  - Enable: Yes
  - Resolution: 12 bits
  - Trigger Source: OCMP3
Leave the rest of the settings as default. Do not enable the interrupt, as this will enable and ISR for the ADC, which we do not want. We will manually enable the ADC interrupt in the code without enabling the ISR.

**Step 5:** Configure the TMR3:
- Use the 1:1 prescaler
- Set the clock to internal peripheral clock
- Set the Time to 320ns (3.125MHz)
    - This will set the period register to 30, based on the clock frequency

**Step 6:** Configure the OCMP3:
- Set the output mode to "Initalise OCx pin low, generate continuous pulses on OCx pin"
- Set the Timer source to Timer 3
- Set the compare value to 15, half of the period of TMR3, so that it triggers ADC4 halfway through the TMR3 period.
- Set the secondary compare value to 17, 2 cycles after the first value, so that it resets the output pin to low after the first pulse.

Note that there is no "output pin" for the OCMP3, but the pulse is used to trigger the ADC4.

**Step 7:** Configure the DMA:
- In the DMA configuration window, enable Channel 0, with trigger source set to "ADC2_DATA_2".

**Step 8:** Generate the code by clicking the "Generate" button in the MCC toolbar.

### ADC Code

#### Initialization

```c
#define ADC_VREF                (3.3f)
#define ADC_MAX_COUNT           (4095)
#define SAMPLE_LEN 383 // samples per channel, 2 channels = actually 766 samples
#define ADC_SRC_ADDR_2 (const void *)((&ADCDATA0) + ADCHS_CH2)
#define ADC_SRC_SIZE 3*sizeof(uint32_t) // *3 to capture ADCDATA2-ADCDATA4 in 1 transfer


ADCGIRQEN1bits.AGIEN2 = 1; //Enb ADC2 AN2 interrupts for DMA

//Setup DMA interupt callback and start a transfer
DMAC_ChannelCallbackRegister(DMAC_CHANNEL_0, ADC_2_ResultReadyCallback, 0);
DMAC_ChannelTransfer(DMAC_CHANNEL_0, ADC_SRC_ADDR_2, ADC_SRC_SIZE, adc_buf, sizeof (adc_buf), sizeof (uint32_t));

TMR3_Start(); //turn on Timer 3 to trigger ADC_2
OCMP3_Enable(); // turn on OCMP3 to trigger ADC_4
```

First we define some constants for the ADC, such as the reference voltage and maximum count. We also define the sample length, which is the number of samples we want to collect from each channel. The ADC_SRC_ADDR_2 is the address of the ADC data register for ADC2, and ADC_SRC_SIZE is the size of the transfer, which is 3 times the size of a uint32_t to capture all 3 ADC data registers (ADCDATA2-ADCDATA4) in one transfer. We need to capture all 3 registers because we are using channels 2 and 4, but need to capture a continuous block of data.

Then we enable the ADC2 interrupts for DMA, register the callback function for the DMA channel, and start the DMA transfer. Finally, we start the timer to trigger ADC3 and enable the output compare module to trigger the ADC4.

This initialization code is placed in the ```APP_Initialize``` function in the `app.c` file.

#### Callback Function
```c
static void ADC_2_ResultReadyCallback(DMAC_TRANSFER_EVENT event, uintptr_t contextHandle) {
    appData.adcDataReady = true;
}
```

This is the callback function that is called when the DMA transfer is complete. It sets a flag to indicate that the ADC data is ready to be processed. The `appData.adcDataReady` variable is used to signal the main application loop that new data is available.

This callback function is placed in the `app.c` file.


## Trigger signal interrupt

### How it works
The trigger signal interrupt is used to start the data acquisition process. When a trigger signal is received, the interrupt service routine (ISR) is called, which starts a new DMA transfer to collect the ADC data. The trigger signal is generated by the sequencer. 

### Configuring the trigger signal interrupt in Harmony
**Step 1:** Open the MCC (MPLAB Content Configurator) in MPLAB X IDE.

**Step 2:** In the "Pin Settings" tab, set pin RA1 (or a different pin on the PCB) as a GPIO input, and check the "Change Notification" box to enable interrupts on pin state change. You can also name the pin "TRIGGER" to make it easier to identify in the code.


### Interrupt Code

#### Initialization
```c
//setup trigger interupt
GPIO_PinInterruptCallbackRegister(TRIGGER_PIN, trigger_callback, (uintptr_t) NULL);
GPIO_PinIntEnable(TRIGGER_PIN, GPIO_INTERRUPT_ON_BOTH_EDGES);
```

First, we register the callback function for the trigger pin, and enable interrupts on both edges (rising and falling) of the pin. We actaully only need the falling edge, but for some reason the rising edge does not work, so we just trigger on both edges and check the pin state in the callback function.

This initialization code is placed in the ```APP_Initialize``` function in the `app.c` file.

#### Callback Function
```c
void trigger_callback(GPIO_PIN pin, uintptr_t context) {
    if (TRIGGER_Get() == 1) {
        //rising edge
    } else if (TRIGGER_Get() == 0) {
        DMAC_ChannelTransfer(DMAC_CHANNEL_0, ADC_SRC_ADDR_2, ADC_SRC_SIZE, adc_buf, sizeof (adc_buf), sizeof (uint32_t));
    }
}
```
This is the callback function that is called when the trigger pin state changes. It checks the state of the pin, and if it is low (falling edge), it starts a new DMA transfer to collect the ADC data. This will cause the ADCs to start sampling and filling up the buffer with new data. Once the buffer is full, the DMA transfer will complete and the ADC_2_ResultReadyCallback function will be called to signal that new data is available.

This callback function is placed in the `app.c` file.

## USB

### How it works
The USB module is used to transmit the collected ADC data to a host computer. The PIC32 MZ EF microcontroller has a built-in USB module that supports USB 2.0 Full Speed (12 Mbps) and High Speed (480 Mbps) modes. In this project, we will use the USB module in High Speed mode to transmit the ADC data. The USB module is configured as a Vendor-specific device, which means that it does not conform to any standard USB class (such as HID, CDC, etc.). This allows us to have more control over the data transfer process, but also requires us to write custom code on the host side to communicate with the device.

The USB module is configured to use the Bulk transfer mode, which is suitable for transferring large amounts of data. The Bulk transfer mode allows for error-free data transfer, but does not guarantee a specific data rate or latency. This is acceptable for our application, as we are only interested in transferring the ADC data as quickly as possible.


### Configuring the USB in Harmony
**Step 1:** Open the MCC (MPLAB Content Configurator) in MPLAB X IDE.
**Step 2:** In the "Device Resouces" tab, under "Harmony", "USB", "Device", select Drag USB Vendor Device Layer into project graph, this will automatically add the USB High Speed Driver, the USB Device Layer, and the Harmony Core modules to the project graph. No other configuration is needed, as the default settings are sufficient for our application. MCC will also try to add FreeRTOS and TIME to the project, but these are not needed, so remove them to prevent compilation errors. 

### USB Code

The USB code is a lot of boilerplate code that is generated by the MCC, so we will only focus on the parts that are relevant to our application.

#### Initialization
The ```APP_Initialize``` function sets the app state to ```APP_STATE_INIT```, and sets up some other appData variables. The USB module is initialized in the ```APP_Tasks``` function, when the app state is ```APP_STATE_INIT```. 

```c
switch (appData.state) {
    case APP_STATE_INIT:
        /* Open the device layer */
        appData.usbDevHandle = USB_DEVICE_Open(USB_DEVICE_INDEX_0, DRV_IO_INTENT_READWRITE);

        if (appData.usbDevHandle != USB_DEVICE_HANDLE_INVALID) {
            /* Register a callback with device layer to get event notification (for end point 0) */
            USB_DEVICE_EventHandlerSet(appData.usbDevHandle, APP_USBDeviceEventHandler, 0);

            appData.state = APP_STATE_WAIT_FOR_CONFIGURATION;
        } else {
            /* The Device Layer is not ready to be opened. We should try
             * again later. */
        }
        break;
}
```

This code opens the USB device layer, and registers a callback function to handle USB events. The app state is then set to ```APP_STATE_WAIT_FOR_CONFIGURATION```, which means that the device is waiting for the host to configure it. Once the device is configured by the host, some more initalization is done (See APP_Tasks -> case APP_STATE_WAIT_FOR_CONFIGURATION), and the app state is set to ```APP_STATE_MAIN_TASK```, and the main application loop can start.

#### USB Event Handler
The USB event handler is a callback function that is called by the USB device layer when certain events occur. The most important event for our application is the ```USB_DEVICE_EVENT_CONFIGURED``` event, which indicates that the device has been configured by the host. When this event occurs, we can start sending and receiving data over USB.

The ```USB_DEVICE_EVENT_ENDPOINT_WRITE_COMPLETE``` is also important, as it indicates that a write operation to an endpoint has completed. This is used to signal that the data has been sent to the host, and we can start sending more data. This Event will set the ```appData.epDataWritePending``` flag to false, which indicates that we can send more data.

The USB event handler is defined in the `app.c` file as ```APP_USBDeviceEventHandler```.

#### Main Task State
The main task state is where the main application logic resides. In this state, we check if the ```appData.epDataWritePending``` flag is false, which indicates that we can send more data over USB. We also check if the ADC data is ready by checking the ```appData.adcDataReady``` flag is true, and if it is, we send the data over USB.

To send the data over USB, we first copy the ADC data from the ```adc_buf``` to the ```transmitDataBuffer```. The ```adc_buf``` is a buffer of 32 bit integers containing the data from 3 ADC channels (ADCDATA2, ADCDATA3, ADCDATA4). We need to extract the data from each channel and convert it to an 8 bit integers, which is done by the following code. We only care about ADC2 and ADC4, so we ignore ADC3. 

```c
// copy ADC data into tx buf
for (int i = 0; i < SAMPLE_LEN; i++) {
    uint8_t input_voltage_2_low = (uint8_t) adc_buf[(i * 3)];
    uint8_t input_voltage_2_high = (uint8_t) (adc_buf[(i * 3)] >> 8);

    uint8_t input_voltage_4_low = (uint8_t) adc_buf[(i * 3) + 2];
    uint8_t input_voltage_4_high = (uint8_t) (adc_buf[(i * 3) + 2] >> 8);

    transmitDataBuffer[i * 4 + 2] = input_voltage_2_low;
    transmitDataBuffer[i * 4 + 3] = input_voltage_2_high;
    transmitDataBuffer[i * 4 + 4] = input_voltage_4_low;
    transmitDataBuffer[i * 4 + 5] = input_voltage_4_high;

}
```

Then we send the data over USB using the ```USB_DEVICE_EndpointWrite``` function. This function takes the USB device handle, the endpoint address, the data buffer, and the size of the data buffer as arguments. 

```c
/* Send the data to the host */
USB_DEVICE_EndpointWrite(
        appData.usbDevHandle,
        &appData.writeTranferHandle,
        appData.endpointTx, &transmitDataBuffer[0],
        sizeof (transmitDataBuffer),
        USB_DEVICE_TRANSFER_FLAGS_MORE_DATA_PENDING);
```

This will send the data to the host, and the ```appData.epDataWritePending``` flag will be set to true, indicating that we cannot send more data until the write operation is complete. The write operation will complete when the ```USB_DEVICE_EVENT_ENDPOINT_WRITE_COMPLETE``` event is received, which will set the ```appData.epDataWritePending``` flag to false, allowing us to send more data.

This main task state code is placed in the ```APP_Tasks``` function in the `app.c` file, under the case ```APP_STATE_MAIN_TASK```.


### Python Code
The python code is used to communicate with the USB device and read the data. The code uses the pyusb library to communicate with the USB device. The code is located in the `pyusb_code` directory.

There are 2 files, usb_file.py and usb_plot.py. The usb_file.py file is used to read the data from the USB device and save it to a file. The usb_plot.py file is used to read the data from the USB device and plot it in real-time. We will not go into detail about the plotting or file saving code, as it is not relevant to the firmware. The important part is the code that communicates with the USB device, which is the same in both files.

```python
import usb.core
import usb.backend.libusb1

last_ID = 0
ADC_SAMPLES = 766
ADC_PERIOD = 0.16 # [us]

def init_usb_device():
    # Adjust path to libusb as needed
    path_to_libusb = '/opt/homebrew/opt/libusb/lib/libusb-1.0.dylib'
    backend = usb.backend.libusb1.get_backend(find_library=lambda x: path_to_libusb)

    dev = usb.core.find(idVendor=0x04d8, idProduct=0x0053, backend=backend)

    if dev is None:
        raise ValueError('Device not found')

    dev.set_configuration()
    return dev
```

First we import the necessary libraries, and define some constants. The `init_usb_device` function is used to initialize the USB device. It uses the `usb.core.find` function to find the device with the specified vendor and product ID. If the device is not found, it raises a ValueError. If the device is found, it sets the configuration of the device and returns the device object. NOTE: You may need to adjust the path to the libusb library depending on your system. 


```python

def read_usb_data(dev):
    global last_ID
    try:
        data = dev.read(0x81, 1536, timeout=10)
    except usb.core.USBError as e:
        if e.errno == 60:
            #operation timed out
            raise ValueError("Operation timed out")
        elif e.errno == 19:
            raise ValueError("Device not found")

    channel_ID = data[0]
    packet_ID = data[1]

    last_ID = packet_ID

    adc_data = np.zeros(ADC_SAMPLES)
    for i in range(0, ADC_SAMPLES):
        adc_data[i] =  data[i * 2 + 2] + (data[i * 2 + 3] << 8)
    return adc_data

```

The `read_usb_data` function is used to read data from the USB device. It uses the `dev.read` function to read data from the endpoint 0x81 (the endpoint address for the IN endpoint). The function reads 1536 bytes of data, which is the size of the data buffer in the firmware. If the read operation times out, it raises a ValueError. If the device is not found, it raises a ValueError. The function then extracts the channel ID and packet ID from the first 2 bytes of the data, and stores the packet ID in the `last_ID` variable. It then extracts the ADC data from the rest of the data, and returns it as a numpy array.


