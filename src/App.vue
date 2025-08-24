<script setup lang="ts">
import { computed, ref, shallowRef, watch } from 'vue';
import FileInfoDialog from './components/FileInfoDialog.vue';
import { ALL_FORMATS, BlobSource, Conversion, Input, MkvOutputFormat, MovOutputFormat, Mp4OutputFormat, Output, QUALITY_HIGH, QUALITY_LOW, QUALITY_MEDIUM, QUALITY_VERY_HIGH, QUALITY_VERY_LOW, StreamTarget, UrlSource, WebMOutputFormat } from 'mediabunny';

const fileInput = ref<HTMLInputElement | null>(null);
const file = ref<File | null>(null);
const input = computed(() => file.value ?
  new Input({
    formats: ALL_FORMATS,
    source: new BlobSource(file.value),
  }) :
  null
);

const currentConversion = ref<Conversion | null>(null);
const currentConversionPromise = ref<Promise<void> | null>(null);
const progressPercentage = ref<number>(0);
const logs = ref<string[]>([]);

const showFileInfoDialog = ref(false);

const enableWidth = ref(true);
const enableHeight = ref(false);
const enableFps = ref(false);
const width = ref<number>(720);
const height = ref<number>(720);
const fps = ref<string>('29.97');
const fit = ref<'contain' | 'cover' | 'fill'>('fill');
const container = ref<'fmp4' | 'mp4' | 'mov' | 'webm' | 'mkv'>('fmp4');
type VideoCodec = 'avc' | 'vp9' | 'av1';
const vcodec = ref<VideoCodec | 'AS_IS' | 'DISCARD'>('avc');
type AudioCodec = 'aac' | 'mp3' | 'opus' | 'vorbis' | 'flac' | 'pcm-s16' | 'pcm-s24' | 'pcm-f32';
const acodec = ref<AudioCodec | 'AS_IS' | 'DISCARD'>('opus'); // Default to opus due to Firefox support

type BitrateQuality = 'verylow' | 'low' | 'medium' | 'high' | 'veryhigh';
const vbitrate = ref<BitrateQuality>('medium');
const abitrate = ref<BitrateQuality>('medium');

const keyFrameSeconds = ref<number>(2);
const aSampleRate = ref<'44.1' | '48' | '96' | '192' | '384'>('96');

const fs = shallowRef<FileSystemDirectoryHandle | null>(null);
const resultUrl = ref<string | null>(null);
const resultMimeType = ref<string | null>(null);
const resultFileName = ref<string | null>(null);
const resultInput = ref<Input | null>(null);
const showResultInfoDialog = ref(false);
watch(resultUrl, () => {
  if (resultUrl.value) {
    resultInput.value = new Input({
      formats: ALL_FORMATS,
      source: new UrlSource(resultUrl.value),
    });
  }
});

const resultFileSize = ref<{ source: number; result: number }>();

function formatSize(bytes: number | null | undefined, fractionDigits = 2): string {
  if (bytes == null || isNaN(bytes)) return '';
  const units = ['B', 'KB', 'MB', 'GB', 'TB'];
  if (bytes === 0) return '0 B';
  const i = Math.min(Math.floor(Math.log(bytes) / Math.log(1024)), units.length - 1);
  const value = bytes / Math.pow(1024, i);
  return `${value.toFixed(i === 0 ? 0 : fractionDigits)} ${units[i]}`;
}

function convertBitrate(quality: BitrateQuality) {
  switch (quality) {
    case 'verylow': return QUALITY_VERY_LOW;
    case 'low': return QUALITY_LOW;
    case 'medium': return QUALITY_MEDIUM;
    case 'high': return QUALITY_HIGH;
    case 'veryhigh': return QUALITY_VERY_HIGH;
    default: return QUALITY_MEDIUM;
  }
}

async function transpile() {
  if (resultUrl.value) URL.revokeObjectURL(resultUrl.value);
  resultUrl.value = null;
  resultMimeType.value = null;
  resultFileSize.value = undefined;

  if (!input.value) return;

  const sourceFileSize = file.value?.size || 0;

  try {
    if (!fs.value) {
      fs.value = await navigator.storage.getDirectory();
    }
    await fs.value.removeEntry('result').catch(() => {});

    //const transformer = new TransformStream({
    //  async transform(chunk, controller) {
    //    // Process the chunk and push the result to the controller
    //    logs.value.push(`${chunk.data.byteLength} bytes added`);
    //    controller.enqueue(chunk);
    //  }
    //});

    logs.value.push(`${new Date().toJSON()}`);
    logs.value.push('Starting transpiling...');

    const output = new Output({
      format: container.value === 'mp4'
          ? new Mp4OutputFormat({
            fastStart: 'in-memory',
          })
          : container.value === 'fmp4'
            ? new Mp4OutputFormat({
              fastStart: 'fragmented',
              minimumFragmentDuration: keyFrameSeconds.value,
            })
          : container.value === 'mov'
            ? new MovOutputFormat({
              fastStart: 'fragmented',
              minimumFragmentDuration: keyFrameSeconds.value,
            })
          : container.value === 'webm'
            ? new WebMOutputFormat()
            : new MkvOutputFormat(),
      target: new StreamTarget(await fs.value.getFileHandle('result', { create: true }).then(handle => handle.createWritable())),
    });

    const conversion = await Conversion.init({
      input: input.value,
      output: output,
      video: vcodec.value === 'DISCARD' ? {
        discard: true,
      } :
        vcodec.value === 'AS_IS' ? 
        undefined
       : {
          codec: vcodec.value,
          bitrate: convertBitrate(vbitrate.value),
          width: enableWidth.value ? width.value : undefined,
          height: enableHeight.value ? height.value : undefined,
          fit: fit.value,
          frameRate: enableFps.value ? Number(fps.value) : undefined,
          forceTranscode: true,
        },
      audio: acodec.value === 'DISCARD' ? {
        discard: true,
      } :
        acodec.value === 'AS_IS' ? 
        undefined
      : {
          codec: acodec.value,
          bitrate: convertBitrate(abitrate.value),
          sampleRate: aSampleRate.value === '44.1' ? 44100 :
            aSampleRate.value === '48' ? 48000 :
            aSampleRate.value === '96' ? 96000 :
            aSampleRate.value === '192' ? 192000 :
            aSampleRate.value === '384' ? 384000 : undefined,
          forceTranscode: true,
        },
    });

    conversion.discardedTracks.forEach(track => {
      logs.value.push(`Track ${track.track.id} (${track.track.type}) discarded: ${track.reason}`);
    });

    conversion.onProgress = async (progress) => {
      progressPercentage.value = Math.round(progress * 100);
    };

    currentConversionPromise.value = conversion.execute().then(async () => {
      logs.value.push('Transpiling completed.');

      fs.value?.getFileHandle('result')
        .then(handle => handle.getFile())
        .then(file => {
          if (file) {
            resultUrl.value = URL.createObjectURL(file);
            resultFileSize.value = { source: sourceFileSize, result: file.size };
          }
        });

      conversion.discardedTracks.forEach(track => {
        logs.value.push(`Track ${track.track.id} (${track.track.type}) discarded: ${track.reason}`);
      });
  
      currentConversion.value = null;
      currentConversionPromise.value = null;
    }).catch(error => {
      currentConversion.value = null;
      currentConversionPromise.value = null;
      progressPercentage.value = 0;
      logs.value.push(`Transpiling failed: ${error instanceof Error ? error.message : String(error)}`);
      console.error(error);
    });

    output.getMimeType().then(mimeType => {
      resultMimeType.value = mimeType;
    });
    resultFileName.value = `${Date.now()}${output.format.fileExtension}`;

    currentConversion.value = conversion;
  } catch (error) {
    console.error(error);
    logs.value.push(`Transpiling failed: ${error instanceof Error ? error.message : String(error)}`);
    currentConversion.value = null;
    currentConversionPromise.value = null;
  }
}

async function cancelTranspile() {
  if (currentConversion.value) {
    currentConversion.value.cancel();
    logs.value.push('Transpiling cancelled.');
    currentConversion.value = null;
    currentConversionPromise.value = null;
  }
  if (resultUrl.value) URL.revokeObjectURL(resultUrl.value);
  resultUrl.value = null;
}
</script>

<template>
  <nav class="navbar bg-body-tertiary">
    <div class="container">
      <h1 class="navbar-brand" href="/">Video Transpiler on Browser</h1>
      <span>powered by <a href="https://mediabunny.dev" target="_blank">Mediabunny</a></span>
    </div>
  </nav>
  <div class="content">
    <div class="container">
      <div>
        <div class="input-group my-3">
          <input type="file" class="form-control flex-grow-1" ref="fileInput" accept="video/*,audio/*" @change="file = fileInput?.files?.[0] || null" />
          <span v-if="file" class="input-group-text" id="basic-addon1">{{ formatSize(file.size) }}</span>
          <button v-if="file" class="btn btn-primary" @click="showFileInfoDialog = true">Show File Info</button>
        </div>
      </div>
      <div class="row mt-5">
        <div>
          <h4>Transpile</h4>
        </div>

        <div class="col-lg-6 my-2">
          <label class="form-label">Video Size</label>
          <div class="input-group mb-3">
            <div class="input-group-text">
              <div class="form-check mt-0">
                <input class="form-check-input" type="checkbox" id="enableWidth" v-model="enableWidth" aria-label="Enable width">
                <label class="form-check-label" for="enableWidth">Width</label>
              </div>
            </div>
            <input type="number" class="form-control" :class="$style.whinput" aria-label="Width" v-model="width" />
            <div class="input-group-text">
              <div class="form-check mt-0">
                <input class="form-check-input" type="checkbox" id="enableHeight" v-model="enableHeight" aria-label="Enable height">
                <label class="form-check-label" for="enableHeight">Height</label>
              </div>
            </div>
            <input type="number" class="form-control" :class="$style.whinput" aria-label="Height" v-model="height" />
            <select class="form-select" aria-label="Fit" v-model="fit">
              <option value="contain">contain</option>
              <option value="cover">cover</option>
              <option value="fill">fill</option>
            </select>
          </div>
          <label class="form-label">Video Settings</label>
          <div class="input-group mb-3">
            <div class="input-group-text">
              <div class="form-check mt-0">
                <input class="form-check-input" type="checkbox" id="enableFps" v-model="enableFps" aria-label="Enable FPS">
                <label class="form-check-label" for="enableFps">fps</label>
              </div>
            </div>
            <select class="form-select" id="vFps" aria-label="Video FPS" v-model="fps">
              <option value="15">15</option>
              <option value="24">24</option>
              <option value="25">25</option>
              <option value="29.97">29.97</option>
              <option value="30">30</option>
              <option value="48">48</option>
              <option value="50">50</option>
              <option value="59.94">59.94</option>
              <option value="60">60</option>
              <option value="100">100</option>
              <option value="119.88">119.88</option>
              <option value="120">120</option>
            </select>
            <div class="form-floating">
              <select class="form-select" id="vBitrate" aria-label="Video Bitrate" v-model="vbitrate">
                <option value="verylow">Very Low</option>
                <option value="low">Low</option>
                <option value="medium">Medium</option>
                <option value="high">High</option>
                <option value="veryhigh">Very High</option>
              </select>
              <label for="vBitrate">Video Bitrate</label>
            </div>
            <div class="form-floating">
              <input type="number" class="form-control" id="keyFrameInterval" aria-label="Key Frame Interval" v-model="keyFrameSeconds" />
              <label for="keyFrameInterval">Key Frame Interval (sec.)</label>
            </div>
          </div>
        </div>

        <div class="col-lg-6 my-2">
          <label class="form-label">Audio Settings</label>
          <div class="input-group mb-3">
            <div class="form-floating">
              <select class="form-select" id="aBitrate" aria-label="Audio Bitrate" v-model="abitrate">
                <option value="verylow">Very Low</option>
                <option value="low">Low</option>
                <option value="medium">Medium</option>
                <option value="high">High</option>
                <option value="veryhigh">Very High</option>
              </select>
              <label for="aBitrate">Audio Bitrate</label>
            </div>
            <div class="form-floating">
              <select class="form-select" id="aSampleRate" aria-label="Audio Sample Rate" v-model="aSampleRate">
                <option value="44.1">44.1 kHz</option>
                <option value="48">48 kHz</option>
                <option value="96">96 kHz</option>
                <option value="192">192 kHz</option>
                <option value="384">384 kHz</option>
              </select>
              <label for="aSampleRate">Audio Sample Rate</label>
            </div>
          </div>
        </div>

        <div class="col-lg-6 my-2">
          <label class="form-label">Output Type</label>
          <div class="input-group mb-3">
            <div class="form-floating">
              <select class="form-select" id="formContainer" aria-label="Fit" v-model="container">
                <option value="fmp4">Fragmented MP4</option>
                <option value="mp4">MP4</option>
                <option value="mov">MOV</option>
                <option value="webm">WebM</option>
                <option value="mkv">MKV</option>
              </select>
              <label for="formContainer">Container Format</label>
            </div>
            <div class="form-floating">
              <select class="form-select" id="formVideoCodec" aria-label="Fit" v-model="vcodec">
                <option value="avc">H.264</option>
                <option value="vp9">VP9</option>
                <option value="av1">AV1</option>
                <option value="AS_IS">No video transpile</option>
                <option value="DISCARD">Discard video tracks</option>
              </select>
              <label for="formVideoCodec">Video Codec</label>
            </div>
            <div class="form-floating">
              <select class="form-select" id="formAudioCodec" aria-label="Fit" v-model="acodec">
                <option value="aac">AAC</option>
                <option value="mp3">MP3</option>
                <option value="opus">Opus</option>
                <option value="vorbis">Vorbis</option>
                <option value="flac">FLAC</option>
                <option value="pcm-s16">PCM 16-bit</option>
                <option value="pcm-s24">PCM 24-bit</option>
                <option value="pcm-f32">PCM 32-bit float</option>
                <option value="AS_IS">No audio transpile</option>
                <option value="DISCARD">Discard audio tracks</option>
              </select>
              <label for="formAudioCodec">Audio Codec</label>
            </div>
          </div>

          <button v-if="currentConversion" class="btn btn-warning w-100" @click="cancelTranspile">Cancel Transpile</button>
          <button v-else-if="file" class="btn btn-primary w-100" @click="transpile">Start Transpile</button>
          <button v-else class="btn btn-outline-secondary w-100" disabled>Choose File First</button>
        </div>

        <div class="col-lg-6 my-2">
          <div class="progress mb-3" role="progressbar" aria-label="Transpile progress" aria-valuenow="0" aria-valuemin="0" aria-valuemax="100">
            <div v-if="currentConversion" class="progress-bar progress-bar-striped progress-bar-animated" :style="{ width: progressPercentage + '%' }">{{ progressPercentage }}%</div>
            <div v-else class="progress-bar" style="width: 0%"></div>
          </div>
          <textarea class="form-control" :class="$style.logs" rows="5" :value="Array.from(logs).reverse().join('\n')" readonly />
        </div>

        <div class="mt-5 mb-3">
          <div v-if="resultUrl" class="input-group mb-3">
            <a class=" btn btn-primary flex-grow-1" :href="resultUrl" :download="resultFileName">Download</a>
            <button class="btn btn-secondary" @click="showResultInfoDialog = true">Show Info</button>
          </div>
          <a v-else class="btn btn-outline-primary disabled w-100">No video available for download</a>
          <p v-if="resultFileSize" class="text-center text-muted">Result {{ formatSize(resultFileSize.result) }} / Source {{ formatSize(resultFileSize.source) }}</p>
          <video v-if="resultUrl" class="w-100" style="max-height: 400px; object-fit: contain;" controls>
            <source :src="resultUrl" :type="resultMimeType || undefined" />
            Your browser does not support the video tag.
          </video>
        </div>
      </div>
    </div>
  </div>
  <FileInfoDialog v-if="showFileInfoDialog" ref="fileInfoDialog" :input="input" @close="showFileInfoDialog = false" />
  <FileInfoDialog v-if="showResultInfoDialog" ref="resultInfoDialog" :input="resultInput" @close="showResultInfoDialog = false" />
</template>

<style lang="scss" module>
.whinput {
  min-width: 6rem !important;
}

.logs {
  font-size: 0.8rem;
  font-family: monospace;
}
</style>
