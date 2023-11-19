<template>
    <n-tabs class="animated-wrapper" animated :value="isConnected ? '1' : '0'">
        <n-tab-pane name="0">
            <template v-if="isBrowserSupported">
                <div class="text-xl">
                    未连接设备
                </div>
                <div class="text-md opacity-80 mb-4">
                    使用线缆将Fastboot模式下的设备连接到该设备, 点击连接.
                </div>
                <div v-if="isUserCancelSelect" class=" mb-4">
                    <div class="text-lg">找不到想要连接的设备?</div>
                    <div class="opacity-80">确保线缆连接稳固.<br> Windows下需要安装驱动才能连接, 请确保安装了兼容的驱动.</div>

                </div>
                <n-button @click="connectDevice" :loading="isDeviceInfoLoading">连接设备</n-button>
            </template>
            <template v-else>
                <div class="text-xl">
                    不受支持的浏览器
                </div>
                <div class="text-md opacity-80 mb-4">
                    请尝试使用最新版的Chrome或者Edge.<br>
                    iOS设备不支持通过浏览器访问USB.<br>
                    不支持Firefox.
                </div>
            </template>
        </n-tab-pane>
        <n-tab-pane name="1">
            <template v-if="!isDeviceReadError">
                <div class="text-xl">
                    {{ DeviceInfo.product }} <span class="text-sm opacity-80">({{ DeviceInfo.serialno }})</span>
                </div>
                <div class="text-md opacity-80 mb-4">
                    Bootloader {{ DeviceInfo.unlocked === "yes" ? "已解锁" : "已锁定" }}
                </div>
            </template>
            <template v-else>
                <div class="text-xl">
                    未知设备
                </div>
                <div class="text-md opacity-80 mb-4">
                    读取设备信息时出现问题, 部分指令可能不可用
                </div>
            </template>
            <n-space vertical>
                <n-card title="从镜像启动">
                    <n-space justify="center">
                        <n-upload ref="BootImageFileDropRef" @change="handleBootImageFileDrop" accept=".img,*"
                            :show-file-list="false">
                            <n-upload-dragger>
                                <div class="flex items-center">
                                    <n-icon :size="48" class="opacity-50 mr-4">
                                        <svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink"
                                            viewBox="0 0 24 24">
                                            <path
                                                d="M18 15v3H6v-3H4v3c0 1.1.9 2 2 2h12c1.1 0 2-.9 2-2v-3h-2zM7 9l1.41 1.41L11 7.83V16h2V7.83l2.59 2.58L17 9l-5-5l-5 5z"
                                                fill="currentColor"></path>
                                        </svg>
                                    </n-icon>
                                    <n-text class="text-lg">
                                        点击或者拖动镜像文件到该区域
                                    </n-text>
                                </div>
                            </n-upload-dragger>
                        </n-upload>
                    </n-space>
                </n-card>
                <n-card title="刷入单个镜像">
                    <n-space justify="center" align="center">
                        <n-upload ref="FlashImageFileDropRef" v-model:file-list="FlashImageFile" accept=".img,*"
                            :show-file-list="false">
                            <n-upload-dragger>
                                <div class="flex items-center">
                                    <n-icon :size="48" class="opacity-50 mr-4">
                                        <svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink"
                                            viewBox="0 0 24 24">
                                            <path
                                                d="M18 15v3H6v-3H4v3c0 1.1.9 2 2 2h12c1.1 0 2-.9 2-2v-3h-2zM7 9l1.41 1.41L11 7.83V16h2V7.83l2.59 2.58L17 9l-5-5l-5 5z"
                                                fill="currentColor"></path>
                                        </svg>
                                    </n-icon>
                                    <n-text class="text-lg">
                                        {{ FlashImageFile[0]?.name || "点击或者拖动镜像文件到该区域" }}
                                    </n-text>
                                </div>
                            </n-upload-dragger>
                        </n-upload>
                        <n-space vertical justify="center" class="w-160px h-full">
                            <n-select :options="FlashTargetOption" v-model:value="FlashSelectedTarget"
                                placeholder="刷写到..." />
                            <n-input v-model:value="FlashCustomTarget" v-if="FlashSelectedTarget === 'other'"
                                placeholder="刷写到..." />
                            <n-button class="w-full overflow-hidden" type="primary"
                                :disabled="!FlashImageFile[0]?.name || !ComputedFlashTarget"
                                @click="requestFlashImageToTarget">
                                刷写 {{ ComputedFlashTarget }}
                            </n-button>
                        </n-space>
                    </n-space>
                </n-card>
                <n-card title="设备操作">
                    <n-space>
                        <n-button @click="rebootTo()">重启到系统</n-button>
                        <n-button @click="rebootTo('recovery')">重启到 Recovery</n-button>
                        <n-button @click="rebootTo('bootloader')">重启到 Bootloader</n-button>
                        <n-button @click="disconnectDevice">断开连接</n-button>
                    </n-space>
                </n-card>
                <n-card title="设备信息">
                    <n-table :single-line="false">
                        <thead>
                            <tr>
                                <th>键</th>
                                <th>值</th>
                            </tr>
                        </thead>
                        <tbody>
                            <tr v-for="line in Object.keys(DeviceInfo)" :key="line">
                                <td>{{ line }}</td>
                                <td>{{ DeviceInfo[line] }}</td>
                            </tr>
                        </tbody>
                    </n-table>
                </n-card>
            </n-space>
        </n-tab-pane>
    </n-tabs>
</template>
<script lang="ts" setup>
import { FastbootDevice, FastbootError } from "android-fastboot";
import { UploadFileInfo, UploadInst } from "naive-ui";
import { useEventListener } from "@vueuse/core";

let device = new FastbootDevice();
const isConnected = ref(device.isConnected);
const DeviceInfo = reactive({} as Record<string, string>);
const dialog = useDialog();
const isBrowserSupported = "usb" in navigator;

const isDeviceInfoLoading = ref(false);
const isUserCancelSelect = ref(false);

useEventListener(navigator.usb, "disconnect", (e: USBConnectionEvent) => {
    if (e.device === device.device) {
        isConnected.value = false;
    }
});

const isDeviceReadError = ref(false);
async function connectDevice() {
    try {
        await device.connect();
    } catch (e) {
        if (e instanceof Error && e.message.includes("No device selected")) {
            isUserCancelSelect.value = true;
        } else {
            dialog.error({
                "title": "连接失败",
                "type": "error",
                "content": "请检查设备连接, 并重试."
            });
        }
        return;
    }
    isDeviceInfoLoading.value = true;
    try {

        (await device.getVariable("all"))
            ?.split("\n")
            .map(val => val.split(":"))
            .forEach(kv => {
                DeviceInfo[kv[0]] = kv[1].trim();
            });

        isDeviceReadError.value = false;
    } catch (e) {
        isDeviceReadError.value = true;
    }
    isDeviceInfoLoading.value = false;
    isUserCancelSelect.value = false;
    isConnected.value = device.isConnected;
}

async function disconnectDevice() {
    await device.device?.close();
    isConnected.value = false;
}

async function rebootTo(target?: string) {
    await device.reboot(target);
    await disconnectDevice();
}

const BootImageFileDropRef = ref<UploadInst>();
async function handleBootImageFileDrop(e: { file: UploadFileInfo }) {
    const filename = e.file.name;
    const dialogInst = dialog.warning({
        title: "从镜像文件启动",
        content: `确定要在设备 "${DeviceInfo.product}" 上启动镜像 "${filename}" 吗?`,
        onPositiveClick() {
            dialogInst.loading = true;
            dialogInst.maskClosable = false;
            return new Promise(async (resolve) => {
                try {
                    await device.bootBlob(e.file.file!, (e) => {
                        dialogInst.positiveText = (e * 100).toFixed(1).toString() + "%";
                        if (e === 1) {
                            dialogInst.positiveText = "设备正在重启..."
                            const clean = useEventListener(navigator.usb, "disconnect", (e: USBConnectionEvent) => {
                                if (e.device === device.device) {
                                    resolve(true);
                                    clean();
                                    dialog.success({
                                        "title": "从镜像文件启动",
                                        "content": "成功启动到镜像文件, 设备正在重启进入镜像.",
                                        positiveText: "确定"
                                    });
                                }
                            });
                        }
                    });
                } catch (e) {
                    resolve(true);
                    if (e instanceof FastbootError) {
                        dialog.error({
                            title: "从镜像文件启动失败",
                            content: "无法从镜像文件启动, Bootloader返回错误: " + e.bootloaderMessage,
                            positiveText: "确定"
                        });
                    } else if (e instanceof Error) {
                        dialog.error({
                            title: "从镜像文件启动失败",
                            content: "无法从镜像文件启动, 出现错误" + e.message,
                            positiveText: "确定"
                        });
                    } else {
                        throw e;
                    }
                }
            });
        },
        onClose() {
            if (dialogInst.loading) return false;
        },
        onNegativeClick() {
            if (dialogInst.loading) return false;
        },
        positiveText: '启动',
        negativeText: '取消',
    });
}


const FlashImageFileDropRef = ref<UploadInst>();
const FlashImageFile = ref<UploadFileInfo[]>([]);
watch(FlashImageFile, () => {
    if (FlashImageFile.value[1]) {
        FlashImageFile.value.splice(0, 1);
    }
});
const FlashSelectedTarget = ref<string>();
const FlashCustomTarget = ref("");
const FlashTargetOption = [
    {
        value: "recovery",
        label: "recovery"
    }, {
        value: "recovery_a",
        label: "recovery_a"
    }, {
        value: "recovery_b",
        label: "recovery_b"
    },
    {
        label: "boot",
        value: "boot"
    }, {
        label: "boot_a",
        value: "boot_a"
    }, {
        label: "boot_b",
        value: "boot_b"
    }, {
        label: "其他",
        value: "other"
    }];

const ComputedFlashTarget = computed(() => {
    if (FlashSelectedTarget.value === "other") return FlashCustomTarget.value;
    return FlashSelectedTarget.value;
});

async function requestFlashImageToTarget() {
    const file = FlashImageFile.value[0];
    const target = ComputedFlashTarget.value;
    if (!file?.file || !target) return;
    const dialogInst = dialog.warning({
        title: "刷写单个镜像",
        content: `确定要在设备 "${DeviceInfo.product}" 上将镜像 "${file.name}" 刷写到分区 ${target} 吗?`,
        onPositiveClick() {
            dialogInst.loading = true;
            dialogInst.maskClosable = false;
            return new Promise(async (resolve) => {
                if (!file?.file) return;
                try {
                    await device.flashBlob(target, file.file, (e) => {
                        dialogInst.positiveText = (e * 100).toFixed(1).toString() + "%";
                        if (e === 1) {
                            resolve(true);
                        }
                    });
                    dialog.success({
                        "title": "刷写单个镜像",
                        "content": "成功刷写镜像文件.",
                        positiveText: "确定"
                    });
                } catch (e) {
                    resolve(true);
                    if (e instanceof FastbootError) {
                        dialog.error({
                            title: "刷写镜像文件失败",
                            content: "无法刷写镜像, Bootloader返回错误: " + e.bootloaderMessage,
                            positiveText: "确定"
                        });
                    } else if (e instanceof Error) {
                        dialog.error({
                            title: "刷写镜像文件失败",
                            content: "无法刷写镜像, 出现错误" + e.message,
                            positiveText: "确定"
                        });
                    } else {
                        throw e;
                    }
                }
            });
        },
        onClose() {
            if (dialogInst.loading) return false; 
        },
        onNegativeClick() {
           if (dialogInst.loading) return false;
        },
        positiveText: '刷写',
        negativeText: '取消',
    });
}
</script>

<style scoped>.animated-wrapper :deep(.n-tabs-nav--top) {
    display: none;
}</style>