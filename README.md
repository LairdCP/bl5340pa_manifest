# BL5340PA Manifest

This manifest repository contains the Laird Connectivity fork of the nRF Connect SDK [Version 2.4.1](https://developer.nordicsemi.com/nRF_Connect_SDK/doc/2.4.1/nrf/index.html) with support for the [BL5340PA](https://www.lairdconnect.com/wireless-modules/bluetooth-modules/bl5340pa-series-long-range-bluetooth-module).

This fork adds support for the BL5340PA DVK board. It also adds a driver for the [nRF21540](https://www.nordicsemi.com/products/nrf21540#:~:text=The%20nRF21540%20is%20a%20'plug,power%20short%2Drange%20wireless%20solutions) Front-End Module  used by the BL5340PA. The driver limits the RF output power based on region and antenna type. This is required for compliance with regulatory requirements. The driver also handles errata with the IO of the FEM. The Laird Connectivity driver works alongside the Nordic MPSL FEM driver.

Out-of-Tree Zephyr sample applications added by this manifest (bl5340pa_samples folder):
- sleepy advertiser
- throughput

## Known Issues

- L2-209 - Power Table for Coded s=2 not being used by Nordic MPSL FEM driver. The s=8 table is being used. The RF power for s=8 is less than s=2. Therefore, the module is still RF compliant, albeit with a lower power output than what is allowed.

- L2-221 - Central device is unable to connect to a peripheral using coded PHY when FEM is operating in GPIO+SPI mode. A hard fault occurs on the network processor of the nRF5340.

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
``

Select CE configuration on BL5340PA.
```
west build -p -b bl5340pa_dvk_cpuapp -- -Dhci_rpmsg_CONFIG_LCZ_FEM_REGION=2
```
