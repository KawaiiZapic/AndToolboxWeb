<template>
  <div class="max-w-1280px mx-auto px-4">
    <n-tabs animated type="line">
      <n-tab-pane name="fastboot" tab="Fastboot">
        <n-dialog-provider>
          <fastboot-view />
        </n-dialog-provider>
      </n-tab-pane>
      <n-tab-pane name="adb" tab="ADB">
          <adb-view />
        </n-tab-pane>
    </n-tabs>
  </div>
</template>
<script lang="ts" setup>
import FastbootView from "./views/fastboot/index.vue";
import AdbView from "./views/adb/index.vue";
</script>