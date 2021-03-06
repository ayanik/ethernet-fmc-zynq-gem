PetaLinux Project source files
==============================

### How to build the PetaLinux projects

In order to make use of these source files, you must:

1. First generate the Vivado project hardware design(s) (the bitstream) and export the design(s) to SDK.
2. Launch PetaLinux by sourcing the `settings.sh` bash script, eg: `source <path-to-installed-petalinux>/settings.sh`
3. Build the PetaLinux project(s) by executing the `build-petalinux` script in Linux.

The script will generate a separate PetaLinux project for all of the generated and exported Vivado projects that
it finds in the Vivado directory of this repo.

### How the script works

The PetaLinux directory contains a `build-petalinux` shell script which can be run in Linux to automatically
generate a PetaLinux project for each of the generated/exported Vivado projects in the Vivado directory.

When executed, the build script searches the Vivado directory for all projects containing `*.sdk` sub-directories.
This locates all projects that have been exported to SDK. Then for every exported project, the script
does the following:

1. Verifies that the `.hdf` and the `.bit` files exist.
2. Determines the CPU type: Zynq or ZynqMP. It currently does this
by looking at the first 3 letters of the project name.
3. Creates a PetaLinux project, referencing the exported hardware design (.hdf).
4. Copies the relevant configuration files from the `src` directory into the created
PetaLinux project.
5. Builds the PetaLinux project.
6. Generates a BOOT.bin and image.ub file.

### Launch PetaLinux on hardware

#### Via JTAG

To launch the PetaLinux project on hardware via JTAG, connect and power up your hardware and then
use the following commands in a Linux command terminal:

1. Change current directory to the PetaLinux project directory:
`cd <petalinux-project-dir>`
2. Download bitstream to the FPGA:
`petalinux-boot --jtag --fpga`
Note that you don't have to specify the bitstream because this command will use the one that it finds
in the `./images/linux` directory.
3. Download the PetaLinux kernel to the FPGA:
`petalinux-boot --jtag --kernel`

#### Via SD card (Zynq)

To launch the PetaLinux project on hardware via SD card, copy the following files to the root of the
SD card:

* `/<petalinux-project>/images/linux/BOOT.bin`
* `/<petalinux-project>/images/linux/image.ub`

Then connect and power your hardware.

### Configuration files

The configuration files contained in the `src` directory include:

* Device tree
* Rootfs configuration (to include ethtool)
* Interface initializations (sets eth0-4 interfaces to DHCP)
* Kernel configuration
* AXI Ethernet driver patch
* MACB GMII-to-RGMII patch

#### AXI Ethernet driver patch

The AXI Ethernet driver requires a patch for the correct configuration of the RGMII interface's 
RX and TX clock skews.

https://github.com/Xilinx/linux-xlnx/blob/master/drivers/net/ethernet/xilinx/xilinx_axienet_main.c

```		} else if (lp->phy_type == XAE_PHY_TYPE_RGMII_2_0) {
			phydev = of_phy_connect(lp->ndev, lp->phy_node,
						axienet_adjust_link, 0,
						PHY_INTERFACE_MODE_RGMII_ID);
```

The section of the code shown above specifies `PHY_INTERFACE_MODE_RGMII_ID` as the RGMII interface
mode (aka "rgmii-id"). That interface mode enables both the RX and TX clock delays in the PHY but in 
fact we need to enable only the RX delay 
(see http://ethernetfmc.com/rgmii-interface-timing-considerations/ for more information).

Our device tree specifies the correct RGMII configuration with the phy-mode setting ("rgmii-rxid"),
and we have access to this setting via the `lp->phy_interface` variable. So to correct the issue, we
replace the above code with the following:

```		} else if (lp->phy_type == XAE_PHY_TYPE_RGMII_2_0) {
			phydev = of_phy_connect(lp->ndev, lp->phy_node,
						axienet_adjust_link, 0,
						lp->phy_interface);
```

The included patch handles this modification - you do not need to manually modify any code.

#### MACB GMII-to-RGMII patch

The Cadence MACB/GEM driver needs a modification so that it configures the GMII-to-RGMII IP with the
link speed each time that a link is negotiated. This patch is based on answer record 67923, but has
been adapted for the latest version of PetaLinux and the Cadence driver:

https://www.xilinx.com/support/answers/67923.html

Specifically, the GMII-to-RGMII core has a register at address 0x10, accessed via the MDIO bus using
PHY address 8 (as specified in the Vivado design). This register needs to be updated with the actual
link speed (10Mbps, 100Mbps or 1Gbps) and it is normally the MAC driver that should make this update.
The Cadence MACB/GEM driver is not designed to update the GMII-to-RGMII core, hence the need for this
patch. Without this patch, the ports will work at 10Mbps only because the core is configured to that
link speed by default.

The included patch handles this modification - you do not need to manually modify any code.

### Port configurations

#### MicroZed, PicoZed, ZC702, ZC706, ZedBoard

* eth0: Ethernet port of the dev board (GEM0)
* eth1: Ethernet FMC Port 3 (GEM1)
* eth2: Ethernet FMC Port 0 (AXI Ethernet)
* eth3: Ethernet FMC Port 1 (AXI Ethernet)
* eth4: Ethernet FMC Port 2 (AXI Ethernet)

#### ZCU102

* eth0: Ethernet FMC Port 0 (GEM0)
* eth1: Ethernet FMC Port 1 (GEM1)
* eth2: Ethernet FMC Port 2 (GEM2)
* eth3: Ethernet FMC Port 3 (GEM3)

Note: eth3 only implemented on the ZCU102-HPC0 design
