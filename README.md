# BL5340PA Manifest

This manifest repository contains the Laird Connectivity fork of the nRF Connect SDK [Version 2.4.1](https://developer.nordicsemi.com/nRF_Connect_SDK/doc/2.4.1/nrf/index.html) with support for the [BL5340PA](https://www.lairdconnect.com/wireless-modules/bluetooth-modules/bl5340pa-series-long-range-bluetooth-module).

This fork adds support for the BL5340PA DVK board. It also adds a driver for the [nRF21540](https://www.nordicsemi.com/products/nrf21540#:~:text=The%20nRF21540%20is%20a%20'plug,power%20short%2Drange%20wireless%20solutions) Front-End Module  used by the BL5340PA. The driver limits the RF output power based on region and antenna type. **This is required for compliance with regulatory requirements**. The driver also handles errata with the IO of the FEM. The Laird Connectivity driver works alongside the Nordic MPSL FEM driver.

Out-of-Tree Zephyr sample applications added by this manifest (bl5340pa_samples folder):
- sleepy advertiser
- throughput

## Known Issues

- L2-209 - Power Table for Coded s=2 not being used by Nordic MPSL FEM driver. The s=8 table is being used. The RF power for s=8 is less than s=2. Therefore, the module is still RF compliant, albeit with a lower power output than what is allowed.

- L2-221 - Central device is unable to connect to a peripheral using coded PHY when FEM is operating in GPIO+SPI mode. A hard fault occurs on the network processor of the nRF5340.

## Artifacts

A couple of example applications have been compiled for the BL5340PA development kit (DVK). There is an internal and external antenna variant of each application. The sleepy advertiser example has been compiled for North America (NA).

The nRF DTM PC application has been forked and rebuilt to reflect the power levels available for the nRF5340 (-1, -2, -3, -4, -5, -6, -7, -8, -12, -16, -20, -40). It can be installed by dragging the tgz file into nRF Connect. This application assumes the firmware was built with CONFIG_DTM_POWER_CONTROL_AUTOMATIC=n.

## Cloning Firmware

> **WARNING:** On Windows do not clone into a starting folder path longer than 12 characters or else the firmware will not build.

This is a Zephyr `west` manifest repository. To learn more about `west` see [here](https://developer.nordicsemi.com/nRF_Connect_SDK/doc/2.4.1/zephyr/develop/west/index.html#west).

To clone this repository properly use the `west` tool. To install `west` you will first need Python3. It is recommended to use Python virtual environments.

Install `west` using `pip3`:

```
pip3 install west
```

Once `west` is installed, clone this repository using `west init` and `west update`:

```
# Checkout the latest manifest on main
west init -m https://github.com/LairdCP/bl5340pa_manifest.git --mr main

# Pull all the source described in the manifest
west update
```
## Preparing to Build

If this is your first time working with a Zephyr project on your computer you should follow the [nRF getting started guide](https://developer.nordicsemi.com/nRF_Connect_SDK/doc/2.4.1/nrf/getting_started.html) to install all the tools.

v0.16.0 of the Zephyr SDK is recommended for building the firmware.

> **WARNING:** Before building the firmware, be sure to install all python dependencies with the following commands:

```
pip3 install -r zephyr/scripts/requirements.txt
pip3 install -r nrf/scripts/requirements.txt
pip3 install -r bootloader/mcuboot/scripts/requirements.txt
```

## Custom Board Files

The BL5340PA DVK board file can be used as a template for creating a custom board file. The BL5340PA DVK shares common items with the BL5340 DVK board file.  Another example is the nRF5340DK which uses the same processor as the BL5340.

### Pin forwarding

In the cpu application (cpuapp) device tree specification (DTS), the FEM pins must be forwarded to the network core.

```
&gpio_fwd {
	nrf21540-gpio-if {
		gpios = <&gpio0 4 0>, /* antenna select */
				<&gpio0 31 0>, /* rx-en-gpios */
				<&gpio0 30 0>, /* tx-en-gpios */
				<&gpio0 29 0>; /* pdn-gpios */
	};
	nrf21540-spi-if {
		gpios = <&gpio1 0 0>, /* clk */
				<&gpio1 1 0>, /* mosi */
				<&gpio1 5 0>, /* csn */
				<&gpio1 4 0>; /* miso */
	};
};
```

### FEM

The mode pin of the nRF21540 FEM is grounded on the BL5340PA module. 

The FEM driver uses the SPI bus. For this reason, UART0 is not available on the network core.

```
nrf_radio_fem: nrf21540_fem {
    compatible = "nordic,nrf21540-fem";
    rx-en-gpios = <&gpio0 31 GPIO_ACTIVE_HIGH>;
    tx-en-gpios = <&gpio0 30 GPIO_ACTIVE_HIGH>;
    pdn-gpios = <&gpio0 29 GPIO_ACTIVE_HIGH>;
    spi-if = <&nrf_radio_fem_spi>;
    supply-voltage-mv = <3300>;
};
```

### Antenna Type and Region

The board file specifies the antenna type and the region (power-table).

The default for the BL5340PA DVK is North America (FCC/IC) and external antenna.

```
lcz_fem: nrf21540_lcz_fem  {
    compatible = "lairdconnect,nrf21540-lcz-fem";
    ant-sel-gpios = <&gpio0 4 GPIO_ACTIVE_HIGH>;
    spi-clk-gpios = <&gpio1 0 GPIO_ACTIVE_HIGH>;
    spi-mosi-gpios = <&gpio1 1 GPIO_ACTIVE_HIGH>;
    spi-csn-gpios = <&gpio1 5 GPIO_ACTIVE_LOW>;
    spi-miso-gpios = <&gpio1 4 GPIO_ACTIVE_HIGH>;
    power-table = <0>;
};
```

Example for internal antenna Australia/NZ.
```
lcz_fem: nrf21540_lcz_fem  {
    compatible = "lairdconnect,nrf21540-lcz-fem";
    ant-sel-gpios = <&gpio0 4 GPIO_ACTIVE_HIGH>;
    spi-clk-gpios = <&gpio1 0 GPIO_ACTIVE_HIGH>;
    spi-mosi-gpios = <&gpio1 1 GPIO_ACTIVE_HIGH>;
    spi-csn-gpios = <&gpio1 5 GPIO_ACTIVE_LOW>;
    spi-miso-gpios = <&gpio1 4 GPIO_ACTIVE_HIGH>;
    internal-antenna;
    power-table = <1>;
};
```

The antenna type and region can also be specified on the command line or in the project configuration. These configurations must be applied to network application.

Select internal antenna.
```
west build -p -b bl5340pa_dvk_cpuapp -- -Dhci_rpmsg_CONFIG_LCZ_FEM_INTERNAL_ANTENNA=y
```

Select CE configuration on BL5340PA.
```
west build -p -b bl5340pa_dvk_cpuapp -- -Dhci_rpmsg_CONFIG_LCZ_FEM_REGION=2
```

# Module Integration

After the module has been integrated into a product, further testing may be required. For example, transmitter and receiver spurious emissions and receiver blocking. The DTM firmware can be used in those tests. Contact [Laird Connectivity](https://www.lairdconnect.com/services/emc-testing) for more information on getting your product certified.

The DTM firmware for the BL5340PA DVK can be used for preliminary testing, but the DTM application should be built using a board file for the product the module is installed on. Detailed instructions can be found [here](https://www.lairdconnect.com/documentation/application-note-using-the-nordic-nrf-connect-sdk-v2-x-with-visual-studio-code-ide-bl65x-and-bl5340).

## Regulatory Compliance and Direct Test Mode (DTM)

The BL5340PA module has been [certified](https://www.lairdconnect.com/documentation/regulatory-information-guide-bl5340pa-series) for use in different countries. The [DTM firmware](https://github.com/LairdCP/sdk-nrf/blob/bl5340pa/ncs2.4.1/samples/bluetooth/direct_test_mode/README.rst) was used to determine the maximum power level that the module can transmit at for each channel and frequency. The power levels used can be found in the [datasheet](https://www.lairdconnect.com/documentation/datasheet-bl5340pa-series) and in the [power tables](https://github.com/LairdCP/sdk-nrf/tree/bl5340pa/ncs2.4.1/drivers/lcz_fem/power_table).

A dedicated hardware tester, [PC application](https://www.nordicsemi.com/Products/Development-tools/nrf-connect-for-desktop), or terminal program can be used to control the DTM firmware. Testers use a serial protocol described by the Bluetooth specification with some vendor specific commands created by [Nordic](https://developer.nordicsemi.com/nRF_Connect_SDK/doc/2.4.1/nrf/samples/bluetooth/direct_test_mode/README.html).

### CE testing
 
Currently, the nRF Connect DTM PC application cannot be used to replicate the testing required for CE.

To replicate testing done for CE, a serial command must be sent to set the FEM gain to 18 dB. This is a register value of 23. The SoC output power is set to -16 for all channels and modulations.

The Python DTM [module](https://github.com/LairdCP/pydtm) provides a wrapper for sending these commands.

# Lowering RF Power from the Nominal 20 dBm

There are two methods for lowering the RF power output. One can be done at compile time and the other can be done at runtime. At this time, neither method applies to the CE version of firmware because its output power table is fixed.

## Compile Time

To lower the output power of the module change CONFIG_BT_CTLR_TX_PWR_ANTENNA=20 to a different value in the network cpu configuration file (e.g., CONFIG_BT_CTLR_TX_PWR_ANTENNA=12). The driver will automatically get as close to the desired value as possible (without exceeding it).It is important to note that all values are not possible because the resolution of the nRF5340 is limited.

## Run-time

In the network cpu configuration file set CONFIG_MPSL_FEM_NRF21540_RUNTIME_PA_GAIN_CONTROL=y. The power can then be set from the application using HCI commands. An example function called set_tx_power is provided in the throughput [sample](https://github.com/LairdCP/bl5340pa_samples/blob/main/throughput/src/main.c) which was taken from the HCI power control [example](https://github.com/LairdCP/sdk-zephyr/tree/bl5340pa/ncs2.4.1/samples/bluetooth/hci_pwr_ctrl).

