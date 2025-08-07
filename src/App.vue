<script setup lang="ts">
import { computed, ref } from 'vue';
import FileInfoDialog from './components/FileInfoDialog.vue';
import { ALL_FORMATS, BlobSource, Input } from 'mediabunny';

const fileInput = ref<HTMLInputElement | null>(null);
const file = ref<File | null>(null);
const input = computed(() => file.value ?
  new Input({
    formats: ALL_FORMATS,
    source: new BlobSource(file.value),
  }) :
  null
);
const showFileInfoDialog = ref(false);
</script>

<template>
  <nav class="navbar bg-body-tertiary">
    <div class="container-fluid">
      <a class="navbar-brand" href="/">Mediabunny Test</a>
    </div>
  </nav>
  <div class="content">
    <div class="container">
      <div class="row">
      <div class="col-md-8 py-3">
        <input type="file" class="form-control" ref="fileInput" accept="video/*,audio/*" @change="file = fileInput?.files?.[0] || null" />
      </div>
      <div class="col-md-4 py-3">
        <button v-if="file" class="btn btn-primary w-100" @click="showFileInfoDialog = true">Show File Info</button>
        <a v-else class="btn btn-primary disabled w-100" role="button" aria-disabled="true">Show File Info</a>
      </div>
      </div>
    </div>
  </div>
  <FileInfoDialog v-if="showFileInfoDialog" ref="fileInfoDialog" :input="input" @close="showFileInfoDialog = false" />
</template>

<style lang="scss" module>

</style>
