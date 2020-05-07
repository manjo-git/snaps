# Storage Performance Development Kit (SPDK) 
## Using The SPDK snap.

Install the SPDK snap, and make sure you have /snap/bin in your path. Once
successfully installed you should see the following:

```
/snap/bin$ ls
spdk.gen-nvme   spdk.rpc         spdk.spdk-tgt           spdk.vhost
spdk.iscsi-tgt  spdk.setup       spdk.spdk-top
spdk.iscsi-top  spdk.spdkcli     spdk.spdk-trace
spdk.nvmf-tgt   spdk.spdk-lspci  spdk.spdk-trace-record
/snap/bin$
```

### Example: Setup NVME-OF over TCP ARM64 initiator and AMD64 target.
#### ARM64 initiator.

##### Set vfio non-iommu.
This is required on the BCM SST100 development board because the dtb used
does not support IOMMU. This is very finiky, if you thing go wrong it 
usually happens when you attach the controller to the PCIe ID. 

My work around was to kill the nvme-tgt, do a spdk.setup cleanup followed by, 
manually removing all vfio modules, and loading only the vfio-pci. Then
set enable_unsafe_noiommu_mode in /sys and modprobe the vfio-pci module.

```
root@sst100:/home/ubuntu# echo 1 > /sys/module/vfio/parameters/enable_unsafe_noiommu_mode
root@sst100:/home/ubuntu# modprobe vfio-pci
```

After that do spdk.setup config. But the following steps should also work, and you should try that first. NOTE below, you need to modprobe vfio-pci *NOT* vfio.

```
ubuntu@sst100:~$ sudo su
root@sst100:/home/ubuntu# echo "options vfio enable_unsafe_noiommu_mode=1" > /etc/modprobe.d/vfio-noiommu.conf
root@sst100:/home/ubuntu# exit
ubuntu@sst100:~$ 
ubuntu@sst100:~$ lsmod | grep vfio
ubuntu@sst100:~$ sudo modprobe vfio-pci
```

##### Use the SPDK setup script
This step is needed to allocate hugepages and unbind any NVMe devices from
native kernel drivers.

```
ubuntu@sst100:~$ sudo HUGEMEM=8192 DRIVER_OVERRIDE=vfio-pci spdk.setup config
0007:01:00.0 (144d a808): nvme -> vfio-pci

Current user memlock limit: 16 MB

This is the maximum amount of memory you will be
able to use with DPDK and VFIO if run as current user.
To change this, please adjust limits.conf memlock limit for current user.

## WARNING: memlock limit is less than 64MB
## DPDK with VFIO may not be able to initialize if run as current user.
modprobe: FATAL: Module msr not found in directory /lib/modules/5.4.0-29-generic
ubuntu@sst100:~$
```

##### Start nvme-tgt
In a seperate window start the nvme target demon

```
ubuntu@sst100:~$ sudo spdk.nvmf-tgt 
Starting SPDK v20.07-pre git sha1 2c0980de7 / DPDK 19.11.0 initialization...
[ DPDK EAL parameters: nvmf --no-shconf -c 0x1 --log-level=lib.eal:6 --log-level=lib.cryptodev:5 --log-level=user1:6 --base-virtaddr=0x200000000000 --match-allocations --file-prefix=spdk_pid3934 ]
EAL: No available hugepages reported in hugepages-32768kB
EAL: No available hugepages reported in hugepages-64kB
EAL: No available hugepages reported in hugepages-1048576kB
EAL: VFIO support initialized
app.c: 646:spdk_app_start: *NOTICE*: Total cores available: 1
reactor.c: 371:_spdk_reactor_run: *NOTICE*: Reactor started on core 0
accel_engine.c: 227:spdk_accel_engine_initialize: *NOTICE*: Accel engine initialized to use software engine.
```

##### Expose the NVMe device over TCP

```
ubuntu@sst100:~$ sudo spdk.rpc nvmf_create_transport -t TCP -u 16384 -p 8 -c 8192
```

Make a note of the PCI ID of the nvme device. 

```
ubuntu@sst100:~$ lspci | grep "Non-Volatile memory controller"
0007:01:00.0 Non-Volatile memory controller: Samsung Electronics Co Ltd NVMe SSD Controller SM981/PM981
ubuntu@sst100:~$ 
```

```
ubuntu@sst100:~$ sudo spdk.rpc bdev_nvme_attach_controller -b NVMe0 -t PCIe -a 0007:01:00.0
NVMe0n1
ubuntu@sst100:~$ sudo spdk.rpc nvmf_create_subsystem nqn.2020-04.io.spdk:cnode1 -a -s SPDK00000000000001 -d SPDK_Controller1
ubuntu@sst100:~$ sudo spdk.rpc nvmf_subsystem_add_ns nqn.2020-04.io.spdk:cnode1 NVMe0n1
ubuntu@sst100:~$ sudo spdk.rpc nvmf_subsystem_add_listener nqn.2020-04.io.spdk:cnode1 -t tcp -a 192.168.0.96 -s 4420
```

Now the NVMe drive is exposed to the target system over TCP. 

#### AMD64 target
##### Load the nvme-tcp module.

```
arm01@amd64:~$ lsmod | grep nvme-tcp
arm01@arm01:~$ sudo modprobe nvme-tcp
```

##### Discover the NVMe drive.

```
arm01@arm01:~$ sudo nvme discover -t tcp -a 192.168.0.96  -s 4420

Discovery Log Number of Records 1, Generation counter 3
=====Discovery Log Entry 0======
trtype:  unrecognized
adrfam:  ipv4
subtype: nvme subsystem
treq:    not required
portid:  0
trsvcid: 4420
subnqn:  nqn.2020-04.io.spdk:cnode1
traddr:  192.168.0.96
arm01@arm01:~$ 
```

##### Connect to the NVMe drive.

```
arm01@arm01:~$ sudo nvme connect -t tcp -n nqn.2020-04.io.spdk:cnode1 -a 192.168.0.96  -s 4420
arm01@arm01:~$ sudo nvme list-subsys
nvme-subsys0 - NQN=nqn.2014.08.org.nvmexpress:144d144dS41GNX1M590398      SAMSUNG MZVLB256HAHQ-000L7              
\
 +- nvme0 pcie 0000:01:00.0
nvme-subsys1 - NQN=nqn.2020-04.io.spdk:cnode1
\
 +- nvme1 tcp traddr=192.168.0.96 trsvcid=4420
arm01@arm01:~$ sudo nvme list
Node             SN                   Model                                    Namespace Usage                      Format           FW Rev  
---------------- -------------------- ---------------------------------------- --------- -------------------------- ---------------- --------
/dev/nvme0n1     S41GNX1M590398       SAMSUNG MZVLB256HAHQ-000L7               1          67.40  GB / 256.06  GB    512   B +  0 B   1L2QEXD7
/dev/nvme1n1     SPDK00000000000001   SPDK_Controller1                         1         500.11  GB / 500.11  GB    512   B +  0 B   20.07   
arm01@arm01:~$
```

##### To disconnect

```
arm01@arm01:~$ nvme disconnect -n "nqn.2020-04.io.spdk:cnode1"
NQN:nqn.2020-04.io.spdk:cnode1 disconnected 0 controller(s)
arm01@arm01:~$
```
