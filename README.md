# Simple example showing how to deploy and test SR-IOV RDMA between Azure HPC VMs Standard_HC44rs

## Deploy VMs

As of February 2019 HC44rs VMs are still in preview, your subscription needs to be whitelisted to be able to deploy these VMs. Learn more [here](https://azure.microsoft.com/en-us/blog/introducing-the-new-hb-and-hc-azure-vm-sizes-for-hpc/).

In this example, I deploy VMs using SLES 12 SP3 just for testing, but minimal supported version of SLES is currently SLES 12 SP4.

Create resource group
```
az group create -n avhc1 -l westus2
```

Create Standard_HC44rs VMs in the resource group
```
az group deployment create -g avhc1 --template-file create-hc-vm.json
```

SSH to the VM and check if controller is being passed through
```
avhc20:~ # lspci
0000:00:00.0 Host bridge: Intel Corporation 440BX/ZX/DX - 82443BX/ZX/DX Host bridge (AGP disabled) (rev 03)
0000:00:07.0 ISA bridge: Intel Corporation 82371AB/EB/MB PIIX4 ISA (rev 01)
0000:00:07.1 IDE interface: Intel Corporation 82371AB/EB/MB PIIX4 IDE (rev 01)
0000:00:07.3 Bridge: Intel Corporation 82371AB/EB/MB PIIX4 ACPI (rev 02)
0000:00:08.0 VGA compatible controller: Microsoft Corporation Hyper-V virtual VGA
98da:00:02.0 Infiniband controller: Mellanox Technologies MT27800 Family [ConnectX-5 Virtual Function]
```

## SLES 12 SP3

I tried two approaches to install Mellanox drivers on my VMs:
1. download, build, and install the latest OFED drivers 
2. use "zypper install" to install the drivers

### Option 1: Download, build, and install Mellanox OFED drivers

Check current kernel version
```
avhc20:~ # uname -a
Linux avhc20 4.4.143-4.13-azure #1 SMP Fri Aug 10 09:24:25 UTC 2018 (52f9184) x86_64 x86_64 x86_64 GNU/Linux
```

Update to latest version in SP3 that kernel-devel-azure package can be installed properly
```
zypper update
reboot
```

Check latest kernel version. For me on Feb 21 '19 it was 4.4.170-4.22-azure.
```
avhc20:~ # uname -a
Linux avhc20 4.4.170-4.22-azure #1 SMP Thu Jan 17 06:00:39 UTC 2019 (5499596) x86_64 x86_64 x86_64 GNU/Linux
```

I tried the steps below by following instructions from Mellanox https://community.mellanox.com/s/article/howto-install-mlnx-ofed-driver

Download SLES 12 SP3 specific OFED from Mellanox
```
wget http://content.mellanox.com/ofed/MLNX_OFED-4.5-1.0.1.0/MLNX_OFED_LINUX-4.5-1.0.1.0-sles12sp3-x86_64.tgz
tar xvf MLNX_OFED_LINUX-4.5-1.0.1.0-sles12sp3-x86_64.tgz
```

Install requirements for building OFED RPM
```
zypper -n install kernel-devel-azure
zypper -n install createrepo rpm-build kernel-syms gcc make python-devel python-libxml2 tk libgtk-2_0-0
```

Build and install OFED with kernel support
```
./MLNX_OFED_LINUX-4.5-1.0.1.0-sles12sp3-x86_64/mlnxofedinstall --add-kernel-support

reboot
```

Check that mlx5 module is loaded
```
avhc21:~ # lsmod | grep mlx5
mlx5_fpga_tools        16384  0
mlx5_ib               356352  0
ib_uverbs             126976  3 mlx5_ib,ib_ucm,rdma_ucm
ib_core               307200  10 rdma_cm,ib_cm,iw_cm,mlx4_ib,mlx5_ib,ib_ucm,ib_umad,ib_uverbs,rdma_ucm,ib_ipoib
mlx5_core             946176  2 mlx5_ib,mlx5_fpga_tools
mlxfw                  24576  1 mlx5_core
devlink                32768  4 mlx4_en,mlx4_ib,mlx4_core,mlx5_core
inet_lro               16384  2 mlx5_core,ib_ipoib
mlx_compat             24576  15 rdma_cm,ib_cm,iw_cm,mlx4_en,mlx4_ib,mlx5_ib,mlx5_fpga_tools,ib_ucm,ib_core,ib_umad,ib_uverbs,mlx4_core,mlx5_core,rdma_ucm,ib_ipoib
vxlan                  49152  2 mlx4_en,mlx5_core
ptp                    20480  3 hv_utils,mlx4_en,mlx5_core
```

### Option 2: Use "zypper install" to get Mellanox drivers

I tried the steps below by following instructions from Mellanox http://www.mellanox.com/pdf/prod_software/SUSE_Linux_Enterprise_Server_(SLES)_12_SP3_Driver_User_Manual.pdf

Install Mellanox drivers and reboot
```
zypper -n install rdma-core librdmacm1 libibmad5 libibumad3 libibverbs perftest
reboot
```

Check that mlx5 module is loaded
```
lsmod | grep mlx5
mlx5_ib               188416  0
ib_core               233472  13 rdma_cm,ib_cm,iw_cm,rpcrdma,mlx5_ib,ib_srp,ib_iser,ib_srpt,ib_umad,ib_uverbs,rdma_ucm,ib_ipoib,ib_isert
mlx5_core             417792  1 mlx5_ib
devlink                32768  1 mlx5_core
vxlan                  49152  1 mlx5_core
ptp                    20480  2 hv_utils,mlx5_core
```

### Configure ib0 interface manually 

Get the rdmaIPv4Address from SharedConfig.xml file
```
cat /var/lib/waagent/SharedConfig.xml
```

Add ib0 interface for the rdmaIPv4Address from above (e.g. 172.16.1.29 )
```
ifconfig ib0 172.16.1.29
```

ifconfig output
```
avhc20:~ # ifconfig
eth0      Link encap:Ethernet  HWaddr 00:0D:3A:F9:8D:15
          inet addr:10.0.0.4  Bcast:10.0.0.255  Mask:255.255.255.0
          inet6 addr: fe80::20d:3aff:fef9:8d15/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:3068 errors:0 dropped:0 overruns:0 frame:0
          TX packets:3456 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:2020722 (1.9 Mb)  TX bytes:618068 (603.5 Kb)

eth1      Link encap:Ethernet  HWaddr 00:15:5D:33:FF:25
          inet6 addr: fe80::215:5dff:fe33:ff25/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:48 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 b)  TX bytes:13852 (13.5 Kb)

ib0       Link encap:InfiniBand  HWaddr 80:00:08:87:FE:80:00:00:00:00:00:00:00:00:00:00:00:00:00:00
          inet addr:172.16.1.28  Bcast:172.16.255.255  Mask:255.255.0.0
          inet6 addr: fe80::215:5dff:fd33:ff25/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:2044  Metric:1
          RX packets:159 errors:0 dropped:0 overruns:0 frame:0
          TX packets:85 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:256
          RX bytes:12763 (12.4 Kb)  TX bytes:8875 (8.6 Kb)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:4 errors:0 dropped:0 overruns:0 frame:0
          TX packets:4 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:264 (264.0 b)  TX bytes:264 (264.0 b)
```


### Configure ib0 interface via waagent

Update waagent to version 2.2.36 (default version on my VM was 2.2.18)
```
zypper remove python-azure-agent
wget https://github.com/Azure/WALinuxAgent/archive/v2.2.36.zip
unzip v2.2.36.zip
cd WALinuxAgent*/
python setup.py install --register-service --force
sed -i -e 's/# OS.EnableRDMA=y/OS.EnableRDMA=y/g' /etc/waagent.conf
sed -i -e 's/# AutoUpdate.Enabled=y/AutoUpdate.Enabled=y/g' /etc/waagent.conf
systemctl restart waagent
```

### Test latency using ib_send_lat

Server
```
avhc20:~ # ib_send_lat -a

************************************
* Waiting for client to connect... *
************************************
---------------------------------------------------------------------------------------
                    Send Latency Test
 Dual-port       : OFF          Device         : mlx5_0
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 RX depth        : 512
 Mtu             : 4096[B]
 Link type       : IB
 Max inline data : 0[B]
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0x2c0 QPN 0x0888 PSN 0x7466df
 remote address: LID 0x2e8 QPN 0x0888 PSN 0x288a8c
---------------------------------------------------------------------------------------
 #bytes #iterations    t_min[usec]    t_max[usec]  t_typical[usec]
 2       1000          2.56           6.11         2.66
 4       1000          2.48           6.41         2.54
 8       1000          2.49           6.22         2.56
 16      1000          2.49           6.36         2.55
 32      1000          2.50           6.06         2.56
 64      1000          2.56           6.28         2.61
 128     1000          2.58           6.11         2.65
 256     1000          2.63           5.71         2.69
 512     1000          2.67           5.89         2.73
 1024    1000          2.81           6.39         2.93
 2048    1000          3.05           7.42         3.15
 4096    1000          3.56           6.56         3.65
 8192    1000          4.20           8.57         4.36
 16384   1000          5.66           27.24        5.88
 32768   1000          7.79           10.89        8.07
 65536   1000          10.82          14.28        11.10
 131072  1000          20.37          23.25        20.61
 262144  1000          31.81          34.65        32.22
 524288  1000          53.74          62.65        59.27
 1048576 1000          97.66          105.15       101.36
 2097152 1000          185.08         192.99       188.32
 4194304 1000          359.43         364.06       361.37
 8388608 1000          706.45         713.13       709.48
---------------------------------------------------------------------------------------
```

Client
```
avhc21:~ # ib_send_lat -a 172.16.1.28
---------------------------------------------------------------------------------------
                    Send Latency Test
 Dual-port       : OFF          Device         : mlx5_0
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 TX depth        : 1
 Mtu             : 4096[B]
 Link type       : IB
 Max inline data : 0[B]
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0x2e8 QPN 0x0888 PSN 0x288a8c
 remote address: LID 0x2c0 QPN 0x0888 PSN 0x7466df
---------------------------------------------------------------------------------------
 #bytes #iterations    t_min[usec]    t_max[usec]  t_typical[usec]
 2       1000          2.57           15.38        2.66
 4       1000          2.49           9.48         2.54
 8       1000          2.49           6.08         2.56
 16      1000          2.49           6.44         2.55
 32      1000          2.51           13.14        2.56
 64      1000          2.55           6.26         2.61
 128     1000          2.59           6.08         2.65
 256     1000          2.63           5.71         2.69
 512     1000          2.67           5.87         2.73
 1024    1000          2.82           6.38         2.93
 2048    1000          3.05           8.29         3.15
 4096    1000          3.56           6.54         3.66
 8192    1000          4.19           8.54         4.37
 16384   1000          5.68           27.13        5.88
 32768   1000          7.81           13.14        8.07
 65536   1000          10.85          16.83        11.10
 131072  1000          20.35          25.60        20.61
 262144  1000          31.81          36.65        32.23
 524288  1000          53.68          61.23        59.42
 1048576 1000          97.63          105.62       101.32
 2097152 1000          185.40         194.59       188.55
 4194304 1000          359.20         365.20       361.40
 8388608 1000          706.63         715.72       708.96
---------------------------------------------------------------------------------------
```

### Test throughput using ib_send_bw

Server
```
avhc20:~ # ib_send_bw -a

************************************
* Waiting for client to connect... *
************************************
---------------------------------------------------------------------------------------
                    Send BW Test
 Dual-port       : OFF          Device         : mlx5_0
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 RX depth        : 512
 CQ Moderation   : 100
 Mtu             : 4096[B]
 Link type       : IB
 Max inline data : 0[B]
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0x2c0 QPN 0x0889 PSN 0x313ab1
 remote address: LID 0x2e8 QPN 0x0889 PSN 0xd12b22
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 2          1000           0.00               10.51                5.512208
 4          1000           0.00               20.64                5.410485
 8          1000           0.00               41.18                5.397476
 16         1000           0.00               82.49                5.405794
 32         1000           0.00               165.01               5.407096
 64         1000           0.00               331.14               5.425436
 128        1000           0.00               660.11               5.407661
 256        1000           0.00               1306.88              5.352986
 512        1000           0.00               2608.38              5.341967
 1024       1000           0.00               5093.40              5.215638
 2048       1000           0.00               8031.43              4.112093
 4096       1000           0.00               8018.60              2.052763
 8192       1000           0.00               11051.50             1.414592
 16384      1000           0.00               11482.77             0.734897
 32768      1000           0.00               11514.51             0.368464
 65536      1000           0.00               11516.88             0.184270
 131072     1000           0.00               11523.18             0.092185
 262144     1000           0.00               11526.74             0.046107
 524288     1000           0.00               11507.83             0.023016
 1048576    1000           0.00               11494.89             0.011495
 2097152    1000           0.00               11510.98             0.005755
 4194304    1000           0.00               11526.34             0.002882
 8388608    1000           0.00               11524.21             0.001441
---------------------------------------------------------------------------------------
```

Client
```
avhc21:~ # ib_send_bw -a 172.16.1.28
---------------------------------------------------------------------------------------
                    Send BW Test
 Dual-port       : OFF          Device         : mlx5_0
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 TX depth        : 128
 CQ Moderation   : 100
 Mtu             : 4096[B]
 Link type       : IB
 Max inline data : 0[B]
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0x2e8 QPN 0x0889 PSN 0xd12b22
 remote address: LID 0x2c0 QPN 0x0889 PSN 0x313ab1
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 2          1000           9.88               9.73                 5.100508
 4          1000           19.99              19.97                5.235632
 8          1000           39.90              39.87                5.226104
 16         1000           80.12              80.02                5.244317
 32         1000           160.24             160.07               5.245032
 64         1000           320.48             320.09               5.244337
 128        1000           638.48             638.23               5.228376
 256        1000           1267.12            1264.57              5.179696
 512        1000           2529.36            2523.15              5.167415
 1024       1000           4944.62            4925.47              5.043684
 2048       1000           7828.98            7822.09              4.004909
 4096       1000           7905.45            7903.68              2.023341
 8192       1000           10886.86            10886.39            1.393457
 16384      1000           11378.37            11376.72            0.728110
 32768      1000           11448.01            11446.87            0.366300
 65536      1000           11469.84            11469.23            0.183508
 131072     1000           11493.34            11493.20            0.091946
 262144     1000           11505.71            11505.68            0.046023
 524288     1000           11489.90            11489.83            0.022980
 1048576    1000           11480.01            11480.01            0.011480
 2097152    1000           11512.62            11497.92            0.005749
 4194304    1000           11513.56            11513.55            0.002878
 8388608    1000           11513.03            11511.92            0.001439
---------------------------------------------------------------------------------------
```

### Test TCP throughput of ib0 interface using iperf3 on CentOS 7.6

```
[root@avhc11 ~]# iperf3 -c 172.16.1.29
Connecting to host 172.16.1.29, port 5201
[  4] local 172.16.1.28 port 37674 connected to 172.16.1.29 port 5201
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-1.00   sec  2.17 GBytes  18.7 Gbits/sec    0   1.33 MBytes
[  4]   1.00-2.00   sec  2.21 GBytes  19.0 Gbits/sec    0   1.33 MBytes
[  4]   2.00-3.00   sec  2.24 GBytes  19.2 Gbits/sec    0   1.34 MBytes
[  4]   3.00-4.00   sec  2.23 GBytes  19.2 Gbits/sec   68   1.03 MBytes
[  4]   4.00-5.00   sec  2.19 GBytes  18.8 Gbits/sec    0   1.16 MBytes
[  4]   5.00-6.00   sec  2.17 GBytes  18.6 Gbits/sec    1    883 KBytes
[  4]   6.00-7.00   sec  2.19 GBytes  18.8 Gbits/sec    0    963 KBytes
[  4]   7.00-8.00   sec  2.19 GBytes  18.8 Gbits/sec    0   1004 KBytes
[  4]   8.00-9.00   sec  2.19 GBytes  18.8 Gbits/sec    0   1.02 MBytes
[  4]   9.00-10.00  sec  2.19 GBytes  18.8 Gbits/sec    0   1.04 MBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-10.00  sec  22.0 GBytes  18.9 Gbits/sec   69             sender
[  4]   0.00-10.00  sec  22.0 GBytes  18.9 Gbits/sec                  receiver

iperf Done.
```

