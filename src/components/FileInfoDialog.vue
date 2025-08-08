<script setup lang="ts">
import type { Input } from 'mediabunny';
import { ref, watch } from 'vue';

type TrackInfo = {
    'Type': string;
    'Codec': string;
    'Full Codec String': string;
    'Duration (seconds)': string;
    'Language Code': string;
    'Packet Count': number;
    'Packet Rate (avg.)': string;
    'Bitrate (avg.)': string;
} & ({
    'Coded Width': number;
    'Coded Height': number;
    'Rotation': number;
} | {
    'Channels': number;
    'Sample Rate (Hz)': number;
});

const props = defineProps<{
    input: Input | null;
}>();

defineEmits<{
    (e: 'close'): void;
}>();

const format = ref<string>('');
const mineType = ref<string>('');
const duration = ref<number>(0);
const tracks = ref<TrackInfo[]>([]);

function formatBitrate(bps: number): string {
    if (bps == null || !isFinite(bps) || bps < 0) return '';
    const units = ['bps', 'Kbps', 'Mbps', 'Gbps', 'Tbps']; // decimal units for bitrate
    let i = 0;
    let value = bps;
    while (value >= 1000 && i < units.length - 1) {
        value /= 1000;
        i++;
    }
    return `${value.toFixed(i === 0 ? 0 : 2)} ${units[i]}`;
}

watch(props, (newInput) => {
    const { input } = newInput;
    if (input) {
        input.getFormat().then(f => { format.value = f.name; });
        input.getMimeType().then(m => { mineType.value = m; });
        input.computeDuration().then(d => {
            duration.value = Math.round(d * 1000) / 1000;
        });
        input.getTracks().then(async t => {
            tracks.value = await Promise.all(t.map(track => (async () => {
                const stats = await track.computePacketStats();
                return {
                    'Type': track.type,
                    'Codec': track.codec,
                    'Full Codec String': await track.getCodecParameterString(),
                    'Duration (seconds)': await track.computeDuration().then(d => Math.round(d * 1000) / 1000),
                    'Language Code': track.languageCode || '',
                    'Packet Count': stats.packetCount,
                    'Packet Rate (avg.)': track.isVideoTrack() ? `${stats.averagePacketRate.toFixed(2)} FPS` : `${stats.averagePacketRate.toFixed(2)}`,
                    'Bitrate (avg.)': formatBitrate(stats.averageBitrate),
                    ...(track.isVideoTrack() ? {
                        'Coded Width': track.codedWidth,
                        'Coded Height': track.codedHeight,
                        'Rotation': track.rotation,
                    } : track.isAudioTrack() ? {
                        'Channels': track.numberOfChannels,
                        'Sample Rate (Hz)': track.sampleRate,
                    } : {})
                } as TrackInfo;
            })()));
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
            <div class="modal-header">
                <h5 class="modal-title">File Information</h5>
                <button type="button" @click="$emit('close')" class="btn-close" data-bs-dismiss="modal"
                    aria-label="Close"></button>
            </div>
            <div class="modal-body">
                <table class="table">
                    <tbody>
                        <tr><th>Format</th><td>{{ format }}</td></tr>
                        <tr><th>Mime Type</th><td>{{ mineType }}</td></tr>
                        <tr><th>Duration</th><td>{{ duration }} seconds</td></tr>
                    </tbody>
                </table>
                <h5>Tracks</h5>
                <div v-for="(track, index) in tracks" :key="index" class="mb-3">
                    <h6>Track {{ index + 1 }} ({{ track.Type }})</h6>
                    <table class="table">
                        <thead>
                            <tr>
                                <th>Property</th>
                                <th>Value</th>
                            </tr>
                        </thead>
                        <tbody>
                            <tr v-for="(value, key) in track" :key="key">
                                <td>{{ key }}</td><td>{{ value }}</td>
                            </tr>
                        </tbody>
                    </table>
                </div>
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
