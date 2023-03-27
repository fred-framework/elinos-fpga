# elinos-fpga
Testing ElinOS working with FPGA on the ZCU102 board.

| :exclamation:  This is under development and not fully tested   |
|-----------------------------------------------------------------|

This example, based on this [example](https://xilinx.github.io/Embedded-Design-Tutorials/docs/2020.2/build/html/docs/Introduction/ZynqMPSoC-EDT/7-design1-using-gpio-timer-interrupts.html) by Xilinx, builds the HW/SW design for testing ElinOS and the FPGA. The example is divided in three parts: Hw design, where the bitstream is generated; sw design, where the application is compiled; and runtime, where the bitstream is loaded and the application is executed.

## HW Design

This step creates the bitstream and the XSA file for Vitis. First, create an empty Vivado project for the target board, in this case, tested wit ZCU102. Next, download the [TCL file](https://github.com/Xilinx/Embedded-Design-Tutorials/tree/2020.2/docs/Introduction/ZynqMPSoC-EDT/ref_files/design1/edt_zcu102.tcl).
This script allows to quickly recreate the hardware block diagram, skipping a good part of the tutorial. Open a terminal within Vivado run `source edt_zcu102.tcl`. Next, generate the wrapper for the top design. Then, Generate the bitstream. Finally, File -> Export -> Export Hardware ... Select Include Bitstream. The XSA file should have been generated, to be used by Vitis.

## SW Design

This step builds the [Linux example software application](https://github.com/Xilinx/Embedded-Design-Tutorials/blob/2020.2/docs/Introduction/ZynqMPSoC-EDT/ref_files/design1/ps_pl_linux_app.c).
Still in Vivado, run Tools -> Launch Vitis IDE. In the Tutorial, we can skip to [Creating the Linux Domain for Linux Applications](https://xilinx.github.io/Embedded-Design-Tutorials/docs/2020.2/build/html/docs/Introduction/ZynqMPSoC-EDT/7-design1-using-gpio-timer-interrupts.html#creating-the-linux-domain-for-linux-applications) since we don't want the bare-metal application examples. Configure `Linux_Domain` as specified in the tutorial. The bif entry can be left empty. Right click in the `edt_zcu102_wrapper` Platform and select Build Project. 

Proceeed to the step [Creating the Linux Application Project](https://xilinx.github.io/Embedded-Design-Tutorials/docs/2020.2/build/html/docs/Introduction/ZynqMPSoC-EDT/7-design1-using-gpio-timer-interrupts.html#creating-the-linux-application-project) in the tutorial. 
Import the [application source code](https://github.com/Xilinx/Embedded-Design-Tutorials/blob/2020.2/docs/Introduction/ZynqMPSoC-EDT/ref_files/design1/ps_pl_linux_app.c) and add `pthread` library in C/C++ Build Settings. Then, build the application. This generates the application elf executable.

Finally, the last step is to generate the boot image which encapsulate the bitstream via Xilinx -> Create Boot Image. Different from the tutorial, the bin file will consist of a single partition with the bitstream. Click in Add partition and select the bit file, partition type as data file, and target PL.
Confirm the partition and Create image. The files bitstream.bit.bin and ps_pl_linux_app.elf must be copied to a sd-card.

You can also perform this last task via command line, as described [here](https://docs.xilinx.com/r/en-US/ug1144-petalinux-tools-reference-guide/DTG-Settings). Create a bif file like this:

```
all: 
{
        [destination_device = pl] <bitstream in .bit> ( Ex: system.bit ) 
}
```

Then run `bootgen`:

```
bootgen -image bitstream.bif -arch zynqmp -o bitstream.bit.bin
```

## Running in ELinos

### Kernel Image Requirements

The following kernel features must be enabled in the ElinOS image:
 - CONFIG_FPGA, CONFIG_FPGA_MGR_ZYNQMP_FPGA: to enable FPGA manager device driver
 - IKCONFIG: to mount /proc/config.gz
 - Be aware that this list of kernel features is probably not complete ...


This can be checked running a command like this:
```
# zcat /proc/config.gz | grep FPGA
# CONFIG_SND_SOC_XTFPGA_I2S is not set
# CONFIG_GS_FPGABOOT is not set
CONFIG_FPGA=y
CONFIG_FPGA_MGR_DEBUG_FS=y
# CONFIG_FPGA_MGR_ALTERA_PS_SPI is not set
# CONFIG_FPGA_MGR_ALTERA_CVP is not set
# CONFIG_FPGA_MGR_XILINX_SPI is not set
# CONFIG_FPGA_MGR_ICE40_SPI is not set
# CONFIG_FPGA_MGR_MACHXO2_SPI is not set
CONFIG_XILINX_AFI_FPGA=y
CONFIG_FPGA_BRIDGE=y
CONFIG_FPGA_REGION=y
CONFIG_OF_FPGA_REGION=y
# CONFIG_FPGA_DFL is not set
CONFIG_FPGA_MGR_ZYNQMP_FPGA=y
```

### Using FPGA Manager

Copy the `bitstream.bit.bin` and `ps_pl_linux_app.elf` to the running ELinOS image. Let's assume that these were copied to the dir `/opt/fpga-mgr/`.

| :warning: WARNING                                              |
|:---------------------------------------------------------------|
| TBD: I suppose the PL device tree should be copied too.  ...   |

Here are some references for using FPGA manager in Linux:
 - https://www.hackster.io/anujvaishnav20/programming-the-pl-at-runtime-with-petalinux-72a820
 - https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18841847/Solution+ZynqMP+PL+Programming. Go to 'Steps for programming the Full Bitstream'
 - https://docs.xilinx.com/r/en-US/ug1144-petalinux-tools-reference-guide/FPGA-Manager-Configuration-and-Usage-for-Zynq-7000-Devices-and-Zynq-UltraScale-MPSoC

Run the following steps to program the FPGA:

```
cp /opt/fpga-mgr/bitstream.bif.bin /opt/fpga-mgr/static.bin
modprobe zynqmp-fpga-fmod
cp /opt/fpga-mgr/static.bin  /lib/firmware/
echo 0 > /sys/class/fpga_manager/fpga0/flags
echo /opt/fpga-mgr/staic.bin  > /sys/class/fpga_manager/fpga0/firmware
```

and then run the application `ps_pl_linux_app.elf`. Press the push buttons to test the application. 

## Authors

 - Alexandre Amory (Feb 2023), [Real-Time Systems Laboratory (ReTiS Lab)](https://retis.santannapisa.it/), [Scuola Superiore Sant'Anna (SSSA)](https://www.santannapisa.it/), Pisa, Italy.

The source code in this repo is originnaly from the [example](https://xilinx.github.io/Embedded-Design-Tutorials/docs/2020.2/build/html/docs/Introduction/ZynqMPSoC-EDT/7-design1-using-gpio-timer-interrupts.html) by Xilinx. No authorship is claimed for these files.

## Funding
 
This software package has been developed in the context of the [AMPERE project](https://ampere-euproject.eu/). This project has received funding from the European Union’s Horizon 2020 research and innovation programme under grant agreement No 871669.
