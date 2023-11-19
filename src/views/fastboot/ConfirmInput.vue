<template>
    <n-input :placeholder="`请输入 ${props.confirmWord} 确认操作`" v-model:value="inputConfirmWord" />
</template>

<script lang="ts" setup>
const props = defineProps<{
    confirmWord: string
}>();

const emit = defineEmits<{
    (e: "validated", v: boolean): void
}>();

const inputConfirmWord = ref("");
let isConfirmed = false;

watch(inputConfirmWord, v => {
    if ((v === props.confirmWord) !== isConfirmed) {
        isConfirmed = v === props.confirmWord;
        emit("validated", isConfirmed);
    }
});
</script>