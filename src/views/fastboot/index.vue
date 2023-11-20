<template>
    <n-tabs class="animated-wrapper" animated :value="ComputedCurrentTab">
        <n-tab-pane name="waiting">
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
        <n-tab-pane name="operate">
            <template v-if="!isDeviceReadError">
                <div class="text-xl">
                    {{ DeviceInfo.product }} <span class="text-sm opacity-80">({{ DeviceInfo.serialno }})</span>
                </div>
                <div class="text-md opacity-80 mb-4">
                    Bootloader {{ DeviceInfo.unlocked === "yes" ? "已解锁" : "已锁定" }}, {{ DeviceInfo['is-userspace'] === "yes"
                        ? "Userspace Fastboot" : "Bootloader Fastboot" }}
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
                        <n-upload :disabled="isDeviceBusy" ref="BootImageFileDropRef" @change="handleBootImageFileDrop"
                            accept=".img,*" :show-file-list="false">
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
                        <n-upload :disabled="isDeviceBusy" class="max-w-420px" ref="FlashImageFileDropRef"
                            v-model:file-list="FlashImageFile" accept=".img,*" :show-file-list="false">
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
                        <n-space vertical justify="center" class="w-240px h-full">
                            <n-select :disabled="isDeviceBusy" :options="FlashTargetOption"
                                v-model:value="FlashSelectedTarget" placeholder="刷写到..." />
                            <n-input :disabled="isDeviceBusy" v-model:value="FlashCustomTarget"
                                v-if="FlashSelectedTarget === 'other'" placeholder="刷写到..." />
                            <n-button class="w-full overflow-hidden" type="primary"
                                :disabled="!FlashImageFile[0]?.name || !ComputedFlashTarget || isDeviceBusy"
                                @click="requestFlashImageToTarget">
                                刷写 {{ ComputedFlashTarget }}
                            </n-button>
                        </n-space>
                    </n-space>
                </n-card>
                <n-card title="刷入线刷包">
                    <n-space justify="center" align="center">
                        <n-upload :disabled="isDeviceBusy" :multiple="false" class="max-w-420px"
                            ref="FlashDirectoryFileDropRef" v-model:file-list="FlashDirectoryFileList"
                            :show-file-list="false" directory>
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
                                        {{ ComputedFlashDirectoryName || "点击或者拖动解压后的线刷包文件夹到该区域" }}
                                    </n-text>
                                </div>
                            </n-upload-dragger>
                        </n-upload>
                        <n-space vertical justify="center" class="w-240px h-full">
                            <n-select :options="FlashDirectoryScriptOptions" v-model:value="FlashDirectorySelectedScript"
                                placeholder="选择刷写脚本..." :disabled="isDeviceBusy">
                                <template #empty>
                                    <n-space vertical class="text-center">
                                        <div class="text-lg">没有可用的刷写脚本</div>
                                        <div class="opacity-70">请重新选择文件夹, 确保被选择的文件夹中有.bat脚本.</div>
                                    </n-space>
                                </template>
                            </n-select>
                            <n-button class="w-full overflow-hidden" type="primary"
                                :disabled="!FlashDirectorySelectedScript || isDeviceBusy"
                                @click="requestFlashDirectoryToTarget">
                                刷写
                            </n-button>
                        </n-space>
                    </n-space>
                </n-card>
                <n-card title="设备操作">
                    <n-space vertical>
                        <n-text>重启</n-text>
                        <n-space>
                            <n-button :disabled="isDeviceBusy" @click="rebootTo()">重启到系统</n-button>
                            <n-button :disabled="isDeviceBusy" @click="rebootTo('bootloader')">重启到 Bootloader</n-button>
                        </n-space>
                        <template v-if="DeviceInfo['current-slot']">
                            <n-text>A/B分区 (当前槽: {{ DeviceInfo['current-slot'].toUpperCase() }})</n-text>
                            <n-space>
                                <n-button :disabled="isDeviceBusy" v-if="DeviceInfo['current-slot'].toUpperCase() === 'B'"
                                    @click="switchSlot('a')">切换到槽 A</n-button>
                                <n-button :disabled="isDeviceBusy" v-else @click="switchSlot('b')">切换到槽 B</n-button>
                            </n-space>
                        </template>
                        <n-text>危险操作</n-text>
                        <n-space>
                            <n-button :disabled="isDeviceBusy" type="error" @click="requestWipeDevice">清除数据</n-button>
                            <n-button :disabled="isDeviceBusy" type="error" @click="requestLockBl">锁定Bootloader</n-button>
                        </n-space>
                        <n-text>连接</n-text>
                        <n-button :disabled="isDeviceBusy" @click="disconnectDevice">断开连接</n-button>
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
        <n-tab-pane name="flashDirectory">
            <n-card title="正在刷入...">
                <div class="flex flex-col">
                    <n-scrollbar class="max-h-75vh">
                        <n-timeline>
                            <n-timeline-item v-for="(state, i) in FlashStateList" :key="i" :content="state.description"
                                :type="state.status === 'pending' ? 'default' : (state.status === 'progressing' ? 'info' : state.status)"
                                :line-type="state.status === 'pending' ? 'dashed' : 'default'" />
                        </n-timeline>
                    </n-scrollbar>
                    <n-space class="mt-8 flex-shrink-0" v-if="!isDeviceBusy">
                        <n-button type="primary" @click="FlashStateList = []">返回</n-button>
                        <n-button @click="rebootTo()">重启设备</n-button>
                    </n-space>
                </div>
            </n-card>
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
const isDeviceBusy = ref(false);

const ComputedCurrentTab = computed(() => {
    if (!isConnected.value) {
        return "waiting";
    } else if (!FlashStateList.value?.length) {
        return "operate"
    } else {
        return "flashDirectory"
    }
});

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
            .map(val => {
                const split = val.lastIndexOf(":");
                return [
                    val.substring(0, split),
                    val.substring(split + 1)
                ]
            })
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
        content() {
            return [
                "确定要在设备 ",
                h("b", DeviceInfo.product),
                " 启动镜像文件 ",
                h("b", filename),
                " 吗?",
                h("br"),
                h("br"),
                "刷入错误的镜像可能",
                h("b", "导致设备损坏"),
                ", 请仔细确认",
                h("b", "设备名称"),
                "以及",
                "将要刷入的",
                h("b", "镜像文件名"),
                "."
            ]
        },
        onPositiveClick() {
            dialogInst.loading = true;
            dialogInst.maskClosable = false;
            return new Promise(async (resolve) => {
                try {
                    isDeviceBusy.value = true;
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
                } finally {
                    isDeviceBusy.value = false;
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
        title: "刷入单个镜像",
        content() {
            return [
                "确定要在设备 ",
                h("b", DeviceInfo.product),
                " 上将镜像 ",
                h("b", file.name),
                " 刷入到分区 ",
                h("b", target),
                " 吗?",
                h("br"),
                h("br"),
                "刷入错误的镜像可能",
                h("b", "导致设备损坏"),
                ", 请仔细确认",
                h("b", "设备名称"),
                ", ",
                "将要刷入的",
                h("b", "镜像文件名"),
                "以及",
                h("b", "目标分区"),
                "."
            ]
        },
        onPositiveClick() {
            dialogInst.loading = true;
            dialogInst.maskClosable = false;
            return new Promise(async (resolve) => {
                if (!file?.file) return;
                isDeviceBusy.value = true;
                try {
                    await device.flashBlob(target, file.file, (e) => {
                        dialogInst.positiveText = (e * 100).toFixed(1).toString() + "%";
                    });
                    dialog.success({
                        "title": "刷入单个镜像",
                        "content": "成功刷入镜像文件.",
                        positiveText: "确定"
                    });
                } catch (e) {
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
                } finally {
                    isDeviceBusy.value = false;
                    resolve(true);
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

import ConfirmInput from "./ConfirmInput.vue";
import { SelectMixedOption } from "naive-ui/es/select/src/interface";
async function requestWipeDevice() {
    const dialogInst = dialog.error({
        title: "清空数据",
        content() {
            return [
                "确定要要清除设备 ",
                h("b", DeviceInfo.product),
                " 上的所有数据吗? ",
                h("br"),
                h("br"),
                "务必",
                h("b", "优先使用Recovery清空数据"),
                ", 在Fastboot下清空数据需要重新格式化cache和userdata分区, 否则可能无法正常进入系统.",
                h("br"),
                h("br"),
                "此操作会导致",
                h("b", "设备上所有数据丢失"),
                ", 包括但不限于",
                h("b", "所有应用和数据, 照片, 聊天记录, 联系人, 短信, 模块"),
                ", 请备份所有重要数据, 仔细确认",
                h("b", "设备名称"),
                "确实为需要清空数据的设备.",
                h("br"),
                h("br"),
                `在下方输入框内输入 "wipe" 确认清空该设备上的所有数据.`,
                h("br"),
                h("br"),
                h(ConfirmInput, {
                    onValidated(validated) {
                        dialogInst.positiveButtonProps!.disabled = !validated;
                    },
                    confirmWord: `wipe`
                })
            ]
        },
        async onPositiveClick() {
            dialogInst.loading = true;
            dialogInst.maskClosable = false;
            try {
                isDeviceBusy.value = true;
                await device.runCommand("erase:userdata");
                await device.runCommand("erase:cache");
            } catch (e) {
                if (e instanceof FastbootError) {
                    dialog.error({
                        title: "清空数据失败",
                        content: "无法清空数据, Bootloader返回错误: " + e.bootloaderMessage,
                        positiveText: "确定"
                    });
                } else if (e instanceof Error) {
                    dialog.error({
                        title: "清空数据失败",
                        content: "无法清空数据, 出现错误" + e.message,
                        positiveText: "确定"
                    });
                } else {
                    throw e;
                }

                dialogInst.loading = false;
                dialogInst.maskClosable = true;
                return false;
            } finally {
                isDeviceBusy.value = false;
            }
        },
        onClose() {
            if (dialogInst.loading) return false;
        },
        onNegativeClick() {
            if (dialogInst.loading) return false;
        },
        positiveText: '清空',
        negativeText: '取消',
        positiveButtonProps: {
            disabled: true
        },
        class: "w-500px!"
    });
}

async function requestLockBl() {
    const dialogInst = dialog.error({
        title: "锁定Bootloader",
        content() {
            return [
                "确定要锁定设备 ",
                h("b", DeviceInfo.product),
                " 上的Bootloader吗? ",
                h("br"),
                h("br"),
                "此操作会导致设备上的",
                h("b", "非官方系统, 非官方Recovery, 修改过的内核, 安装了Magisk/KSU的系统无法启动"),
                ", 同时会阻止通过Fastboot刷写任何设备的任何分区, 导致",
                h("b", "无法通过线刷救砖"),
                ".",
                h("br"),
                "在未还原到官方系统的情况下锁定Bootloader会直接导致手机",
                h("b", "无法启动且无简单的补救措施"),
                ".",
                h("br"),
                "同时锁定Bootloader会",
                h("b", "清空设备上所有数据"),
                ", 包括但不限于",
                h("b", "所有应用和数据, 照片, 聊天记录, 联系人, 短信, 模块"),
                ", 并请备份所有重要数据, 仔细确认",
                h("b", "设备名称"),
                "确实为需要锁定Bootloader的设备.",
                h("br"),
                h("br"),
                `在下方输入框内输入 "lock" 确认确定要锁定该设备上的Bootloader.`,
                h("br"),
                h("br"),
                h(ConfirmInput, {
                    onValidated(validated) {
                        dialogInst.positiveButtonProps!.disabled = !validated;
                    },
                    confirmWord: `lock`
                })
            ]
        },
        async onPositiveClick() {
            dialogInst.loading = true;
            dialogInst.maskClosable = false;
            try {
                isDeviceBusy.value = true;
                await device.runCommand("flashing:lock");
            } catch (e) {
                if (e instanceof FastbootError) {
                    dialog.error({
                        title: "锁定Bootloader失败",
                        content: "无法锁定Bootloader, Bootloader返回错误: " + e.bootloaderMessage,
                        positiveText: "确定"
                    });
                } else if (e instanceof Error) {
                    dialog.error({
                        title: "锁定Bootloader失败",
                        content: "无法锁定Bootloader, 出现错误" + e.message,
                        positiveText: "确定"
                    });
                } else {
                    throw e;
                }

                dialogInst.loading = false;
                dialogInst.maskClosable = true;
                return false;
            } finally {
                isDeviceBusy.value = false;
            }
        },
        onClose() {
            if (dialogInst.loading) return false;
        },
        onNegativeClick() {
            if (dialogInst.loading) return false;
        },
        positiveText: '锁定',
        negativeText: '取消',
        positiveButtonProps: {
            disabled: true
        },
        class: "w-500px!"
    });
}

async function switchSlot(slot: string) {
    try {
        isDeviceBusy.value = true;
        await device.runCommand("set_active:" + slot);
        DeviceInfo["current-slot"] = (await device.getVariable("current-slot"))!;
    } catch (e) {
        if (e instanceof FastbootError) {
            dialog.error({
                title: "切换槽失败",
                content: "无法切换槽, Bootloader返回错误: " + e.bootloaderMessage,
                positiveText: "确定"
            });
        } else if (e instanceof Error) {
            dialog.error({
                title: "切换槽失败",
                content: "无法切换槽, 出现错误" + e.message,
                positiveText: "确定"
            });
        } else {
            throw e;
        }
        return false;
    } finally {
        isDeviceBusy.value = false;
    }

}

const FlashDirectoryFileDropRef = ref<UploadInst>();
const FlashDirectoryFileList = ref<UploadFileInfo[]>([]);
const FlashDirectorySelectedScript = ref<string>();
const FlashDirectoryScriptOptions = ref<SelectMixedOption[]>([]);
const ComputedFlashDirectoryName = computed(() => {
    return FlashDirectoryFileList.value[0]?.fullPath?.split("/")[1];
});

watch(FlashDirectoryFileList, (v) => {
    FlashDirectoryScriptOptions.value = v.filter(f => {
        if (f.fullPath) {
            const sp = f.fullPath.split("/");
            return sp.length === 3 && sp[2].endsWith(".bat");
        }
        return false;
    }).map(v => {
        return {
            label: v.name,
            value: v.fullPath!
        }
    });

    if (FlashDirectoryScriptOptions.value.length > 0) {
        FlashDirectorySelectedScript.value = FlashDirectoryScriptOptions.value[0].value as string;
    } else {
        FlashDirectorySelectedScript.value = undefined;
    }
});

watch(FlashDirectoryFileDropRef as never, (v: { $el: HTMLDivElement }) => {
    if (v?.$el) {
        v.$el.querySelector("input")?.addEventListener("change", () => {
            FlashDirectoryFileList.value = [];
        }, { capture: true });
        v.$el.querySelector("div.n-upload-trigger")?.addEventListener("drop", () => {
            FlashDirectoryFileList.value = [];
        }, { capture: true });
    }
});

async function requestFlashDirectoryToTarget() {
    const script = FlashDirectoryFileList.value.find(f => f.fullPath === FlashDirectorySelectedScript.value);
    if (!script?.file) return;
    dialog.warning({
        title: "刷入线刷包",
        content() {
            return [
                "确定要在设备 ",
                h("b", DeviceInfo.product),
                " 上刷入线刷包 ",
                h("b", ComputedFlashDirectoryName.value),
                " 的 ",
                h("b", script.name),
                " 吗?",
                h("br"),
                h("br"),
                "刷入错误的线刷包可能",
                h("b", "导致设备损坏"),
                ", 请仔细确认",
                h("b", "设备名称"),
                "以及",
                "将要刷入的",
                h("b", "线刷包文件夹名"),
                "."
            ]
        },
        onPositiveClick() {
            flashDirectoryToTarget();
        },
        positiveText: '刷写',
        negativeText: '取消',
    });
}

interface FlashState {
    description: string;
    status: "pending" | "progressing" | "success" | "error",
    progress?: string;
}
const FlashStateList = ref<FlashState[]>();

function parseFlashScript(script: string): { type: string, value: any }[] {
    return script.split("\n")
        .map(v => v.replaceAll(" %*", "").replaceAll("%~dp0", ".\\").replaceAll("\\", "/")).map(v => {
            if (v.startsWith("fastboot flash")) {
                const flashInfo = v.split(" ").map(v => v.trim()).filter(v => !!v && !v.startsWith("-"));
                return {
                    type: "flash",
                    value: { target: flashInfo[2], file: flashInfo[3] }
                }
            } else if (v.startsWith("fastboot erase")) {
                const eraseInfo = v.split(" ").map(v => v.trim()).filter(v => !!v && !v.startsWith("-"));
                return {
                    type: "erase",
                    value: eraseInfo[2]
                }
            } else if (v.startsWith("fastboot reboot")) {
                return {
                    type: "reboot",
                    value: ""
                }
            } else {
                return { type: "other", value: v }
            }
        })
}

function findImageFileByName(filename: string): File | undefined {
    let path = filename;
    const dirname = ComputedFlashDirectoryName.value;
    if (!path.startsWith("/")) {
        if (path.startsWith("./")) {
            path = `/${dirname}/` + path.substring(2);
        } else {
            path = `/${dirname}/` + path;
        }
    }
    return FlashDirectoryFileList.value.find(f => f.fullPath === path)?.file || undefined;
}

async function flashDirectoryToTarget() {
    FlashStateList.value = [];
    const state = FlashStateList.value;
    let currentState: FlashState;
    try {
        isDeviceBusy.value = true;
        currentState = reactive({
            description: "正在解析刷写脚本...",
            status: "progressing"
        });
        state.push(currentState);
        const script = FlashDirectoryFileList.value.find(f => f.fullPath === FlashDirectorySelectedScript.value);
        const scriptContent = await script?.file?.text();
        if (!scriptContent) {
            throw Error("刷写脚本解析失败: 无法读取脚本内容");
        }
        if (DeviceInfo.product && !scriptContent.includes(`^product: *${DeviceInfo.product}`)) {
            const resp = await new Promise((resolve) => {
                dialog.warning({
                    title: "刷入线刷包",
                    content: "将要刷入的线刷包可能与当前设备不符 (" + DeviceInfo.product + "), 强行刷入可能导致设备损坏, 仍要刷入吗?",
                    onPositiveClick() { resolve(true); },
                    onNegativeClick() { resolve(false); },
                    closable: false,
                    maskClosable: false,
                    closeOnEsc: false,
                    positiveText: "强行刷入",
                    negativeText: "取消"
                })
            });
            if (!resp) {
                throw Error("刷入失败: 操作被取消");
            }
        }
        const scriptActions = parseFlashScript(scriptContent).filter(v => v.type !== "other");

        currentState.description = `将要刷写 ${scriptActions.filter(v => v.type === 'flash').length} 个分区.`;
        currentState.status = "success";
        state.push({
            "description": "开始执行刷写脚本...",
            "status": "success"
        });

        const stateMap = new Map();
        for (const action of scriptActions) {
            let stepState: FlashState;
            if (action.type === "flash") {
                stepState = reactive({
                    description: `刷写 ${action.value.target} 分区`,
                    status: "pending"
                });
            } else if (action.type === "erase") {
                stepState = reactive({
                    description: `擦除 ${action.value} 分区`,
                    status: "pending"
                });
            } else if (action.type === "reboot") {
                stepState = reactive({
                    description: `重启设备`,
                    status: "pending"
                });
            } else {
                throw Error("执行脚本失败: 未定义操作 " + action.type);
            }
            stateMap.set(action, stepState);
            FlashStateList.value.push(stepState);
        }
        for (const action of scriptActions) {
            const stepState: FlashState = stateMap.get(action)!;
            currentState = stepState;
            currentState.status = "progressing";
            if (action.type === "flash") {
                const partition = action.value.target;
                const file = findImageFileByName(action.value.file);
                if (!file) {
                    throw Error(`刷写 ${partition} 分区失败: 找不到 ${action.value.file}.`)
                }
                try {
                    const stateStr = currentState.description;
                    await device.flashBlob(partition, file, progress => {
                        currentState.progress = (progress * 100).toFixed(1);
                        currentState.description = stateStr + `... ${currentState.progress}%`
                    });
                    currentState.description = stateStr;

                } catch (e) {
                    if (e instanceof FastbootError) {
                        throw Error(`刷写 ${partition} 分区失败, Bootloader返回错误: ${e.bootloaderMessage}`);
                    }
                    throw e;
                }
            } else if (action.type === "erase") {
                const partition = action.value;
                try {
                    await device.runCommand(`erase:${partition}`);
                } catch (e) {
                    if (e instanceof FastbootError) {
                        throw Error(`擦除 ${partition} 分区失败, Bootloader返回错误: ${e.bootloaderMessage}`);
                    }
                    throw e;
                }
            } else if (action.type === "reboot") {
                try {
                    await device.runCommand(`reboot`);
                } catch (e) {
                    if (e instanceof FastbootError) {
                        throw Error(`重启设备失败, Bootloader返回错误: ${e.bootloaderMessage}`);
                    }
                    throw e;
                }
            } else {
                throw Error("执行脚本失败: 未定义操作 " + action.type);
            }
            currentState.status = "success";

            stateMap.set(action, stepState);
            FlashStateList.value.push(stepState);
        }
        dialog.success({
            title: "刷写成功",
            content: "已成功刷入线刷包.",
            positiveText: "确定"
        });
    } catch (e) {
        if (e instanceof Error && currentState!) {
            currentState.description = e.message;
            currentState.status = "error";
            dialog.error({
                title: "刷写时出现问题",
                content: e.message,
                positiveText: "确定"
            });
        } else {
            dialog.error({
                title: "刷写时出现问题",
                content: String(e),
                positiveText: "确定"
            });
        }
    } finally {
        isDeviceBusy.value = false;
    }

}
</script>

<style scoped>
.animated-wrapper :deep(.n-tabs-nav--top) {
    display: none;
}
</style>