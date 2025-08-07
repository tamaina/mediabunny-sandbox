<script setup lang="ts">
import type { Input } from 'mediabunny';
import { ref, watch } from 'vue';

const props = defineProps<{
    input: Input | null;
}>();

defineEmits<{
    (e: 'close'): void;
}>();

const format = ref<string>('');
const mineType = ref<string>('');
const duration = ref<number>(0);

watch(props, (newInput) => {
    const { input } = newInput;
    if (input) {
        input.getFormat().then(f => { format.value = f.name; });
        input.getMimeType().then(m => { mineType.value = m; });
        input.computeDuration().then(d => {
            duration.value = Math.round(d * 1000) / 1000; // Convert milliseconds to seconds
        });
    } else {
        format.value = '';
        mineType.value = '';
        duration.value = 0;
    }
}, { immediate: true });
</script>

<template>
<div ref="root" class="modal" :class="$style.root" tabindex="-1" @click.self="$emit('close')">
    <div class="modal-dialog modal-lg modal-dialog-scrollable">
        <div class="modal-content">
            <form class="modal-header" method="dialog">
                <h5 class="modal-title">File Information</h5>
                <button type="button" @click="$emit('close')" class="btn-close" data-bs-dismiss="modal"
                    aria-label="Close"></button>
            </form>
            <div class="modal-body">
                <table class="table">
                    <tbody>
                        <tr>
                            <th>Format</th>
                            <td>{{ format }}</td>
                        </tr>
                        <tr>
                            <th>Mime Type</th>
                            <td>{{ mineType }}</td>
                        </tr>
                        <tr>
                            <th>Duration</th>
                            <td>{{ duration }} seconds</td>
                        </tr>
                    </tbody>
                </table>
            </div>
        </div>
    </div>
</div>
</template>

<style lang="scss" module>
.root {
    display: block;
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.5);
    z-index: 1050;
}
.modal-dialog {
    max-width: 90%;
    margin: auto;
}
</style>
