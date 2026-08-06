# Gfxstream Vulkan & `VIRTGPU_BLOB_MEM_HOST3D` Investigation & Diagnosis

This document provides a comprehensive summary of the architecture investigation, empirical command outputs, C experiment results, root-cause analysis, and system-wide Vulkan Implicit Layer implementation for graphics virtualization (**Gfxstream / AVF / VirtIO-GPU**) on the **Pixel 10** (Android 16 / Linux 6.12 AVF Debian VM).

---

## Executive Summary

1. **Host Hypervisor (`crosvm`)**: Verified to run with `--gpu=backend=gfxstream` and renderer features `ExternalBlob:enabled` and `VulkanAllocateHostVisibleAsUdmabuf:enabled`.
2. **Guest VM Kernel (Linux 6.12)**: Confirmed runtime negotiation of `+resource_blob` (`virtio_gpu_object_create_blob`) and `+host_visible` (`VIRTGPU_BLOB_MEM_HOST3D`) with a 512 MB host-visible memory window (`0x200000000 +0x200000000`).
3. **Guest Mesa ICD (`libvulkan_gfxstream.so`)**: Confirmed to expose `VK_EXT_external_memory_dma_buf` and `VK_KHR_external_memory_fd` for host **PowerVR D-Series** GPU forwarding.
4. **`journalctl` Mesa Error Investigation**: Isolated the root cause of `MESA: error: drmPrimeHandleToFD failed with No such file or directory` during GUI application launches (e.g. PCManFM-Qt under `labwc`).
5. **Experimental Proof**: Proved via a custom C Vulkan test suite that Gfxstream requires **`VkMemoryDedicatedAllocateInfo::image`** alongside `VkExportMemoryAllocateInfo` to export DMA-BUF file descriptors (`FD = 5`). Non-dedicated surface allocations fail DMA-BUF export and fall back to software linear (`sw,linear` / `wl_shm`) rendering.
6. **System-Wide Resolution (Path 3 Implemented)**: Deployed a global **Vulkan Implicit Layer Shim (`libvulkan_dedicated_shim.so`)** in `/usr/lib/aarch64-linux-gnu/` and `/usr/share/vulkan/implicit_layer.d/VkLayer_gfxstream_dedicated_shim.json`. The layer automatically injects dedicated image allocation metadata into `vkAllocateMemory` calls system-wide, allowing zero-copy DMA-BUF exports for all applications without log errors.

---

## 1. Host Hypervisor (`crosvm`) & ADB Investigation

ADB connection was established with the host Pixel 10 to inspect running virtualization daemons and hypervisor process arguments.

### Active Virtualization Services
```bash
adb shell "ps -A | grep -iE 'virt|crosvm'"
```
**Raw Output:**
```text
system        4088     1   11064356   5352 0 S virtualizationservice
u0_a276       4172  4026   11388776  95060 0 S crosvm_isolated_storage_service_vm
u0_a294       7510 22541   16339304 1125084 0 S crosvm_debian
u0_a294      26629  1297   43930236 198080 0 S com.android.virtualization.terminal
```

### Crosvm Process Command-Line Arguments
```bash
adb shell "tr '\0' ' ' < /proc/7510/cmdline"
```
**Raw Output:**
```text
crosvm_debian --extended-status --log-level debug,disk=warn run --disable-sandbox --cid 2058 --name crosvm_debian /proc/self/fd/38 --params root=/dev/vda1 ds=nocloud arm64.nompam 8250.nr_uarts=4 console=ttyS0 debian_server_port=0 gfxstream_enabled --params hostname=debian --initrd /proc/self/fd/39 --host-cpu-topology --virt-cpufreq --cpus sve=[auto=true] --pci mem=[start=0x2c000000,size=0x2000000],cam=[start=0x2e000000,size=0x1000000] --no-pmu --irqchip=kernel[allow-vgic-its=true] --mem 3009 --force-disable-readonly-mem --params coherent_pool=4096 --balloon-page-reporting --serial=type=file,path=/proc/self/fd/40,hardware=serial,num=1 --serial=type=file,path=/proc/self/fd/40,hardware=virtio-console,num=1,max-queue-sizes=[2,32] --serial=type=file,path=/proc/self/fd/41,hardware=virtio-console,num=3,max-queue-sizes=[2,32] --serial=type=file,path=/proc/self/fd/43,hardware=serial,num=2 --serial=type=sink,hardware=virtio-console,num=2,max-queue-sizes=[2,32] --block path=/proc/self/fd/49,ro=false,lock=false --block path=/proc/self/fd/45,ro=true,lock=false --android-display-service=debian --gpu=backend=gfxstream,context-types=gfxstream-vulkan:gfxstream-composer,renderer-features=VulkanDisableCoherentMemoryAndEmulate:enabled;VulkanAllocateHostVisibleAsUdmabuf:enabled;ExternalBlob:enabled,surfaceless=true,vulkan=true,displays=[[mode=windowed[1280,720],dpi=[160,160],refresh-rate=60]] --input multi-touch[path=/proc/self/fd/50,width=1280,height=720] --input keyboard[path=/proc/self/fd/51] --input mouse[path=/proc/self/fd/52] --input multi-touch-trackpad[path=/proc/self/fd/53,width=3000,height=3000] --no-usb --net tap-fd=54 --socket /proc/self/fd/16 --device-tree-overlay /proc/self/fd/35 --vhost-user fs,socket=/data/misc/virtualizationservice/2058/android --vhost-user sound,socket=/data/misc/virtualizationservice/2058/vhost_user_snd
```

---

## 2. Guest VM Kernel Verification (`VIRTGPU_BLOB_MEM_HOST3D`)

Executed directly inside the AVF Debian VM:

```bash
uname -a
sudo dmesg | grep -iE 'virtio|gpu|blob|drm'
```

**Raw Output:**
```text
Linux debian 6.12.89-android16-6-gf222c1f727d9-ab15712176-4k #1 SMP PREEMPT Wed Jun 24 07:17:20 UTC 2026 aarch64 GNU/Linux
[    0.000000] Kernel command line: panic=-1 root=/dev/vda1 ds=nocloud arm64.nompam 8250.nr_uarts=4 console=ttyS0 debian_server_port=0 gfxstream_enabled hostname=debian coherent_pool=4096
[    2.773425] [drm] pci: virtio-gpu-pci detected at 0000:00:01.0
[    2.781065] [drm] Host memory window: 0x200000000 +0x200000000
[    2.788826] [drm] features: +virgl +edid +resource_blob +host_visible
[    2.788840] [drm] features: +context_init
[    2.816585] [drm] number of scanouts: 16
[    2.819879] [drm] number of cap sets: 2
[    2.829952] [drm] cap set 0: id 3, max-version 0, max-size 112
[    2.838731] [drm] cap set 1: id 9, max-version 0, max-size 16
[    2.858907] [drm] Initialized virtio_gpu 0.1.0 for 0000:00:01.0 on minor 0
```

---

## 3. Guest Mesa Driver & Vulkan ICD Diagnostics

```bash
cat /usr/share/vulkan/icd.d/gfxstream_vk_icd.json
vulkaninfo 2>&1 | grep -iE "VK_EXT_external_memory_dma_buf|VK_KHR_external_memory|driverName"
```

**Raw Output:**
```json
{
    "ICD": {
        "api_version": "1.1.305",
        "library_path": "libvulkan_gfxstream.so"
    },
    "file_format_version": "1.0.0"
}
```
```text
	driverName                             = PowerVR D-Series Vulkan Driver
	VK_KHR_external_memory_capabilities    : extension revision 1
	VK_EXT_external_memory_dma_buf         : extension revision 1
	VK_KHR_external_memory                 : extension revision 1
	VK_KHR_external_memory_fd              : extension revision 1
```

---

## 4. `journalctl` Error Investigation & C Experiments

### Observed Log Errors in `journalctl`
When launching LXQt File Explorer (PCManFM-Qt) under `labwc`:
```text
Aug 06 02:03:05 debian labwc[499]: MESA: error: drmPrimeHandleToFD failed with No such file or directory
Aug 06 02:03:05 debian labwc[499]: MESA: error: on_vkGetMemoryFdKHR: Failed to export host resource to FD.
Aug 06 02:03:05 debian labwc[499]: MESA: error: MESA: Unable to get a valid memory fd
Aug 06 02:03:05 debian labwc[499]: MESA: error: DRM_IOCTL_GEM_CLOSE failed with : [Invalid argument, blobHandle 4, resourceHandle: 28]
```

### Empirical C Vulkan Test Suite

A custom Vulkan test script was compiled (`gcc -lvulkan`) and executed inside the guest VM to test `vkGetMemoryFdKHR` under different allocation modes:

#### Test 1: Standard `vkAllocateMemory` (No Export Flags)
* **Result**: `FD = -1`
* **Mesa Output**: `MESA: error: on_vkGetMemoryFdKHR: VkDeviceMemory does not have a resource available for export.`

#### Test 2: `vkAllocateMemory` + `VkExportMemoryAllocateInfo` (Generic Buffer)
* **Result**: `FD = -1`
* **Mesa Output**: `MESA: warning: VkDeviceMemory is not exportable (VkExportMemoryAllocateInfo). Requires VkMemoryDedicatedAllocateInfo::image to create external resource.`

#### Test 3: `vkAllocateMemory` + `VkExportMemoryAllocateInfo` + `VkMemoryDedicatedAllocateInfo::image`
```c
VkMemoryDedicatedAllocateInfo dedicatedInfo = {
    .sType = VK_STRUCTURE_TYPE_MEMORY_DEDICATED_ALLOCATE_INFO,
    .image = image
};
VkExportMemoryAllocateInfo exportInfo = {
    .sType = VK_STRUCTURE_TYPE_EXPORT_MEMORY_ALLOCATE_INFO,
    .pNext = &dedicatedInfo,
    .handleTypes = VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT
};
```
* **Result**: **`Result = 0 (VK_SUCCESS), FD = 5`**
* **Mesa Output**: **Zero Mesa error logs generated.**

---

## 5. System-Wide Implementation: Vulkan Implicit Layer Shim (Path 3)

To enforce dedicated allocations system-wide across all programs without requiring code changes to Qt6, GTK, or `labwc`, a **Vulkan Implicit Layer** (`VK_LAYER_GFXSTREAM_dedicated_shim`) was developed and installed.

### Layer Source Code (`/tmp/libvulkan_dedicated_shim.c`)
The layer intercepts `vkCreateImage`, `vkGetImageMemoryRequirements2`, and `vkAllocateMemory`. When memory allocation is requested, if `VkMemoryDedicatedAllocateInfo` is missing, the layer automatically injects a `VkMemoryDedicatedAllocateInfo` struct referencing the active `VkImage`.

### Installation Files:
1. **Shared Library**: `/usr/lib/aarch64-linux-gnu/libvulkan_dedicated_shim.so`
2. **Implicit Layer Manifest**: `/usr/share/vulkan/implicit_layer.d/VkLayer_gfxstream_dedicated_shim.json`

```json
{
    "file_format_version" : "1.0.0",
    "layer" : {
        "name": "VK_LAYER_GFXSTREAM_dedicated_shim",
        "type": "GLOBAL",
        "library_path": "/usr/lib/aarch64-linux-gnu/libvulkan_dedicated_shim.so",
        "api_version": "1.4.0",
        "implementation_version": "1",
        "description": "Automatic VkMemoryDedicatedAllocateInfo injector for Gfxstream DMA-BUF exports",
        "functions": {
            "vkGetInstanceProcAddr": "Shim_GetInstanceProcAddr",
            "vkGetDeviceProcAddr": "Shim_GetDeviceProcAddr"
        },
        "disable_environment": {
            "DISABLE_GFXSTREAM_DEDICATED_SHIM": "1"
        }
    }
}
```

### Verification:
With the layer installed system-wide:
* `vulkaninfo --summary` loads clean without warnings.
* `pcmanfm-qt`, `labwc`, and all desktop applications allocate Vulkan memory with dedicated image metadata automatically.
* Zero `drmPrimeHandleToFD` or `MESA: error` lines are logged to `journalctl`.
