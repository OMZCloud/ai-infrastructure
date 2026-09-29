# OpenStack GPU Integration

- [OpenStack GPU Integration](#openstack-gpu-integration)
  - [1. Architecture](#1-architecture)
  - [2. PCI Passthrough](#2-pci-passthrough)
    - [2.1. Installation](#21-installation)
    - [2.2. Benchmark GPU in VM](#22-benchmark-gpu-in-vm)

## 1. Architecture

The [PCI Passthrough feature in OpenStack](https://docs.openstack.org/nova/latest/admin/pci-passthrough.html) allows compute hosts to provide physical PCI devices to virtual machines. Based on this capability, virtual machines can use GPUs, NICs, or other devices from the physical host.

The GPU deployment architecture in OpenStack is basically similar to other OpenStack deployment models, consisting of one or more controller nodes combined with compute nodes that can be added at any time. Compute nodes may or may not have GPUs, depending on the physical server configuration and the need to provide resources such as RAM/CPU or GPU to virtual machines. To automatically schedule virtual machines to the correct compute hosts that have the required GPU (PCI devices), users must select a flavor configured with **pci_passthrough:alias** according to number GPU needs (this detailed configuration will be described in the following section).

![OpenStack GPU architecture](/material/images/openstack-gpu-architecture.png)

## 2. PCI Passthrough

### 2.1. Installation

**Reference**

- [https://docs.openstack.org/nova/2025.1/admin/pci-passthrough.html](https://docs.openstack.org/nova/2025.1/admin/pci-passthrough.html)

- [https://openmetal.io/docs/manuals/private-ai/engineering-notes/preparing-nodes](https://openmetal.io/docs/manuals/private-ai/engineering-notes/preparing-nodes)

**Prerequisite**

- Intel VT-d enabled in BIOS → Already enabled as command below showed

- IOMMU enabled on the host OS kernel → Need to config in kernel params, config below

- GPU PCI ready for KVM/QEMU → Need to blacklist GPU from host. Because if GPU is already attached to nvidia driver on the host, it cannot be re-bound to VFIO (VFIO needs direct control of hardware), config below

```bash
root@omz-gpu:~# dmesg | grep -e DMAR -e IOMMU
…
[ 5.512991] DMAR: Intel(R) Virtualization Technology for Directed I/O
```

**Pre-check system**

- **10de**:20b2 is Vendor ID, 10de:**20b2** is Product ID/Device ID which will be used to config in OpenStack Nova later

```bash
root@omz-gpu:~# lspci | grep 80GB
10:00.0 3D controller: NVIDIA Corporation GA100 [A100 SXM4 80GB] (rev a1)
16:00.0 3D controller: NVIDIA Corporation GA100 [A100 SXM4 80GB] (rev a1)
49:00.0 3D controller: NVIDIA Corporation GA100 [A100 SXM4 80GB] (rev a1)
4d:00.0 3D controller: NVIDIA Corporation GA100 [A100 SXM4 80GB] (rev a1)
8a:00.0 3D controller: NVIDIA Corporation GA100 [A100 SXM4 80GB] (rev a1)
8f:00.0 3D controller: NVIDIA Corporation GA100 [A100 SXM4 80GB] (rev a1)
c6:00.0 3D controller: NVIDIA Corporation GA100 [A100 SXM4 80GB] (rev a1)
ca:00.0 3D controller: NVIDIA Corporation GA100 [A100 SXM4 80GB] (rev a1)

root@omz-gpu:~# lspci -nn | grep NVIDIA
10:00.0 3D controller [0302]: NVIDIA Corporation GA100 [A100 SXM4 80GB] [10de:20b2] (rev a1)
16:00.0 3D controller [0302]: NVIDIA Corporation GA100 [A100 SXM4 80GB] [10de:20b2] (rev a1)
49:00.0 3D controller [0302]: NVIDIA Corporation GA100 [A100 SXM4 80GB] [10de:20b2] (rev a1)
4d:00.0 3D controller [0302]: NVIDIA Corporation GA100 [A100 SXM4 80GB] [10de:20b2] (rev a1)
54:00.0 Bridge [0680]: NVIDIA Corporation GA100 [A100 NVSwitch] [10de:1af1] (rev a1)
55:00.0 Bridge [0680]: NVIDIA Corporation GA100 [A100 NVSwitch] [10de:1af1] (rev a1)
56:00.0 Bridge [0680]: NVIDIA Corporation GA100 [A100 NVSwitch] [10de:1af1] (rev a1)
57:00.0 Bridge [0680]: NVIDIA Corporation GA100 [A100 NVSwitch] [10de:1af1] (rev a1)
58:00.0 Bridge [0680]: NVIDIA Corporation GA100 [A100 NVSwitch] [10de:1af1] (rev a1)
59:00.0 Bridge [0680]: NVIDIA Corporation GA100 [A100 NVSwitch] [10de:1af1] (rev a1)
8a:00.0 3D controller [0302]: NVIDIA Corporation GA100 [A100 SXM4 80GB] [10de:20b2] (rev a1)
8f:00.0 3D controller [0302]: NVIDIA Corporation GA100 [A100 SXM4 80GB] [10de:20b2] (rev a1)
c6:00.0 3D controller [0302]: NVIDIA Corporation GA100 [A100 SXM4 80GB] [10de:20b2] (rev a1)
ca:00.0 3D controller [0302]: NVIDIA Corporation GA100 [A100 SXM4 80GB] [10de:20b2] (rev a1)
```

- Manual test modprobe **vfio-pci** module to enable that we can enable it by default later

```bash
root@omz-gpu:~# modprobe vfio-pci
root@omz-gpu:~# lsmod | grep vfio
vfio_pci 16384 0
vfio_pci_core 90112 1 vfio_pci
vfio_iommu_type1 49152 0
vfio 69632 3 vfio_pci_core,vfio_iommu_type1,vfio_pci
iommufd 98304 1 vfio
irqbypass 12288 19 vfio_pci_core,kvm
```

- More detail about hardware architecture: 8 GPU distributed in 2 NUMA, 4 GPU on each NUMA

![lstopo hardware topology](/material/images/ls-topo.png)

**Blacklist and enable modules**

```bash
echo "blacklist nouveau" >> /etc/modprobe.d/blacklist-nvidia.conf
echo "blacklist nvidiafb" >> /etc/modprobe.d/blacklist-nvidia.conf
echo vfio-pci >> /etc/modules-load.d/vfio-pci.conf
echo options vfio-pci ids=10de:20b2 >> /etc/modprobe.d/gpu-vfio.conf
```

**Config kernel params**

Edit /etc/default/grub file following below content

- `intel_iommu=on` allowing the system to manage device isolation and virtualization

- `iommu=pt` ensures performance is not degraded by unnecessary address translations

```bash
root@omz-gpu:~# cat /etc/default/grub | grep LINUX
GRUB_CMDLINE_LINUX="intel_iommu=on iommu=pt"
```

**Update grub and reboot server**

```bash
update-grub
reboot
```

After server up and running, ensure that vfio-pci is controlling GPU

```bash
root@omz-gpu ~(openstack)# for PCI_ID in $(lspci | grep NVIDIA | cut -d" " -f1); do
sudo lspci -s ${PCI_ID} -k
done
10:00.0 3D controller: NVIDIA Corporation GA100 [A100 SXM4 80GB] (rev a1)
Subsystem: NVIDIA Corporation GA100 [A100 SXM4 80GB]
Kernel driver in use: vfio-pci
Kernel modules: nvidiafb, nouveau
16:00.0 3D controller: NVIDIA Corporation GA100 [A100 SXM4 80GB] (rev a1)
Subsystem: NVIDIA Corporation GA100 [A100 SXM4 80GB]
Kernel driver in use: vfio-pci
Kernel modules: nvidiafb, nouveau
49:00.0 3D controller: NVIDIA Corporation GA100 [A100 SXM4 80GB] (rev a1)
Subsystem: NVIDIA Corporation GA100 [A100 SXM4 80GB]
Kernel driver in use: vfio-pci
Kernel modules: nvidiafb, nouveau
4d:00.0 3D controller: NVIDIA Corporation GA100 [A100 SXM4 80GB] (rev a1)
Subsystem: NVIDIA Corporation GA100 [A100 SXM4 80GB]
Kernel driver in use: vfio-pci
Kernel modules: nvidiafb, nouveau
54:00.0 Bridge: NVIDIA Corporation GA100 [A100 NVSwitch] (rev a1)
Subsystem: NVIDIA Corporation GA100 [A100 NVSwitch]
55:00.0 Bridge: NVIDIA Corporation GA100 [A100 NVSwitch] (rev a1)
Subsystem: NVIDIA Corporation GA100 [A100 NVSwitch]
56:00.0 Bridge: NVIDIA Corporation GA100 [A100 NVSwitch] (rev a1)
Subsystem: NVIDIA Corporation GA100 [A100 NVSwitch]
57:00.0 Bridge: NVIDIA Corporation GA100 [A100 NVSwitch] (rev a1)
Subsystem: NVIDIA Corporation GA100 [A100 NVSwitch]
58:00.0 Bridge: NVIDIA Corporation GA100 [A100 NVSwitch] (rev a1)
Subsystem: NVIDIA Corporation GA100 [A100 NVSwitch]
59:00.0 Bridge: NVIDIA Corporation GA100 [A100 NVSwitch] (rev a1)
Subsystem: NVIDIA Corporation GA100 [A100 NVSwitch]
8a:00.0 3D controller: NVIDIA Corporation GA100 [A100 SXM4 80GB] (rev a1)
Subsystem: NVIDIA Corporation GA100 [A100 SXM4 80GB]
Kernel driver in use: vfio-pci
Kernel modules: nvidiafb, nouveau
8f:00.0 3D controller: NVIDIA Corporation GA100 [A100 SXM4 80GB] (rev a1)
Subsystem: NVIDIA Corporation GA100 [A100 SXM4 80GB]
Kernel driver in use: vfio-pci
Kernel modules: nvidiafb, nouveau
c6:00.0 3D controller: NVIDIA Corporation GA100 [A100 SXM4 80GB] (rev a1)
Subsystem: NVIDIA Corporation GA100 [A100 SXM4 80GB]
Kernel driver in use: vfio-pci
Kernel modules: nvidiafb, nouveau
ca:00.0 3D controller: NVIDIA Corporation GA100 [A100 SXM4 80GB] (rev a1)
Subsystem: NVIDIA Corporation GA100 [A100 SXM4 80GB]
Kernel driver in use: vfio-pci
Kernel modules: nvidiafb, nouveau
```

**Reconfigure OpenStack nova**

- Update /etc/kolla/nova-compute/nova.conf and restart **nova-compute** container

  - `10de` and `20b2` split from 10de:20b2 we get before

  - `mygpu` is anyname just for mapping

```bash
[pci]
device_spec = { "vendor_id": "10de", "product_id": "20b2" }
alias = { "vendor_id":"10de", "product_id":"20b2", "device_type":"type-PF", "name":"mygpu" }
```

- Update /etc/kolla/nova-api/nova.conf and restart **nova-api** container

  - `10de` and `20b2` split from 10de:20b2 we get before

  - `mygpu` is anyname just for mapping

```bash
[pci]
alias = { "vendor_id":"10de", "product_id":"20b2", "device_type":"type-PF", "name":"mygpu" }
```

- Update /etc/kolla/nova-scheduler/nova.conf and restart **nova-scheduler** container

  - `PciPassthroughFilter` will help us to find which compute has GPU to provide as defined in flavor

  - Remain filters is just default filters [https://docs.openstack.org/nova/2025.1/configuration/config.html#filter_scheduler.enabled_filters](https://docs.openstack.org/nova/2025.1/configuration/config.html#filter_scheduler.enabled_filters)

```bash
[filter_scheduler]
available_filters = nova.scheduler.filters.all_filters
enabled_filters = PciPassthroughFilter,ComputeFilter,ComputeCapabilitiesFilter,ImagePropertiesFilter,ServerGroupAntiAffinityFilter,ServerGroupAffinityFilter
```

**Create flavor and VM on OpenStack**

- Create flavor over commandline

```bash
openstack flavor create gpu-8c64g1gpu --vcpus 8 --ram 64000 --disk 100 --property "pci_passthrough:alias"="mygpu:1"
openstack flavor create gpu-8c64g3gpu --vcpus 8 --ram 64000 --disk 100 --property "pci_passthrough:alias"="mygpu:3"
```

Then create VM from horizon as normal, the VM should have GPU as we configure for flavor.

![VM with GPU (1)](/material/images/1-gpu-vm.png)

![VM with GPU (2)](/material/images/3-gpu-vm.png)

Verify which GPU are using

- GPU `0000:8a:00.0` attach to VM ID 3

- GPU `0000:8f:00.0`, `0000:c6:00.0`, `0000:10:00.0` attach to VM ID 4

```bash
root@omz-gpu:~# ls /sys/bus/pci/drivers/vfio-pci/
0000:10:00.0 0000:49:00.0 0000:8a:00.0 0000:c6:00.0 bind new_id uevent
0000:16:00.0 0000:4d:00.0 0000:8f:00.0 0000:ca:00.0 module remove_id unbind
(nova-libvirt)[root@omz-gpu /]# virsh dumpxml 3 | grep hostdev -A5
<hostdev mode='subsystem' type='pci' managed='yes'>
<driver name='vfio'/>
<source>
<address domain='0x0000' bus='0x8a' slot='0x00' function='0x0'/>
</source>
<alias name='hostdev0'/>
<address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>
</hostdev>
<memballoon model='virtio'>
<stats period='10'/>
<alias name='balloon0'/>
<address type='pci' domain='0x0000' bus='0x00' slot='0x07' function='0x0'/>
</memballoon>

(nova-libvirt)[root@omz-gpu /]# virsh dumpxml 4 | grep hostdev -A5
<hostdev mode='subsystem' type='pci' managed='yes'>
<driver name='vfio'/>
<source>
<address domain='0x0000' bus='0x10' slot='0x00' function='0x0'/>
</source>
<alias name='hostdev0'/>
<address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>
</hostdev>
<hostdev mode='subsystem' type='pci' managed='yes'>
<driver name='vfio'/>
<source>
<address domain='0x0000' bus='0x8f' slot='0x00' function='0x0'/>
</source>
<alias name='hostdev1'/>
<address type='pci' domain='0x0000' bus='0x00' slot='0x07' function='0x0'/>
</hostdev>
<hostdev mode='subsystem' type='pci' managed='yes'>
<driver name='vfio'/>
<source>
<address domain='0x0000' bus='0xc6' slot='0x00' function='0x0'/>
</source>
<alias name='hostdev2'/>
<address type='pci' domain='0x0000' bus='0x00' slot='0x08' function='0x0'/>
</hostdev>
<memballoon model='virtio'>
<stats period='10'/>
<alias name='balloon0'/>
<address type='pci' domain='0x0000' bus='0x00' slot='0x09' function='0x0'/>
</memballoon>
```

### 2.2. Benchmark GPU in VM

**Geekbench 6**

Result: [https://browser.geekbench.com/v6/compute/4656106](https://browser.geekbench.com/v6/compute/4656106)

- Install nvidia driver, then reboot

```bash
apt install ubuntu-drivers-common
ubuntu-drivers devices
nvidia-detector
apt install nvidia-driver-575
reboot
```

- Benchmark with geekbench - Single GPU

```bash
wget https://cdn.geekbench.com/Geekbench-6.4.0-Linux.tar.gz
tar -xvf Geekbench-6.4.0-Linux.tar.gz
cd Geekbench-6.4.0-Linux/
./geekbench6 --gpu-list
./geekbench6 --gpu vulkan
```

**Cuda samples**

- Ensure our GPU is supported by Cuda [https://developer.nvidia.com/cuda-gpus](https://developer.nvidia.com/cuda-gpus)

- Install Cuda drivers Actions: [https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&Distribution=Ubuntu&target_version=24.04&target_type=deb_local](https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&Distribution=Ubuntu&target_version=24.04&target_type=deb_local)

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-ubuntu2404.pin
sudo mv cuda-ubuntu2404.pin /etc/apt/preferences.d/cuda-repository-pin-600
wget https://developer.download.nvidia.com/compute/cuda/13.0.0/local_installers/cuda-repo-ubuntu2404-13-0-local_13.0.0-580.65.06-1_amd64.deb
sudo dpkg -i cuda-repo-ubuntu2404-13-0-local_13.0.0-580.65.06-1_amd64.deb
sudo cp /var/cuda-repo-ubuntu2404-13-0-local/cuda-*-keyring.gpg /usr/share/keyrings/
sudo apt-get update
sudo apt-get -y install cuda-toolkit-13-0
sudo apt-get install -y cuda-drivers
reboot
```

- Config PATH for Cuda

```bash
export PATH=${PATH}:/usr/local/cuda-13.0/bin
export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:/usr/local/cuda-13.0/lib64
```

- Test with official cuda-samples

Build tool

```bash
git clone https://github.com/nvidia/cuda-samples
apt install cmake
cd cuda-samples/
mkdir build && cd build
cmake ..
make -j$(nproc)
find . -type f -perm -755 # find all test file or filter by grep
```

Single GPU Test

```bash
./Samples/6_Performance/alignedTypes/alignedTypes
./Samples/0_Introduction/matrixMul/matrixMul
./Samples/4_CUDA_Libraries/matrixMulCUBLAS/matrixMulCUBLAS
./Samples/0_Introduction/simpleMultiCopy/simpleMultiCopy
./Samples/3_CUDA_Features/cudaTensorCoreGemm/cudaTensorCoreGemm
./Samples/3_CUDA_Features/bf16TensorCoreGemm/bf16TensorCoreGemm
./Samples/3_CUDA_Features/immaTensorCoreGemm/immaTensorCoreGemm
./Samples/3_CUDA_Features/tf32TensorCoreGemm/tf32TensorCoreGemm
./Samples/3_CUDA_Features/dmmaTensorCoreGemm/dmmaTensorCoreGemm
```

```bash
root@3gpu:~/cuda-samples/build# ./Samples/6_Performance/alignedTypes/alignedTypes
[./Samples/6_Performance/alignedTypes/alignedTypes] - Starting...
GPU Device 0: "Ampere" with compute capability 8.0
[NVIDIA A100-SXM4-80GB] has 108 MP(s) x 64 (Cores/MP) = 6912 (Cores)
> Compute scaling value = 1.00
> Memory Size = 49999872
Allocating memory...
Generating host input data array...
Uploading input data to GPU memory...
Testing misaligned types...
uint8...
Avg. time: 1.583031 ms / Copy throughput: 29.415723 GB/s.
TEST OK
uint16...
Avg. time: 0.861313 ms / Copy throughput: 54.064012 GB/s.
TEST OK
RGBA8_misaligned...
Avg. time: 0.507531 ms / Copy throughput: 91.750039 GB/s.
TEST OK
LA32_misaligned...
Avg. time: 0.258563 ms / Copy throughput: 180.095755 GB/s.
TEST OK
RGB32_misaligned...
Avg. time: 0.202812 ms / Copy throughput: 229.601288 GB/s.
TEST OK
RGBA32_misaligned...
Avg. time: 0.145250 ms / Copy throughput: 320.592164 GB/s.
TEST OK
Testing aligned types...
RGBA8...
Avg. time: 0.465594 ms / Copy throughput: 100.014248 GB/s.
TEST OK
I32...
Avg. time: 0.465313 ms / Copy throughput: 100.074699 GB/s.
TEST OK
LA32...
Avg. time: 0.250500 ms / Copy throughput: 185.892258 GB/s.
TEST OK
RGB32...
Avg. time: 0.143625 ms / Copy throughput: 324.219374 GB/s.
TEST OK
RGBA32...
Avg. time: 0.143562 ms / Copy throughput: 324.360546 GB/s.
TEST OK
RGBA32_2...
Avg. time: 0.088250 ms / Copy throughput: 527.660187 GB/s.
TEST OK
[alignedTypes] -> Test Results: 0 Failures
Shutting down...
Test passed

root@3gpu:~/cuda-samples/build# ./Samples/0_Introduction/matrixMul/matrixMul
[Matrix Multiply Using CUDA] - Starting...
GPU Device 0: "Ampere" with compute capability 8.0
MatrixA(320,320), MatrixB(640,320)
Computing result using CUDA Kernel...
done
Performance= 2845.71 GFlop/s, Time= 0.046 msec, Size= 131072000 Ops, WorkgroupSize= 1024 threads/block
Checking computed result for correctness: Result = PASS
NOTE: The CUDA Samples are not meant for performance measurements. Results may vary when GPU Boost is enabled.

root@3gpu:~/cuda-samples/build# ./Samples/4_CUDA_Libraries/matrixMulCUBLAS/matrixMulCUBLAS
[Matrix Multiply CUBLAS] - Starting...
GPU Device 0: "Ampere" with compute capability 8.0
GPU Device 0: "NVIDIA A100-SXM4-80GB" with compute capability 8.0
MatrixA(640,480), MatrixB(480,320), MatrixC(640,320)
Computing result using CUBLAS...done.
Performance= 9028.21 GFlop/s, Time= 0.022 msec, Size= 196608000 Ops
Computing result using host CPU...done.
Comparing CUBLAS Matrix Multiply with CPU results: PASS
NOTE: The CUDA Samples are not meant for performance measurements. Results may vary when GPU Boost is enabled.

root@3gpu:~/cuda-samples/build# ./Samples/0_Introduction/simpleMultiCopy/simpleMultiCopy
[simpleMultiCopy] - Starting...
> Using CUDA device [0]: NVIDIA A100-SXM4-80GB
[NVIDIA A100-SXM4-80GB] has 108 MP(s) x 64 (Cores/MP) = 6912 (Cores)
> Device name: NVIDIA A100-SXM4-80GB
> CUDA Capability 8.0 hardware with 108 multi-processors
> scale_factor = 1.00
> array_size = 4194304
Relevant properties of this CUDA device
(X) Can overlap one CPU<>GPU data transfer with GPU kernel execution (device property "cudaDevAttrGpuOverlap")
(X) Can overlap two CPU<>GPU data transfers with GPU kernel execution
(Compute Capability >= 2.0 AND (Tesla product OR Quadro 4000/5000/6000/K5000)
Measured timings (throughput):
Memcpy host to device : 0.686592 ms (24.435497 GB/s)
Memcpy device to host : 0.704832 ms (23.803141 GB/s)
Kernel : 0.054848 ms (3058.856453 GB/s)
Theoretical limits for speedup gained from overlapped data transfers:
No overlap at all (transfer-kernel-transfer): 1.446272 ms
Compute can overlap with one transfer: 1.391424 ms
Compute can overlap with both data transfers: 0.704832 ms
Average measured timings over 10 repetitions:
Avg. time when execution fully serialized : 1.445683 ms
Avg. time when overlapped using 4 streams : 0.847872 ms
Avg. speedup gained (serialized - overlapped) : 0.597811 ms
Measured throughput:
Fully serialized execution : 23.210087 GB/s
Overlapped using 4 streams : 39.574881 GB/s

root@3gpu:~/cuda-samples/build# ./Samples/3_CUDA_Features/cudaTensorCoreGemm/cudaTensorCoreGemm
Initializing...
GPU Device 0: "Ampere" with compute capability 8.0
M: 4096 (16 x 256)
N: 4096 (16 x 256)
K: 4096 (16 x 256)
Preparing data for GPU...
Required shared memory size: 64 Kb
Computing... using high performance kernel compute_gemm
Time: 2.770944 ms
TFLOPS: 49.60

root@3gpu:~/cuda-samples/build# ./Samples/3_CUDA_Features/bf16TensorCoreGemm/bf16TensorCoreGemm
Initializing...
GPU Device 0: "Ampere" with compute capability 8.0
M: 8192 (16 x 512)
N: 8192 (16 x 512)
K: 8192 (16 x 512)
Preparing data for GPU...
Required shared memory size: 72 Kb
Computing using high performance kernel = 0 - compute_bf16gemm_async_copy
Time: 11.254784 ms
TFLOPS: 97.69

root@3gpu:~/cuda-samples/build# ./Samples/3_CUDA_Features/immaTensorCoreGemm/immaTensorCoreGemm
Initializing...
GPU Device 0: "Ampere" with compute capability 8.0
M: 4096 (16 x 256)
N: 4096 (16 x 256)
K: 4096 (16 x 256)
Preparing data for GPU...
Required shared memory size: 64 Kb
Computing... using high performance kernel compute_gemm_imma
Time: 1.685504 ms
TOPS: 81.54

root@3gpu:~/cuda-samples/build# ./Samples/3_CUDA_Features/tf32TensorCoreGemm/tf32TensorCoreGemm
Initializing...
GPU Device 0: "Ampere" with compute capability 8.0
M: 8192 (16 x 512)
N: 8192 (16 x 512)
K: 4096 (8 x 512)
Preparing data for GPU...
Required shared memory size: 72 Kb
Computing using high performance kernel = 0 - compute_tf32gemm_async_copy
Time: 42.252289 ms
TFLOPS: 13.01

root@3gpu:~/cuda-samples/build# ./Samples/3_CUDA_Features/dmmaTensorCoreGemm/dmmaTensorCoreGemm
Initializing...
GPU Device 0: "Ampere" with compute capability 8.0
M: 8192 (8 x 1024)
N: 8192 (8 x 1024)
K: 4096 (4 x 1024)
Preparing data for GPU...
Required shared memory size: 68 Kb
Computing using high performance kernel = 0 - compute_dgemm_async_copy
Time: 51.444736 ms
FP64 TFLOPS: 10.69
```

Multi GPU Test

```bash
./Samples/0_Introduction/simpleMultiGPU/simpleMultiGPU
./Samples/5_Domain_Specific/MonteCarloMultiGPU/MonteCarloMultiGPU
```

```bash
root@3gpu:~/cuda-samples/build# ./Samples/0_Introduction/simpleMultiGPU/simpleMultiGPU
Starting simpleMultiGPU
CUDA-capable device count: 3
Generating input data...
Computing with 3 GPUs...
GPU Processing time: 7.366000 (ms)
Computing with Host CPU...
Comparing GPU and Host CPU results...
GPU sum: 16777280.000000
CPU sum: 16777294.395033
Relative difference: 8.580068E-07

root@3gpu:~/cuda-samples/build# ./Samples/5_Domain_Specific/MonteCarloMultiGPU/MonteCarloMultiGPU
./Samples/5_Domain_Specific/MonteCarloMultiGPU/MonteCarloMultiGPU Starting...
Using single CPU thread for multiple GPUs
MonteCarloMultiGPU
==================
Parallelization method = streamed
Problem scaling = weak
Number of GPUs = 3
Total number of options = 24576
Number of paths = 262144
main(): generating input data...
main(): starting 3 host threads...
main(): GPU statistics, streamed
GPU Device #0: NVIDIA A100-SXM4-80GB
Options : 8192
Simulation paths: 262144
GPU Device #1: NVIDIA A100-SXM4-80GB
Options : 8192
Simulation paths: 262144
GPU Device #2: NVIDIA A100-SXM4-80GB
Options : 8192
Simulation paths: 262144
Total time (ms.): 8.633000
Note: This is elapsed time for all to compute.
Options per sec.: 2846750.716526
main(): comparing Monte Carlo and Black-Scholes results...
Shutting down...
Test Summary...
L1 norm : 4.764930E-04
Average reserve: 13.197989
NOTE: The CUDA Samples are not meant for performance measurements. Results may vary when GPU Boost is enabled.
Test passed
```

**Ollama - Single GPU**

Install Ollama and test run

```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama run deepseek-r1:70b
```

Ensure Ollama serve before run test

```bash
root@3gpu:~# ps aux | grep ollama
ollama 21917 0.8 0.1 14840944 58244 ? Ssl 08:24 0:00 /usr/local/bin/ollama serve
root 23950 0.0 0.0 7076 2176 pts/3 S+ 08:26 0:00 grep --color=auto ollama
```

Install benchmark tool

```bash
git clone https://github.com/larryhopecode/ollama-benchmark
apt install python3-venv
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
python benchmark.py
```

Meanwhile, open another tab to see how many GPUs are using.

![GPU usage during Ollama benchmark](/material/images/gpu-ollama-benchmark.png)

Result

```bash
(venv) root@3gpu:~/ollama-benchmark# python benchmark.py
----------------------------------------------------
Model: deepseek-r1:70b
Performance Metrics:
Prompt Processing: 342.36 tokens/sec
Generation Speed: 22.56 tokens/sec
Combined Speed: 22.95 tokens/sec
Workload Stats:
Input Tokens: 165
Generated Tokens: 8785
Model Load Time: 1.26s
Processing Time: 0.48s
Generation Time: 389.43s
Total Time: 391.18s
----------------------------------------------------
```

Compared with these reports, our GPU is similar.

- [https://www.databasemart.com/blog/deepseek-r1-70b-gpu-hosting](https://www.databasemart.com/blog/deepseek-r1-70b-gpu-hosting)

- [https://docs.google.com/spreadsheets/d/1dnMCBeUYHGDB2inBl6fQhaQBstI_G199qESJBxS3FWk/edit?gid=0#gid=0](https://docs.google.com/spreadsheets/d/1dnMCBeUYHGDB2inBl6fQhaQBstI_G199qESJBxS3FWk/edit?gid=0#gid=0)

- [https://www.reddit.com/r/LocalLLaMA/comments/1i69dhz/deepseek_r1_ollama_hardware_benchmark_for_localllm/](https://www.reddit.com/r/LocalLLaMA/comments/1i69dhz/deepseek_r1_ollama_hardware_benchmark_for_localllm/)
