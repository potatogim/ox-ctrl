# OX: A DFC-based Open-Channel SSD Controller

```
COMING SOON...
OX 2.0 is coming with a new full-fledged FTL and Near-Data Processing support!

RELEASE
OX 1.4.0 is released. The main feature is supporting pblk target (host FTL) by enabling 
sector-granularity reads. It allows file systems on top of the DFC namespace. It requires 
the linux kernel 4.13 or higher. Please, check the 'pblk: Host-side FTL setup' section.
```
Documentation:  
- https://github.com/DFC-OpenSource/ox-ctrl/wiki  

Publications:
- uFLIP-OC: Understanding Flash I/O Patterns on Open-Channel Solid-State Drives                                         
  https://dl.acm.org/citation.cfm?id=3124680.3124741

OX is a controller solution for programmable devices like the DFC (https://github.com/DFC-OpenSource/). OX exposes the
device as a LightNVM compatible Open-Channel SSD. OX has been developed for potentially support different FTL responsabilities (e.g. write buffering, ECC) or a full-fledged FTL. FTLs are registered within the OX core, enabling applications to select channels to be managed by a specific FTL. The LightNVM FTL, already in OX, parses LightNVM (http://lightnvm.io/) commands and lets the DFC being managed by the host. e.g. The Linux kernel and user-space applications are able to see, manage and isolate the SSD geometry (channels, LUNs, blocks and pages).


OX is developed under a project at the IT-University of Copenhagen and welcomes all the DFC Community for potential contributions.

Detailed information can be found here (https://github.com/DFC-OpenSource/ox-ctrl/wiki), latest releases here (https://github.com/DFC-OpenSource/ox-ctrl/releases) and code under development here (https://github.com/DFC-OpenSource/ox-ctrl/tree/for-next).

# OX 1.3/1.4 evaluation:  
 Evaluated with FOX 1.0 (https://github.com/DFC-OpenSource/fox).
```
READ : ~385 MB/s
WRITE: ~298 MB/s

The FPGA 03.01.00 provides a maximum throughtput of 400 MB/s READ and 300 MB/s WRITE.
```
![OX Controller bandwidth](https://itu.dk/research/dfc-data/images/ox-performance.png)

Under ./bin you can find OX already compiled:
```
 - ox-ctrl is a essential-only binary for a production environment. 
 - ox-ctrl-test is a complete binary including all tests and administration mode. 
 - ox-ctrl-volt has Volt Storage Media Manager. An in-memory backend instead of NAND, 
 for testing purposes.
 
 IMPORTANT:
 - rootfs_03.00.00_ox-1.x.sqsh is the rootfs containg ox-ctrl. If you do not use this 
 rootfs, you have to disable  the 'issd-nvme' application that runs automatically by other rootfs.
 ```
OX is a user-space application developed in C and cross-compiled for the DFC arm cores. After the board startup, OX must be initialized with one of these commands:
```
$ ox-ctrl start (production)
$ ox-ctrl debug (print everything that is going on)
$ ox-ctrl 
```

# The host kernel

LightNVM is the Linux kernel support for Open-Channel SSDs. It is included in the kernel since version 4.4. For a better OX experience, we recommend the kernel 4.11 or higher. It enables liblightnvm (http://lightnvm.io/liblightnvm/), the user-space library for Open-Channel SSDs. 

Installing this kernel, you will be able to run a wide set of workloads on the DFC using FOX (https://github.com/DFC-OpenSource/fox), a tool for testing Open-Channel SSDs.

pblk (http://lightnvm.io/pblk-tools/) is a host-based full-fledged FTL. It exposes an open-channel SSD as a block device and is currently available in the kernel 4.13 or higher. It means that a standard file system like EXT4 or F2FS can be mounted on top of the DFC. OX release 1.4.0 supports pblk and file systems.

# IF YOU DON'T HAVE A DFC: OX emulation with QEMU

If you do not have an OX-enabled board like the DFC, you can emulate OX with QEMU. We have developed a version of QEMU containing OX with few modifications for the emulated environment. Please follow the instructions in (https://github.com/DFC-OpenSource/qemu-ox). This QEMU version uses VOLT, our volatile media manager with 4 GB of emulated NAND. So, all data will be lost if you turn QEMU off.

# IF YOU HAVE A DFC WITHOUT NAND DIMMs: OX Volt Media Manager

If you have a DFC but does not have a storage FPGA card with NAND DIMMs, your option is using Volt, our volatile backend in the DFC DRAM. Just start OX with one of the follow commands:
```
$ ox-ctrl-volt start
$ ox-ctrl-volt debug
```

# DFC NVMe Driver support

We are working to make the DFC NVMe driver support upstream. For now you can find the driver available in (https://github.com/ivpi/nvme-driver-DFC). 

Remember to blacklist the nvme driver during the system startup. You can do this modifying the follow line in the file "/etc/default/grub", this line also limits the host memory to 8GB (DFC limitation):
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash blacklist=nvme mem=8G"
```

# FTL support:

OX 2.0 is coming with a new full-fledged FTL (use the DFC as a block device only by running OX). You can also use pblk for a host-based FTL (http://lightnvm.io/pblk-tools/). Pblk is a full-fledged FTL upstream in the Linux kernel 4.13 or higher. With pblk, it is be possible to mount a file system on top of OX and the DFC.

# pblk: Host-side FTL setup
```
(1) Install the right kernel with user IO support (kernel 4.13 or higher):
  - https://www.kernel.org/
  
(2) Blacklist the nvme driver (not necessary in QEMU. For the DFC, we use an OX-enabled driver);
  
(3) Start OX Controller in the DFC (in QEMU it will be already started). You have to install OX on the DFC before;
  In the DFC console:
  $ ox-ctrl start

(4) Create the bad block tables for all channels. Skip this step if you already did it in your NAND DIMM.
  Follow the 'Bad block table creation' section for details.
  
(5) Build and load the NVMe driver (in QEMU it will be already loaded);
  - https://github.com/ivpi/nvme-driver-DFC
  $ sudo insmod <driver_folder>/nvme-core.ko
  $ sudo insmod <driver_folder>/nvme.ko
  
(6) Check the kernel log, you should see the device registration messages;
  lnvm: Dragon Fire Card NVMe Driver support.
  nvm: registered nvme0n1 [4/2/512/1024/32/8]
  
(7) Check if pblk is compiled and started;
  $ sudo nvme lnvm info
  
(8) Initialize pblk on the DFC namespace;
  $ sudo nvme lnvm create -t pblk -d nvme0n1 -n dfc -b 0 -e 31
  It might take a while. Please check the kernel log for completion.
  
(9) Create a partition table;
  $ sudo fdisk /dev/dfc
  use 'g' for creating a new partition table
  use 'n' for creating a new partition
  use 'w' for writing the changes to storage
 
(10) Format your partition with a file system;
  $ sudo mkfs.ext4 /dev/dfc1
  Check kernel log for completion.

(11) Mount the partition;
  $ sudo mount /dev/dfc1 <mount_folder>

(12) You might need to give permissions to your user;
  $ sudo chown <user>:<user> <mount_folder>
  
(13) Enjoy your new storage device transferring some files to it :)
```

# FOX testing setup:
```

(1) Install the right kernel with user IO support (kernel 4.11 or higher):
  - https://www.kernel.org/
  
(2) Blacklist the nvme driver (not necessary in QEMU. For the DFC, we use an OX-enabled driver);

(3) Install liblightnvm;
  - http://lightnvm.io/liblightnvm/
  
(4) Start OX Controller in the DFC (in QEMU it will be already started). You have to install OX on the DFC before;
  In the DFC console:
  $ ox-ctrl start

(5) Create the bad block tables for all channels. Skip this step if you already did it in your NAND DIMM.
  Follow the 'Bad block table creation' section for details.
  
(6) Build and load the NVMe driver (in QEMU it will be already loaded);
  - https://github.com/ivpi/nvme-driver-DFC
  $ sudo insmod <driver_folder>/nvme-core.ko
  $ sudo insmod <driver_folder>/nvme.ko
  
(7) Check the kernel log, you should see the device registration messages;
  lnvm: Dragon Fire Card NVMe Driver support.
  nvm: registered nvme0n1 [4/2/512/1024/32/8]
  
(8) Run the tests with the tool (https://github.com/DFC-OpenSource/fox), or use liblightnvm as you wish.
```

# FPGA version and DFC config
 
This controller supports the FPGA 3.01.00.

The latest stable DFC firmware version (with OX 1.4.0):
```
========================================
        Image           Version                
 ========================================
    PBL                   03.00.01
    DPC                   03.00.00
    MC                    03.00.00
    Kernel                05.00.00
    U-boot                03.00.00
    rootfs                03:00:00
    CPLD_NIC              A12 
    CPLD_SSD              A02 
    FPGA                  03.01.00

```
The latest DFC hardware configuration:
```
LS2085 Dragon Fire Card (soon LS2088);
2 NAND DIMM modules installed in the slots M1 and M3 of the storage card.
```

# Internal components:

```
      - Media managers: NAND, DDR, etc.
      - Flash Translation Layers + LightNVM raw FTL support
      - Interconnect handler: PCIe, network fabric, etc.
      - NVMe queues and tail/head doorbells
      - NVMe, LightNVM and Fabric(not implemented) command parser
      - RDMA handler(not implemented): RoCE, InfiniBand, etc.

      Media managers (MMGR): Identifies the non-volatile memory, and registers a channel
        abstraction with its geometry to be available to the stack. Media managers
        implement read/write/erase commands to specific pages and blocks. OX may have
        several media managers exposing channels from different sources.
        
      Flash Translation Layers (FTL): Manages MMGR channels, accepts IO commands (a
        command may have several pages) and sends page-level commands to the media
        managers. It implements responsibilities such as WL, L2P, GC, BB and ECC. Several
        FTLs may run within OX and be responsible to manage different MMGR channels.

      Interconnect handlers (ICH): Communication between controller and host. This layer
        implements responsibilities such as NVMe register mapping and MSI interruptions. It
        gathers data from the host and sends it to the command parser layer.
        
      NVMe queue support: Read/write NVMe registers mapped by the ICH, manage admin/
        IO NVMe queue(s) and perform the DMA data transfer if FPGA does not provide it.
        
      Command parsers: Parses commands from the ICH and call the right FTL
        instance to handle it.
```
We have implemented a PCIe interconnection handler, a media manager to expose 8 channels, NVMe and LightNVM support for the DFC.

# Bad block table creation
The bad block table is an essential component of a flash block device, without it you can expect data corruption due
application bad block allocation. OX has an admin command for creating per-channel bad block tables. Please follow the instruction in 'ox-ctrl-test admin -t create-bbt' command.

When attempting to create a bad block table in a given channel, OX will ask which process you want to use. There are 3 available processes:
``` 
   1 - Full scan (Erase, write full, read full, compare buffers) - RECOMMENDED. Might take a long time
           - Backup the channel data before. ALL DATA WILL BE LOST.
           
   2 - Fast scan (Only erase the blocks) - Fast creation, but can have omitted bad blocks.
           - Backup the channel data before. ALL DATA WILL BE LOST.
           
   3 - Emergency table (Creates an empty bad block table without erasing blocks) - Very fast and the channel is not erased.
           - If you can't make a backup of your channel, use this option for initializing OX, however, all blocks will be
              marked as good. You should use the DFC in read-only state if you create an emergency bad block table.
```
Bad block table creation option is available in the ox-ctrl-test binary.

# Installation:

The Makefile creates 3 binaries (ox-ctrl, ox-ctrl-test and ox-ctrl-volt). 
 - ox-ctrl is a essential-only binary for a production environment. 
 - ox-ctrl-test is a complete binary including all tests and administration mode. 
 - ox-ctrl-volt has Volt Storage Media Manager. An in-memory backend instead of NAND, for testing purposes.

Under /bin you can find the binaries + the DFC rootfs with OX included. 

WE RECOMMEND THE USE OF ROOTFS. AFTER UPGRADE THE ROOTFS FIRMWARE, ONLY TYPE 'ox-ctrl', 'ox-ctrl-test' OR 'ox-ctrl-volt' TO USE OX.

# OX command line

$ ox-ctrl start
```
Use 'start' to run the controller with standard settings.
```

$ ox-ctrl debug
```
Use 'debug' to run the controller in debug mode (verbose).
```
$ ox-ctrl null
```
Use 'null' to run the controller as a null device. No data transfer is performed, OX will complete all 
I/Os succesfully. This mode is useful for testing the NVMe queues.
```
$ ox-ctrl null-debug
```
Use 'null-debug' to run the controller as a null device and show the debug information in the screen.
```
$ ox-ctrl --help
```
*** OX Controller ***
 
 An OpenChannel SSD Controller

 Available commands:
  start            Start controller with standard configuration
  debug            Start controller and print Admin/IO commands
  test             Start controller, run tests and close
  admin            Execute specific tasks within the controller
  null             Start controller with Null IOs (NVMe queue tests)
  null-debug       Null IOs and print Admin/IO commands
 
 Initial release developed by Ivan L. Picoli, the red-eagle


  -?, --help                 Give this help list
      --usage                Give a short usage message
  -V, --version              Print program version

Report bugs to Ivan L. Picoli <ivpi@itu.dk>

```
The same commands apply for VOLT media menager (ox-ctrl-volt).

$ ox-ctrl-test test --help
```
Use this command to run tests, it will start the controller, run the tests and
close the controller.

 Examples:
  Show all available set of tests + subtests:
    ox-ctrl test -l

  Run all available tests:
    ox-ctrl test -a

  Run a specific test set:
    ox-ctrl test -s <set_name>

  Run a specific subtest:
    ox-ctrl test -s <set_name> -t <subtest_name>

  -a, --all[=run_all]        Use to run all tests.
  -l, --list[=list]          Show available tests.
  -s, --set=test_set         Test set name. <char>
  -t, --test=test            Subtest name. <char>
  -?, --help                 Give this help list
      --usage                Give a short usage message
  -V, --version              Print program version

Mandatory or optional arguments to long options are also mandatory or optional
for any corresponding short options.

Report bugs to Ivan L. Picoli <ivpi@itu.dk>.

```

$ ox-ctrl-test admin --help
```
Use this command to run specific tasks within the controller.

 Examples:
  Show all available Admin tasks:
    ox-ctrl admin -l

  Run a specific admin task:
    ox-ctrl admin -t <task_name>

  -l, --list[=list]          Show available admin tasks.
  -t, --task=admin_task      Admin task to be executed. <char>
  -?, --help                 Give this help list
      --usage                Give a short usage message
  -V, --version              Print program version

Mandatory or optional arguments to long options are also mandatory or optional
for any corresponding short options.

Report bugs to Ivan L. Picoli <ivpi@itu.dk>.
```

$ ox-ctrl-test admin -l
```
OX Controller ADMIN
 
Available OX Admin Tasks: 

  - 'erase-blk': erase specific blocks.
     eg. ox-ctrl-test admin -t erase-blk (follow instructions within command)

  - 'create-bbt': creates a new bad block table for a specific channel.
     eg. ox-ctrl-test admin -t create-bbt (follow instructions within command)
```

# OX layer registration:

```
Besides the DFC, OX is extensible to other devices through the layer registration interface. In order to
use OX in other platforms, it is needed to develop the follow layers:

- Interconnect handler: Here you map the NVMe registers according your platform, or make a Fabrics interconnection.
- Media Manager: Here is the channel abstraction for the non-volatile memory, to be used by the other OX layers.
- FTL: The same FTL can be used in any platform, but you can develop a new one and register it within the OX core.
```
