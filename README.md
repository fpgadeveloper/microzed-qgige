microzed-qgige
==============

### This repo has moved

This repo has been merged into the [Ethernet FMC Zynq GEM repo](https://github.com/fpgadeveloper/ethernet-fmc-zynq-gem "Ethernet FMC Zynq GEM repo")
in an effort to group similar example designs into a common repository and simplify code maintenance. Please use the linked
repository for the latest sources.

Example design for the [Quad Gigabit Ethernet FMC](http://ethernetfmc.com "Ethernet FMC") on the [MicroZed](http://microzed.org "MicroZed")
and [MicroZed FMC Carrier](http://zedboard.org/product/microzed-fmc-carrier "MicroZed FMC Carrier").

![Ethernet FMC and MicroZed](http://ethernetfmc.com/wp-content/uploads/2015/07/robust-ethernet-fmc-microzed-4.jpg "Ethernet FMC and MicroZed")

### Description

This project demonstrates the use of the Opsero [Quad Gigabit Ethernet FMC](http://ethernetfmc.com "Robust Ethernet FMC").
The design contains 3 soft Tri-mode Ethernet MACs plus one RGMII-to-GMII
IP core to make use of the spare hard Ethernet MAC in the PS of the Zynq.
Each of the 3 soft Ethernet MACs are configured with DMAs.

![Ethernet FMC Quad Gig AXI Ethernet](http://ethernetfmc.com/wp-content/uploads/2014/10/qgige_gmii_to_rgmii.png "Zynq Quad Gig Ethernet All AXI Ethernet")

### Requirements

* Vivado 2016.1 (see Library modifications below)
* [Robust Ethernet FMC](http://ethernetfmc.com "Robust Ethernet FMC")
* [MicroZed 7Z010 or 7Z020](http://microzed.org "MicroZed")
* [MicroZed FMC Carrier](http://zedboard.org/product/microzed-fmc-carrier "MicroZed FMC Carrier")
* [Xilinx Soft TEMAC license](http://ethernetfmc.com/getting-a-license-for-the-xilinx-tri-mode-ethernet-mac/ "Xilinx Soft TEMAC license")

### Single port limit

This example supports lwIP running on only one port of the Ethernet FMC. You can configure the port
on which to run lwIP by setting the `ETH_FMC_PORT` define in the `main.c` file of the SDK application.
Valid values for `ETH_FMC_PORT` are 0,1,2 or 3.

* When using ports 0..2 the BSP setting "use_axieth_on_zynq" must be set to 1.
* When using port 3, the BSP setting "use_axieth_on_zynq" must be set to 0.

The application will not compile if the correct BSP settings have not been set. To change BSP settings:
right click on the BSP and click `Board Support Package Settings` from the context menu.

**Extra note:** When changing `ETH_FMC_PORT` from 0-2 to 3 (ie. when switching to GEM1), it has been noticed that
you have to power cycle the board. When the SDK project is configured for AXI Ethernet, it must make some
Zynq configurations that are not compatible with the GEM1 configuration.

### Uses Zynq Fabric clocks

To generate the 125MHz and 200MHz clocks required by the AXI Ethernet IPs, this design uses two Zynq
fabric clocks rather than using the Ethernet FMC's on-board 125MHz clock. Generally this is due to resource
limitations of the MicroZed 7Z010, but to be more specific:

* Using the on-board 125MHz clock + Zynq fabric 200MHz clock leads to a timing closure problem that I have not
yet been able to get around.
* Using the on-board 125MHz clock into a clock wizard to generate the 200MHz clock is not possible due to the Zynq 7Z010
only containing two MMCMs.

### Installation of MicroZed board definition files

To use this project, you must first install the board definition files
for the MicroZed into your Vivado installation.

The following folders contain the board definition files and can be found in this project repository at this location:

https://github.com/fpgadeveloper/microzed-qgige/tree/master/Vivado/boards/board_files

* `microzed_7010`
* `microzed_7020`

Copy those folders and their contents into the `C:\Xilinx\Vivado\2016.1\data\boards\board_files` folder (this may
be different on your machine, depending on your Vivado installation directory).

### Library modifications for Vivado 2016.1

To use this project, some modifications must be made to the lwIP libraries
provided by the Xilinx SDK. These modifications can be made either to the
BSP code of your SDK workspace, or to the SDK sources. I personally
recommend modifying the SDK sources as every rebuild of the BSP results
in the BSP sources being overwritten with the SDK sources.

#### Modification to xaxiemacif_dma.c 

Open the following file:

`C:\Xilinx\SDK\2016.1\data\embeddedsw\ThirdParty\sw_services\lwip141_v1_4\src\contrib\ports\xilinx\netif\xaxiemacif_dma.c`

Replace this line of code:

`dmaconfig = XAxiDma_LookupConfig(XPAR_AXIDMA_0_DEVICE_ID);`

With this one:

`dmaconfig = XAxiDma_LookupConfig(xemac->topology_index);`

#### Modification to xemacpsif_physpeed.c

Open the following file:

`C:\Xilinx\SDK\2016.1\data\embeddedsw\ThirdParty\sw_services\lwip141_v1_4\src\contrib\ports\xilinx\netif\xemacpsif_physpeed.c`

Add the following define statement to the code:

`#define XPAR_GMII2RGMIICON_0N_ETH1_ADDR 8`

That defines the "PHY address" of the GMII-to-RGMII converter so that the
Phy_Setup function can properly set the converter's link speed once the
autonegotiation sequence has completed. See the datasheet of the
GMII-to-RGMII converter for more details.

#### Modification to xaxiemacif_physpeed.c

Open the following file:

`C:\Xilinx\SDK\2016.1\data\embeddedsw\ThirdParty\sw_services\lwip141_v1_4\src\contrib\ports\xilinx\netif\xaxiemacif_physpeed.c`

Add the following define statement to the code:

`#define MARVEL_PHY_88E1510_MODEL 0x1D0`

That defines the PHY model identifier for the Marvell 88E1510 PHYs that are
found on the Ethernet FMC.

Add the following function code just above the function called `get_IEEE_phy_speed`:

```c
unsigned int get_phy_speed_88E1510(XAxiEthernet *xaxiemacp, u32 phy_addr)
{
	u16 temp;
	u16 control;
	u16 status;
	u16 partner_capabilities;

	xil_printf("Start PHY autonegotiation \r\n");

	XAxiEthernet_PhyWrite(xaxiemacp,phy_addr, IEEE_PAGE_ADDRESS_REGISTER, 2);
	XAxiEthernet_PhyRead(xaxiemacp, phy_addr, IEEE_CONTROL_REG_MAC, &control);
	//control |= IEEE_RGMII_TXRX_CLOCK_DELAYED_MASK;
	control &= ~(0x10);
	XAxiEthernet_PhyWrite(xaxiemacp, phy_addr, IEEE_CONTROL_REG_MAC, control);

	XAxiEthernet_PhyWrite(xaxiemacp, phy_addr, IEEE_PAGE_ADDRESS_REGISTER, 0);

	XAxiEthernet_PhyRead(xaxiemacp, phy_addr, IEEE_AUTONEGO_ADVERTISE_REG, &control);
	control |= IEEE_ASYMMETRIC_PAUSE_MASK;
	control |= IEEE_PAUSE_MASK;
	control |= ADVERTISE_100;
	control |= ADVERTISE_10;
	XAxiEthernet_PhyWrite(xaxiemacp, phy_addr, IEEE_AUTONEGO_ADVERTISE_REG, control);

	XAxiEthernet_PhyRead(xaxiemacp, phy_addr, IEEE_1000_ADVERTISE_REG_OFFSET,
																	&control);
	control |= ADVERTISE_1000;
	XAxiEthernet_PhyWrite(xaxiemacp, phy_addr, IEEE_1000_ADVERTISE_REG_OFFSET,
																	control);

	XAxiEthernet_PhyWrite(xaxiemacp, phy_addr, IEEE_PAGE_ADDRESS_REGISTER, 0);
	XAxiEthernet_PhyRead(xaxiemacp, phy_addr, IEEE_COPPER_SPECIFIC_CONTROL_REG,
																&control);
	control |= (7 << 12);	/* max number of gigabit attempts */
	control |= (1 << 11);	/* enable downshift */
	XAxiEthernet_PhyWrite(xaxiemacp, phy_addr, IEEE_COPPER_SPECIFIC_CONTROL_REG,
																control);
	XAxiEthernet_PhyRead(xaxiemacp, phy_addr, IEEE_CONTROL_REG_OFFSET, &control);
	control |= IEEE_CTRL_AUTONEGOTIATE_ENABLE;
	control |= IEEE_STAT_AUTONEGOTIATE_RESTART;

	XAxiEthernet_PhyWrite(xaxiemacp, phy_addr, IEEE_CONTROL_REG_OFFSET, control);

	XAxiEthernet_PhyRead(xaxiemacp, phy_addr, IEEE_CONTROL_REG_OFFSET, &control);
	control |= IEEE_CTRL_RESET_MASK;
	XAxiEthernet_PhyWrite(xaxiemacp, phy_addr, IEEE_CONTROL_REG_OFFSET, control);

	while (1) {
		XAxiEthernet_PhyRead(xaxiemacp, phy_addr, IEEE_CONTROL_REG_OFFSET, &control);
		if (control & IEEE_CTRL_RESET_MASK)
			continue;
		else
			break;
	}
	xil_printf("Waiting for PHY to complete autonegotiation.\r\n");

	XAxiEthernet_PhyRead(xaxiemacp, phy_addr, IEEE_STATUS_REG_OFFSET, &status);
	while ( !(status & IEEE_STAT_AUTONEGOTIATE_COMPLETE) ) {
		sleep(1);
		XAxiEthernet_PhyRead(xaxiemacp, phy_addr, IEEE_COPPER_SPECIFIC_STATUS_REG_2,
																	&temp);
		if (temp & IEEE_AUTONEG_ERROR_MASK) {
			xil_printf("Auto negotiation error \r\n");
		}
		XAxiEthernet_PhyRead(xaxiemacp, phy_addr, IEEE_STATUS_REG_OFFSET,
																&status);
		}

	xil_printf("autonegotiation complete \r\n");

	XAxiEthernet_PhyRead(xaxiemacp, phy_addr, IEEE_SPECIFIC_STATUS_REG, &partner_capabilities);

	if ( ((partner_capabilities >> 14) & 3) == 2)/* 1000Mbps */
		return 1000;
	else if ( ((partner_capabilities >> 14) & 3) == 1)/* 100Mbps */
		return 100;
	else					/* 10Mbps */
		return 10;
}
```

That is a custom function that communicates with the 88E1510 PHY to determine the autonegotiated link speed.

Now find this block of code:

```c
	if (phy_identifier == MARVEL_PHY_IDENTIFIER) {
		if (phy_model == MARVEL_PHY_88E1116R_MODEL) {
			return get_phy_speed_88E1116R(xaxiemacp, phy_addr);
		} else if (phy_model == MARVEL_PHY_88E1111_MODEL) {
			return get_phy_speed_88E1111(xaxiemacp, phy_addr);
		}
	}
```

and replace it with this block of code:

```c
	if (phy_identifier == MARVEL_PHY_IDENTIFIER) {
		if (phy_model == MARVEL_PHY_88E1116R_MODEL) {
			return get_phy_speed_88E1116R(xaxiemacp, phy_addr);
		} else if (phy_model == MARVEL_PHY_88E1111_MODEL) {
			return get_phy_speed_88E1111(xaxiemacp, phy_addr);
		} else if (phy_model == MARVEL_PHY_88E1510_MODEL) {
			return get_phy_speed_88E1510(xaxiemacp, phy_addr);
		}
	}
```

We have just added an extra else-if statement to call our custom PHY speed function added earlier.

### License

Feel free to modify the code for your specific application.

### Fork and share

If you port this project to another hardware platform, please send me the
code or push it onto GitHub and send me the link so I can post it on my
website. The more people that benefit, the better.

### About the author

I'm an FPGA consultant and I provide FPGA design services to innovative
companies around the world. I believe in sharing knowledge and
I regularly contribute to the open source community.

Jeff Johnson
[FPGA Developer](http://www.fpgadeveloper.com "FPGA Developer")