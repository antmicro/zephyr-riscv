# Zephyr port to riscv32 architecture
The Zephyr Project is a small, scalable real-time operating system for use on resource-constrained systems supporting multiple architectures.

RISC-V is an open-source instruction set architecture.

This repository provides a port of Zephyr for the riscv32 architecture. Following the zephyr architecture, the port has been splitted into
three parts:
- riscv32 architecture support
- riscv32-based SOC support
- riscv32 board support

riscv32 architecture support is a generic riscv port for handling boot, interrupts/exceptions/fault and context-switching.

The riscv32-based SOC support accounts for SOC-specific:
- system layout
- interrupt handling
- bit manipulation operations
- atomic operations
- peripheral/bus support, such as, timer,  uart, gpio, i2c, spi, etc

The riscv32 board support characterizes the configuration of a given riscv32-based SOC for a particular board. For example,
`zedboard_pulpino` is a configuration of the `pulpino-soc` for the `zedboard`.

`zephyr-riscv` currently handles the following SOCs:
- riscv32-qemu, supporting the sifive machine model
- pulpino

The riscv32 port has successfully been merged into the Zephyr Project master repository at https://gerrit.zephyrproject.org and shall be released in Zephyr 1.7.

The riscv toolchain (gcc, binutils, gdb) and riscv-qemu have successfully been merged into the zephyr SDK and shall be released in the 0.9 SDK version.
A pre-release SDK binary can be obtained from the following link:

https://jenkins.zephyrproject.org/job/meta-zephyr-sdk-merge/68/artifact/meta-zephyr-sdk/scripts/zephyr-sdk-0.9-setup.run

## Compiling `zephyr-riscv` for the `qemu_riscv32` board
Requirements:

either

- zephyr-sdk-0.9

or
- latest `riscv-gnu-toolchain` compiled for riscv32 
- latest `riscv-qemu` supporting the sifive machine model

If you want to compile your own toolchain and riscv-qemu, follow the steps below:

### Getting and compiling the `riscv-gnu-toolchain`
```sh
$ git clone --recursive https://github.com/fractalclone/riscv-gnu-toolchain.git
$ cd riscv-gnu-toolchain
$ ./configure --prefix=/opt/riscv --with-xlen=32 --with-arch=RV32IMAFD
$ sudo make -j
```
After compilation, the `riscv-gnu-toolchain` shall be found at `/opt/riscv`

### Getting and compiling `riscv-qemu`
```sh
$ git clone https://github.com/riscv/riscv-qemu
$ cd riscv-qemu
$ git submodule update --init pixman
$ ./configure --target-list=riscv64-softmmu,riscv32-softmmu --prefix=/opt/riscv
$ make -j
$ sudo make install
```
After compilation, `qemu-system-riscv32` shall be found at `/opt/riscv/bin`

### Installing the zephyr-SDK
If you will use the zephyr-0.9-sdk, install it as follows:
```sh
$ chmod +x zephyr-sdk-0.9-setup.run
$ sudo ./zephyr-sdk-0.9-setup.run
```
After installation, the zephyr-SDK shall be found in `/opt/zephyr-sdk` (if the default target directory for the SDK has been chosen during the installation process).

### Get `zephyr-riscv` and compile a sample app for the `qemu_riscv32` board

Gettting the `zephyr-riscv` sources
```sh
$ git clone https://github.com/fractalclone/zephyr-riscv.git
$ cd zephyr-riscv
```
Within the `zephyr-riscv` directory, setup the zephyr environment variables as follows:
```sh
$ source zephyr-env.sh
```
#### Compiling zephyr using the zephyr-sdk
Configure zephyr to use the zephyr-sdk by exporting the following environment variables:
```sh
$ export ZEPHYR_SDK_INSTALL_DIR=/opt/zephyr-sdk
$ export ZEPHYR_GCC_VARIANT=zephyr
```
Compile and run the `philosophers` sample app for the `qemu_riscv32` board as follows:
```sh
$ cd samples/philosophers
$ make BOARD=qemu_riscv32 run
```
The command shall compile and run the `philosophers` application using the `qemu-system-riscv32` found in the zephyr-sdk
To exit qemu, press `Ctrl-a x`

#### Compiling zephyr using external `riscv-gnu-toolchain`
If you want to use an external `riscv-gnu-toolchain`, configure zephyr to use the `riscv-gnu-toolchain` by exporting the following environment variables:
```sh
$ export RISCV32_TOOLCHAIN_PATH=/path/to/riscv-toolchain
$ export ZEPHYR_GCC_VARIANT=riscv32
```
Compile the `philosophers` sample app for the `qemu_riscv32` board as follows:
```sh
$ cd samples/philosophers
$ make BOARD=qemu_riscv32
```
Once compiled, the `zephyr.elf` file shall be found at `outdir/qemu_riscv32/zephyr.elf`

### Running 'zephyr.elf' using an external `qemu-system-riscv32`
Within the `philosophers` directory:
```sh
$ /path/to/qemu-system-riscv32 -kernel outdir/qemu_riscv32/zephyr.elf -m 32 -machine sifive -nographic
```
To exit qemu, press `Ctrl-a x`

## Compiling `zephyr-riscv` for the `zedboard_pulpino` board
Compiling the zephyr for pulpino will require either:
- the zephyr-sdk-0.9, or
- a riscv-gnu-toolchain compiled from sources at https://github.com/fractalclone/riscv-gnu-toolchain.git, or
- the pulpino-specific toolchain at https://github.com/pulp-platform/ri5cy_gnu_toolchain

The zephyr-sdk-0.9 has a patch that allows the pulpino-specific code to be compiled with the latest `riscv-gnu-toolchain`. Using an unpatched generic `riscv-gnu-toolchain` won't work. This is due to the fact that the `eret` opcode required by pulpino has been removed in the latest `riscv-gnu-toolchain`.

It is also to be noted that, within the latest `riscv-gnu-toolchain`, the `wfi` opcode encoding has changed from 0x10200073 to 0x10500073. However, pulpino only understands 0x10200073 and will generate an illegal instruction fault when trying to execute 0x10500073. Moreover, 0x10200073 is now used to encode the `sret` opcode. For this reason, the port of zephyr to riscv32 architecture comprises a `CONFIG_RISCV_GENERIC_TOOLCHAIN` config variable, which when set, will replace `wfi` by `sret` within the pulpino-specific code. 

### Compiling a sample app for the `zedboard_pulpino` board using the zephyr-sdk
Compiling zephyr for the `zedboard_pulpino`board using the zephyr-sdk is done the same way as for the `qemu_riscv32` board. Example, assuming that the zephyr-sdk environment variables have already been set, compiling the `philosophers` sample app is performed as follows within the `philosophers` directory:
```sh
$ make BOARD=zedboard_pulpino
```
After compilation, the `zephyr.s19` file required by pulpino will be found at `outdir/zedboard_pulpino/zephyr.s19`

Within the `philosophers` directory, convert the `zephyr.s19` to a `spi_stim.txt` file using the utility found at https://github.com/fractalclone/riscv-binaries/blob/master/pulpino/s19toslm.py as follows:
```sh
$ /path/to/s19toslm.py outdir/zedboard_pulpino/zephyr.s19
```
The `spi_stim.txt` will be generated inside the `philosophers` directory. 

Follow the instructions given at https://github.com/pulp-platform/pulpino/tree/master/fpga to setup the zedboard for running the pulpino core. 

Once you have a zedboard running linux and the pulpino fpga bitstream, transfer the `spi_stim.txt` file as well as the `spiload` application to the zedboard. (a precompiled version of the `spiload` app can be obtained at https://github.com/fractalclone/riscv-binaries/blob/master/pulpino/spiload)

Once copied, use the `spiload` application to load the `spi_stim.txt` firmware to the pulpino core as follows:
```sh
$ ./spiload -t10000 spi_stim.txt
```
You can also test the `samples/basic/disco_fever`, `samples/basic/blinky`, `samples/basic/button` and `samples/basic/disco` apps for pulpino.

### Compiling a sample app for the `zedboard_pulpino` board using the pulpino-specific toolchain
To compile a sample app for the `zedboard_pulpino` board using the pulpino-specific toolchain, one must first add the following `CONFIG_RISCV_GENERIC_TOOLCHAIN=n` in the `prj.conf` file found within the sample app directory. This shall allow zephyr to account for pulpino-specific opcodes, like bit-manipulation, and prevent the replacement of `wfi` by `sret`.
Example, within the `philosophers` directory:

Add the `CONFIG_RISCV_GENERIC_TOOLCHAIN=n` to the `prj.conf` file, if not already set
```sh
$ echo "CONFIG_RISCV_GENERIC_TOOLCHAIN=n" >> prj.conf
```
Configure zephyr to use the pulpino-specific toolchain by exporting the following environment variables:
```sh
$ export RISCV32_TOOLCHAIN_PATH=/path/to/ri5cy_gnu_toolchain/install
$ export ZEPHYR_GCC_VARIANT=riscv32
```

Compile the philosophers application for the `zedboard_pulpino` board
```sh
make BOARD=zedboard_pulpino
```
