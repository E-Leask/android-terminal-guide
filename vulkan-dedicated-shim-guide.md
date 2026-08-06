# Vulkan Implicit Layer Shim (`libvulkan_dedicated_shim.so`) Deployment & Reference Guide

This document provides complete instructions, full production C source code, Vulkan JSON manifest specs, and step-by-step deployment procedure to build, install, and configure the **Vulkan Implicit Layer Shim** (`VK_LAYER_GFXSTREAM_dedicated_shim`).

---

## Executive Summary & Architecture Problem

### Problem Statement
When running graphics-accelerated applications under **Gfxstream / AVF VirtIO-GPU** on Android (e.g. Pixel 10 PowerVR host GPU), exporting guest Vulkan memory to DMA-BUF file descriptors via `vkGetMemoryFdKHR` strictly requires:
* `VkExportMemoryAllocateInfo::handleTypes` = `VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT`
* **`VkMemoryDedicatedAllocateInfo::image`** = Valid `VkImage` handle

Standard Wayland compositors (`labwc`), Qt6 toolkits (`PCManFM-Qt`), and GTK desktop components omit `VkMemoryDedicatedAllocateInfo` when allocating surface memory. As a result, the Gfxstream pass-through driver (`libvulkan_gfxstream.so`) fails `drmPrimeHandleToFD` (`ENOENT`), causing applications to fall back to software linear (`sw,linear` / `wl_shm`) rendering with severe log error output.

### Solution Architecture
The **Vulkan Implicit Layer Shim** runs system-wide between Vulkan applications and the Vulkan Loader/Gfxstream driver. It automatically:
1. Intercepts `vkCreateImage`, `vkGetImageMemoryRequirements`, `vkGetImageMemoryRequirements2`, and `vkAllocateMemory`.
2. Maintains a thread-local queue of pending images matched by allocation size.
3. Automatically injects `VkMemoryDedicatedAllocateInfo::image` (and `VkExportMemoryAllocateInfo` if missing) into `pAllocateInfo->pNext` at allocation time.
4. Enforces clean zero-copy DMA-BUF exports across all applications without requiring code modifications to Qt, GTK, or Wayland compositors.

---

## 1. Complete C Source Code (`libvulkan_dedicated_shim.c`)

Save the following source code to `/tmp/libvulkan_dedicated_shim.c`:

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <pthread.h>
#include <dlfcn.h>
#include <vulkan/vulkan.h>
#include <vulkan/vk_layer.h>

#define MAX_PENDING 32
typedef struct {
    VkImage image;
    VkDeviceSize size;
    uint32_t memoryTypeBits;
} PendingImageInfo;

static __thread PendingImageInfo t_pending_images[MAX_PENDING];
static __thread int t_pending_count = 0;

typedef struct {
    PFN_vkGetDeviceProcAddr GetDeviceProcAddr;
    PFN_vkAllocateMemory AllocateMemory;
    PFN_vkCreateImage CreateImage;
    PFN_vkGetImageMemoryRequirements GetImageMemoryRequirements;
    PFN_vkGetImageMemoryRequirements2 GetImageMemoryRequirements2;
    PFN_vkDestroyImage DestroyImage;
} DeviceDispatchTable;

#define MAX_ENTRIES 32
static struct {
    VkDevice device;
    DeviceDispatchTable dispatch;
} g_device_map[MAX_ENTRIES];
static pthread_mutex_t g_device_map_lock = PTHREAD_MUTEX_INITIALIZER;

static struct {
    VkInstance instance;
    PFN_vkGetInstanceProcAddr gipa;
    PFN_vkEnumerateDeviceExtensionProperties pfnEnumerateDeviceExtensionProperties;
} g_instance_map[MAX_ENTRIES];
static pthread_mutex_t g_instance_map_lock = PTHREAD_MUTEX_INITIALIZER;

static DeviceDispatchTable* get_device_dispatch(VkDevice device) {
    pthread_mutex_lock(&g_device_map_lock);
    for (int i = 0; i < MAX_ENTRIES; i++) {
        if (g_device_map[i].device == device) {
            pthread_mutex_unlock(&g_device_map_lock);
            return &g_device_map[i].dispatch;
        }
    }
    pthread_mutex_unlock(&g_device_map_lock);
    return NULL;
}

static void set_device_dispatch(VkDevice device, DeviceDispatchTable dispatch) {
    pthread_mutex_lock(&g_device_map_lock);
    for (int i = 0; i < MAX_ENTRIES; i++) {
        if (g_device_map[i].device == VK_NULL_HANDLE || g_device_map[i].device == device) {
            g_device_map[i].device = device;
            g_device_map[i].dispatch = dispatch;
            pthread_mutex_unlock(&g_device_map_lock);
            return;
        }
    }
    pthread_mutex_unlock(&g_device_map_lock);
}

static PFN_vkGetInstanceProcAddr get_instance_gipa(VkInstance instance) {
    pthread_mutex_lock(&g_instance_map_lock);
    for (int i = 0; i < MAX_ENTRIES; i++) {
        if (g_instance_map[i].instance == instance) {
            pthread_mutex_unlock(&g_instance_map_lock);
            return g_instance_map[i].gipa;
        }
    }
    pthread_mutex_unlock(&g_instance_map_lock);
    return NULL;
}

static void set_instance_info(VkInstance instance, PFN_vkGetInstanceProcAddr gipa) {
    pthread_mutex_lock(&g_instance_map_lock);
    for (int i = 0; i < MAX_ENTRIES; i++) {
        if (g_instance_map[i].instance == VK_NULL_HANDLE || g_instance_map[i].instance == instance) {
            g_instance_map[i].instance = instance;
            g_instance_map[i].gipa = gipa;
            g_instance_map[i].pfnEnumerateDeviceExtensionProperties = 
                (PFN_vkEnumerateDeviceExtensionProperties)gipa(instance, "vkEnumerateDeviceExtensionProperties");
            pthread_mutex_unlock(&g_instance_map_lock);
            return;
        }
    }
    pthread_mutex_unlock(&g_instance_map_lock);
}

static void push_pending_image_reqs(VkImage img, VkDeviceSize size, uint32_t memBits) {
    if (img == VK_NULL_HANDLE) return;
    for (int i = 0; i < t_pending_count; i++) {
        if (t_pending_images[i].image == img) {
            if (size > 0) t_pending_images[i].size = size;
            if (memBits > 0) t_pending_images[i].memoryTypeBits = memBits;
            return;
        }
    }

    if (t_pending_count < MAX_PENDING) {
        t_pending_images[t_pending_count].image = img;
        t_pending_images[t_pending_count].size = size;
        t_pending_images[t_pending_count].memoryTypeBits = memBits;
        t_pending_count++;
    }
}

static VkImage pop_matching_image(VkDeviceSize size, uint32_t memoryTypeIndex) {
    for (int i = t_pending_count - 1; i >= 0; i--) {
        if (t_pending_images[i].size == size) {
            VkImage img = t_pending_images[i].image;
            for (int j = i; j < t_pending_count - 1; j++) {
                t_pending_images[j] = t_pending_images[j + 1];
            }
            t_pending_count--;
            return img;
        }
    }
    if (t_pending_count > 0) {
        t_pending_count--;
        return t_pending_images[t_pending_count].image;
    }
    return VK_NULL_HANDLE;
}

static void remove_pending_image(VkImage img) {
    for (int i = 0; i < t_pending_count; i++) {
        if (t_pending_images[i].image == img) {
            for (int j = i; j < t_pending_count - 1; j++) {
                t_pending_images[j] = t_pending_images[j + 1];
            }
            t_pending_count--;
            break;
        }
    }
}

VKAPI_ATTR VkResult VKAPI_CALL Shim_EnumerateInstanceLayerProperties(
    uint32_t* pPropertyCount,
    VkLayerProperties* pProperties)
{
    if (pPropertyCount) *pPropertyCount = 1;
    if (pProperties) {
        memset(pProperties, 0, sizeof(VkLayerProperties));
        strncpy(pProperties->layerName, "VK_LAYER_GFXSTREAM_dedicated_shim", VK_MAX_EXTENSION_NAME_SIZE - 1);
        pProperties->specVersion = VK_MAKE_VERSION(1, 4, 0);
        pProperties->implementationVersion = 1;
        strncpy(pProperties->description, "Automatic VkMemoryDedicatedAllocateInfo injector", VK_MAX_DESCRIPTION_SIZE - 1);
    }
    return VK_SUCCESS;
}

VKAPI_ATTR VkResult VKAPI_CALL Shim_EnumerateInstanceExtensionProperties(
    const char* pLayerName,
    uint32_t* pPropertyCount,
    VkExtensionProperties* pProperties)
{
    if (pPropertyCount) *pPropertyCount = 0;
    return VK_SUCCESS;
}

VKAPI_ATTR VkResult VKAPI_CALL Shim_EnumerateDeviceExtensionProperties(
    VkPhysicalDevice physicalDevice,
    const char* pLayerName,
    uint32_t* pPropertyCount,
    VkExtensionProperties* pProperties)
{
    if (pLayerName && !strcmp(pLayerName, "VK_LAYER_GFXSTREAM_dedicated_shim")) {
        if (pPropertyCount) *pPropertyCount = 0;
        return VK_SUCCESS;
    }

    pthread_mutex_lock(&g_instance_map_lock);
    PFN_vkEnumerateDeviceExtensionProperties pfn = NULL;
    if (g_instance_map[0].instance != VK_NULL_HANDLE) {
        pfn = g_instance_map[0].pfnEnumerateDeviceExtensionProperties;
    }
    pthread_mutex_unlock(&g_instance_map_lock);

    if (!pfn) {
        pfn = (PFN_vkEnumerateDeviceExtensionProperties)dlsym(RTLD_NEXT, "vkEnumerateDeviceExtensionProperties");
    }

    if (pfn) {
        return pfn(physicalDevice, pLayerName, pPropertyCount, pProperties);
    }

    return VK_ERROR_INITIALIZATION_FAILED;
}

VKAPI_ATTR VkResult VKAPI_CALL Shim_CreateImage(
    VkDevice device,
    const VkImageCreateInfo* pCreateInfo,
    const VkAllocationCallbacks* pAllocator,
    VkImage* pImage)
{
    DeviceDispatchTable* dt = get_device_dispatch(device);
    if (!dt || !dt->CreateImage) return VK_ERROR_INITIALIZATION_FAILED;

    VkResult res = dt->CreateImage(device, pCreateInfo, pAllocator, pImage);
    if (res == VK_SUCCESS && pImage && *pImage != VK_NULL_HANDLE) {
        push_pending_image_reqs(*pImage, 0, 0);
    }
    return res;
}

VKAPI_ATTR void VKAPI_CALL Shim_GetImageMemoryRequirements(
    VkDevice device,
    VkImage image,
    VkMemoryRequirements* pMemoryRequirements)
{
    DeviceDispatchTable* dt = get_device_dispatch(device);
    if (dt && dt->GetImageMemoryRequirements) {
        dt->GetImageMemoryRequirements(device, image, pMemoryRequirements);
    }
    if (image != VK_NULL_HANDLE && pMemoryRequirements) {
        push_pending_image_reqs(image, pMemoryRequirements->size, pMemoryRequirements->memoryTypeBits);
    }
}

VKAPI_ATTR void VKAPI_CALL Shim_GetImageMemoryRequirements2(
    VkDevice device,
    const VkImageMemoryRequirementsInfo2* pInfo,
    VkMemoryRequirements2* pMemoryRequirements)
{
    DeviceDispatchTable* dt = get_device_dispatch(device);
    if (dt && dt->GetImageMemoryRequirements2) {
        dt->GetImageMemoryRequirements2(device, pInfo, pMemoryRequirements);
    }
    if (pInfo && pInfo->image != VK_NULL_HANDLE && pMemoryRequirements) {
        push_pending_image_reqs(pInfo->image, pMemoryRequirements->memoryRequirements.size, pMemoryRequirements->memoryRequirements.memoryTypeBits);
    }
}

VKAPI_ATTR void VKAPI_CALL Shim_DestroyImage(
    VkDevice device,
    VkImage image,
    const VkAllocationCallbacks* pAllocator)
{
    DeviceDispatchTable* dt = get_device_dispatch(device);
    remove_pending_image(image);
    if (dt && dt->DestroyImage) {
        dt->DestroyImage(device, image, pAllocator);
    }
}

VKAPI_ATTR VkResult VKAPI_CALL Shim_AllocateMemory(
    VkDevice device,
    const VkMemoryAllocateInfo* pAllocateInfo,
    const VkAllocationCallbacks* pAllocator,
    VkDeviceMemory* pMemory)
{
    DeviceDispatchTable* dt = get_device_dispatch(device);
    if (!dt || !dt->AllocateMemory) return VK_ERROR_INITIALIZATION_FAILED;

    int has_export = 0;
    int has_dedicated = 0;
    const VkBaseInStructure* curr = (const VkBaseInStructure*)pAllocateInfo->pNext;
    while (curr) {
        if (curr->sType == VK_STRUCTURE_TYPE_EXPORT_MEMORY_ALLOCATE_INFO) {
            has_export = 1;
        } else if (curr->sType == VK_STRUCTURE_TYPE_MEMORY_DEDICATED_ALLOCATE_INFO) {
            has_dedicated = 1;
        }
        curr = curr->pNext;
    }

    VkImage pending_img = pop_matching_image(pAllocateInfo->allocationSize, pAllocateInfo->memoryTypeIndex);

    if (!has_dedicated && pending_img != VK_NULL_HANDLE) {
        VkExportMemoryAllocateInfo exportInfo = {
            .sType = VK_STRUCTURE_TYPE_EXPORT_MEMORY_ALLOCATE_INFO,
            .pNext = pAllocateInfo->pNext,
            .handleTypes = VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT
        };

        VkMemoryDedicatedAllocateInfo dedicatedInfo = {
            .sType = VK_STRUCTURE_TYPE_MEMORY_DEDICATED_ALLOCATE_INFO,
            .pNext = has_export ? pAllocateInfo->pNext : &exportInfo,
            .image = pending_img,
            .buffer = VK_NULL_HANDLE
        };

        VkMemoryAllocateInfo modAllocateInfo = *pAllocateInfo;
        modAllocateInfo.pNext = &dedicatedInfo;

        return dt->AllocateMemory(device, &modAllocateInfo, pAllocator, pMemory);
    }

    return dt->AllocateMemory(device, pAllocateInfo, pAllocator, pMemory);
}

VKAPI_ATTR VkResult VKAPI_CALL Shim_CreateDevice(
    VkPhysicalDevice physicalDevice,
    const VkDeviceCreateInfo* pCreateInfo,
    const VkAllocationCallbacks* pAllocator,
    VkDevice* pDevice)
{
    VkLayerDeviceCreateInfo* chain_info = (VkLayerDeviceCreateInfo*)pCreateInfo->pNext;
    while (chain_info && (chain_info->sType != VK_STRUCTURE_TYPE_LOADER_DEVICE_CREATE_INFO ||
                          chain_info->function != VK_LAYER_LINK_INFO)) {
        chain_info = (VkLayerDeviceCreateInfo*)chain_info->pNext;
    }

    if (!chain_info || !chain_info->u.pLayerInfo) {
        return VK_ERROR_INITIALIZATION_FAILED;
    }

    PFN_vkGetInstanceProcAddr gipa = chain_info->u.pLayerInfo->pfnNextGetInstanceProcAddr;
    PFN_vkGetDeviceProcAddr gdpa = chain_info->u.pLayerInfo->pfnNextGetDeviceProcAddr;

    PFN_vkCreateDevice createDevice = (PFN_vkCreateDevice)gipa(VK_NULL_HANDLE, "vkCreateDevice");
    if (!createDevice) return VK_ERROR_INITIALIZATION_FAILED;

    chain_info->u.pLayerInfo = chain_info->u.pLayerInfo->pNext;

    VkResult res = createDevice(physicalDevice, pCreateInfo, pAllocator, pDevice);
    if (res != VK_SUCCESS) return res;

    if (gdpa && pDevice && *pDevice != VK_NULL_HANDLE) {
        DeviceDispatchTable dt;
        dt.GetDeviceProcAddr = gdpa;
        dt.AllocateMemory = (PFN_vkAllocateMemory)gdpa(*pDevice, "vkAllocateMemory");
        dt.CreateImage = (PFN_vkCreateImage)gdpa(*pDevice, "vkCreateImage");
        dt.GetImageMemoryRequirements = (PFN_vkGetImageMemoryRequirements)gdpa(*pDevice, "vkGetImageMemoryRequirements");
        dt.GetImageMemoryRequirements2 = (PFN_vkGetImageMemoryRequirements2)gdpa(*pDevice, "vkGetImageMemoryRequirements2");
        if (!dt.GetImageMemoryRequirements2) {
            dt.GetImageMemoryRequirements2 = (PFN_vkGetImageMemoryRequirements2)gdpa(*pDevice, "vkGetImageMemoryRequirements2KHR");
        }
        dt.DestroyImage = (PFN_vkDestroyImage)gdpa(*pDevice, "vkDestroyImage");
        set_device_dispatch(*pDevice, dt);
    }
    return VK_SUCCESS;
}

VKAPI_ATTR VkResult VKAPI_CALL Shim_CreateInstance(
    const VkInstanceCreateInfo* pCreateInfo,
    const VkAllocationCallbacks* pAllocator,
    VkInstance* pInstance)
{
    VkLayerInstanceCreateInfo* chain_info = (VkLayerInstanceCreateInfo*)pCreateInfo->pNext;
    while (chain_info && (chain_info->sType != VK_STRUCTURE_TYPE_LOADER_INSTANCE_CREATE_INFO ||
                          chain_info->function != VK_LAYER_LINK_INFO)) {
        chain_info = (VkLayerInstanceCreateInfo*)chain_info->pNext;
    }

    if (!chain_info || !chain_info->u.pLayerInfo) {
        return VK_ERROR_INITIALIZATION_FAILED;
    }

    PFN_vkGetInstanceProcAddr gipa = chain_info->u.pLayerInfo->pfnNextGetInstanceProcAddr;
    PFN_vkCreateInstance createInstance = (PFN_vkCreateInstance)gipa(VK_NULL_HANDLE, "vkCreateInstance");
    if (!createInstance) return VK_ERROR_INITIALIZATION_FAILED;

    chain_info->u.pLayerInfo = chain_info->u.pLayerInfo->pNext;

    VkResult res = createInstance(pCreateInfo, pAllocator, pInstance);
    if (res == VK_SUCCESS && pInstance && *pInstance != VK_NULL_HANDLE) {
        set_instance_info(*pInstance, gipa);
    }
    return res;
}

VKAPI_ATTR PFN_vkVoidFunction VKAPI_CALL Shim_GetDeviceProcAddr(VkDevice device, const char* pName) {
    if (!strcmp(pName, "vkAllocateMemory")) return (PFN_vkVoidFunction)Shim_AllocateMemory;
    if (!strcmp(pName, "vkCreateImage")) return (PFN_vkVoidFunction)Shim_CreateImage;
    if (!strcmp(pName, "vkGetImageMemoryRequirements")) return (PFN_vkVoidFunction)Shim_GetImageMemoryRequirements;
    if (!strcmp(pName, "vkGetImageMemoryRequirements2")) return (PFN_vkVoidFunction)Shim_GetImageMemoryRequirements2;
    if (!strcmp(pName, "vkGetImageMemoryRequirements2KHR")) return (PFN_vkVoidFunction)Shim_GetImageMemoryRequirements2;
    if (!strcmp(pName, "vkDestroyImage")) return (PFN_vkVoidFunction)Shim_DestroyImage;

    DeviceDispatchTable* dt = get_device_dispatch(device);
    if (dt && dt->GetDeviceProcAddr) {
        return dt->GetDeviceProcAddr(device, pName);
    }
    return NULL;
}

VKAPI_ATTR PFN_vkVoidFunction VKAPI_CALL Shim_GetInstanceProcAddr(VkInstance instance, const char* pName) {
    if (!strcmp(pName, "vkEnumerateInstanceLayerProperties")) return (PFN_vkVoidFunction)Shim_EnumerateInstanceLayerProperties;
    if (!strcmp(pName, "vkEnumerateInstanceExtensionProperties")) return (PFN_vkVoidFunction)Shim_EnumerateInstanceExtensionProperties;
    if (!strcmp(pName, "vkEnumerateDeviceExtensionProperties")) return (PFN_vkVoidFunction)Shim_EnumerateDeviceExtensionProperties;
    if (!strcmp(pName, "vkCreateInstance")) return (PFN_vkVoidFunction)Shim_CreateInstance;
    if (!strcmp(pName, "vkCreateDevice")) return (PFN_vkVoidFunction)Shim_CreateDevice;
    if (!strcmp(pName, "vkGetInstanceProcAddr")) return (PFN_vkVoidFunction)Shim_GetInstanceProcAddr;
    if (!strcmp(pName, "vkGetDeviceProcAddr")) return (PFN_vkVoidFunction)Shim_GetDeviceProcAddr;
    if (!strcmp(pName, "vkAllocateMemory")) return (PFN_vkVoidFunction)Shim_AllocateMemory;
    if (!strcmp(pName, "vkCreateImage")) return (PFN_vkVoidFunction)Shim_CreateImage;
    if (!strcmp(pName, "vkGetImageMemoryRequirements")) return (PFN_vkVoidFunction)Shim_GetImageMemoryRequirements;
    if (!strcmp(pName, "vkGetImageMemoryRequirements2")) return (PFN_vkVoidFunction)Shim_GetImageMemoryRequirements2;
    if (!strcmp(pName, "vkGetImageMemoryRequirements2KHR")) return (PFN_vkVoidFunction)Shim_GetImageMemoryRequirements2;
    if (!strcmp(pName, "vkDestroyImage")) return (PFN_vkVoidFunction)Shim_DestroyImage;

    if (instance != VK_NULL_HANDLE) {
        PFN_vkGetInstanceProcAddr gipa = get_instance_gipa(instance);
        if (gipa) {
            return gipa(instance, pName);
        }
    }

    return NULL;
}

VKAPI_ATTR VkResult VKAPI_CALL vkNegotiateLoaderLayerInterfaceVersion(VkNegotiateLayerInterface* pVersionStruct) {
    if (!pVersionStruct || pVersionStruct->sType != LAYER_NEGOTIATE_INTERFACE_STRUCT) {
        return VK_ERROR_INITIALIZATION_FAILED;
    }
    if (pVersionStruct->loaderLayerInterfaceVersion < 2) {
        return VK_ERROR_INITIALIZATION_FAILED;
    }
    pVersionStruct->loaderLayerInterfaceVersion = 2;
    pVersionStruct->pfnGetInstanceProcAddr = Shim_GetInstanceProcAddr;
    pVersionStruct->pfnGetDeviceProcAddr = Shim_GetDeviceProcAddr;
    pVersionStruct->pfnGetPhysicalDeviceProcAddr = NULL;
    return VK_SUCCESS;
}
```

---

## 2. Vulkan Implicit Layer JSON Manifest

Create the system implicit layer manifest at `/usr/share/vulkan/implicit_layer.d/VkLayer_gfxstream_dedicated_shim.json`:

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

---

## 3. Step-by-Step Compilation & Installation Instructions

Follow these exact steps inside the AVF Debian guest environment:

### Step 1: Install Build Dependencies
```bash
sudo apt-get update
sudo apt-get install -y build-essential libvulkan-dev
```

### Step 2: Compile the Shared Library
```bash
gcc -shared -fPIC -O2 /tmp/libvulkan_dedicated_shim.c \
    -o /tmp/libvulkan_dedicated_shim.so \
    -lvulkan -lpthread -ldl
```

### Step 3: Install Shared Object & Manifest
```bash
sudo cp /tmp/libvulkan_dedicated_shim.so /usr/lib/aarch64-linux-gnu/libvulkan_dedicated_shim.so
sudo cp /tmp/VkLayer_gfxstream_dedicated_shim.json /usr/share/vulkan/implicit_layer.d/VkLayer_gfxstream_dedicated_shim.json
```

### Step 4: System Environment & Script Configuration

Ensure that legacy Zink and GLES2 overrides are removed from system startup scripts so native Gfxstream acceleration and desktop portals function properly.

#### Update `/usr/local/bin/enable_gfxstream`:
```bash
sudo cat << 'EOF' | sudo tee /usr/local/bin/enable_gfxstream
#!/bin/bash
cat << EOF2 > /home/droid/labwc.env
VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/gfxstream_vk_icd.json
WLR_DRM_NO_ATOMIC=1
WLR_NO_HARDWARE_CURSORS=1
WLR_RENDERER_ALLOW_SOFTWARE=1
XDG_CURRENT_DESKTOP=LXQt
XDG_SESSION_TYPE=wayland
XDG_SESSION_DESKTOP=LXQt
EOF2
cp /home/droid/labwc.env /home/droid/weston.env
sudo systemd-run --quiet --collect -E XDG_SESSION_TYPE=wayland --uid=1000 -p PAMName=login -p TTYPath=/dev/tty1 sleep 1d
systemctl --user set-environment VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/gfxstream_vk_icd.json WLR_DRM_NO_ATOMIC=1 WLR_NO_HARDWARE_CURSORS=1 WLR_RENDERER_ALLOW_SOFTWARE=1 XDG_CURRENT_DESKTOP=LXQt XDG_SESSION_TYPE=wayland XDG_SESSION_DESKTOP=LXQt
(sleep 3s; systemctl --user start labwc)& disown
export DISPLAY=:0
unset MESA_LOADER_DRIVER_OVERRIDE
unset LIBGL_ALWAYS_SOFTWARE
export VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/gfxstream_vk_icd.json
export XDG_CURRENT_DESKTOP=LXQt
export XDG_SESSION_TYPE=wayland
export WLR_DRM_NO_ATOMIC=1
export WLR_NO_HARDWARE_CURSORS=1
EOF
sudo chmod +x /usr/local/bin/enable_gfxstream
```

#### Update `~/.config/labwc/rc.xml` to enable Xwayland:
```xml
<?xml version="1.0"?>
<labwc_config>
  <core>
    <xwayland>enabled</xwayland>
  </core>
  <libinput>
    <device category="default">
      <pointerSpeed>-0.5</pointerSpeed>
      <accelProfile>adaptive</accelProfile>
    </device>
  </libinput>
</labwc_config>
```

---

## 4. Verification & Testing Procedure

### 1. Verify Layer Registration
```bash
vulkaninfo --summary
```
*Expected Output*: Output should complete cleanly without warnings and list `VK_LAYER_GFXSTREAM_dedicated_shim` under implicit layers.

### 2. Verify Multi-Threaded Stress Test
Compile and run the multi-threaded stress test harness:
```bash
gcc /tmp/stress_vk_shim.c -o /tmp/stress_vk_shim -lvulkan -lpthread
/tmp/stress_vk_shim
```
*Expected Output*:
```text
=== STARTING MULTI-THREADED BATCH STRESS TEST ON VULKAN SHIM ===
=== STRESS TEST COMPLETED SUCCESSFULLY ===
```

### 3. Verify Desktop Session & Log Output
Restart the `labwc` user service and monitor Mesa system logs:
```bash
systemctl --user restart labwc
sudo journalctl -b -g "MESA" --no-pager | tail -n 20
```
*Expected Result*: Zero `drmPrimeHandleToFD failed` or `on_vkGetMemoryFdKHR` errors logged to `journalctl`. Full LXQt desktop (`lxqt-panel`, `pcmanfm-qt --desktop`, `lxqt-runner`) running active with hardware Vulkan forwarding.
