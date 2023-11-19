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
import { FastbootDevice } from "android-fastboot";

let device = new FastbootDevice();
const isConnected = ref(device.isConnected);
const DeviceInfo = reactive({} as Record<string, string>);
const dialog = useDialog();
const isBrowserSupported = "usb" in navigator;

const isDeviceInfoLoading = ref(false);
const isUserCancelSelect = ref(false);

navigator.usb?.addEventListener("disconnect", e => {
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


</script>

<style scoped>
.animated-wrapper :deep(.n-tabs-nav--top) {
    display: none;
}
</style>