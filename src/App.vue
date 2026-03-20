<script setup>
import { onBeforeUnmount, onMounted, ref } from 'vue';
import { createWorker } from 'tesseract.js';

const API_URL = 'https://boxel.co.uk/api/cardreader';

const videoRef = ref(null);
const canvasRef = ref(null);
const stream = ref(null);
const capturedImage = ref('');
const extractedText = ref('');
const status = ref('Requesting camera access...');
const loading = ref(false);
const ocrProgress = ref(0);
const submitMessage = ref('');

const form = ref({
  fullName: '',
  company: '',
  jobTitle: '',
  email: '',
  phone: '',
  website: '',
  address: ''
});

const normalizePhone = (value) => value.replace(/[^\d+\-()\s]/g, '').trim();

const parseBusinessCard = (text) => {
  const lines = text
    .split('\n')
    .map((line) => line.trim())
    .filter(Boolean);

  const emailMatch = text.match(/[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}/gi);
  const phoneMatch = text.match(/(?:\+?\d[\d\s().-]{7,}\d)/g);
  const webMatch = text.match(/(?:https?:\/\/)?(?:www\.)?[a-z0-9-]+\.[a-z]{2,}(?:\/[\w\-.~:/?#[\]@!$&'()*+,;=]*)?/gi);

  const nonContactLines = lines.filter((line) => {
    const lower = line.toLowerCase();
    return !lower.includes('@') && !/\d{3,}/.test(lower) && !lower.includes('www.') && !lower.includes('http');
  });

  const fullName = nonContactLines[0] || '';
  const company = nonContactLines[1] || '';
  const jobTitle = nonContactLines[2] || '';

  const addressCandidates = lines.filter((line) => {
    const lower = line.toLowerCase();
    return /street|st\.?|road|rd\.?|avenue|ave\.?|lane|ln\.?|drive|dr\.?|suite|floor|building|blvd|boulevard|city|state|zip|postcode/.test(lower);
  });

  return {
    fullName,
    company,
    jobTitle,
    email: emailMatch?.[0] || '',
    phone: phoneMatch ? normalizePhone(phoneMatch[0]) : '',
    website: webMatch ? webMatch.find((url) => !url.includes('@')) || '' : '',
    address: addressCandidates.join(', ')
  };
};

const stopCamera = () => {
  if (stream.value) {
    stream.value.getTracks().forEach((track) => track.stop());
    stream.value = null;
  }
};

const startCamera = async () => {
  submitMessage.value = '';
  try {
    stream.value = await navigator.mediaDevices.getUserMedia({
      video: { facingMode: { ideal: 'environment' } },
      audio: false
    });

    if (videoRef.value) {
      videoRef.value.srcObject = stream.value;
      await videoRef.value.play();
    }

    status.value = 'Camera ready. Align the business card and tap Capture.';
  } catch (error) {
    status.value = `Could not access camera: ${error.message}`;
  }
};

const capturePhoto = () => {
  if (!videoRef.value || !canvasRef.value) return;

  const video = videoRef.value;
  const canvas = canvasRef.value;
  canvas.width = video.videoWidth;
  canvas.height = video.videoHeight;

  const context = canvas.getContext('2d');
  context.drawImage(video, 0, 0, canvas.width, canvas.height);
  capturedImage.value = canvas.toDataURL('image/jpeg', 0.95);
  status.value = 'Image captured. Run OCR to extract fields.';
};

const runOCR = async () => {
  if (!capturedImage.value) {
    status.value = 'Capture an image first.';
    return;
  }

  loading.value = true;
  submitMessage.value = '';
  ocrProgress.value = 0;
  status.value = 'Running OCR...';

  const worker = await createWorker('eng', 1, {
    logger: (message) => {
      if (message.status === 'recognizing text') {
        ocrProgress.value = Math.round(message.progress * 100);
      }
    }
  });

  try {
    const result = await worker.recognize(capturedImage.value);
    extractedText.value = result.data.text || '';
    form.value = {
      ...form.value,
      ...parseBusinessCard(extractedText.value)
    };
    status.value = 'OCR completed. Review and edit fields before sending.';
  } catch (error) {
    status.value = `OCR failed: ${error.message}`;
  } finally {
    await worker.terminate();
    loading.value = false;
  }
};

const submitCard = async () => {
  submitMessage.value = '';
  status.value = 'Submitting data...';

  const payload = {
    ...form.value,
    extractedText: extractedText.value,
    capturedAt: new Date().toISOString()
  };

  try {
    const response = await fetch(API_URL, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(payload)
    });

    if (!response.ok) {
      const errorText = await response.text();
      throw new Error(errorText || `API responded with ${response.status}`);
    }

    submitMessage.value = 'Business card saved successfully.';
    status.value = 'Submission complete.';
  } catch (error) {
    submitMessage.value = `Failed to submit: ${error.message}`;
    status.value = 'Submission failed.';
  }
};

onMounted(() => {
  startCamera();
});

onBeforeUnmount(() => {
  stopCamera();
});
</script>

<template>
  <main class="app">
    <h1>Business Card Scanner</h1>
    <p class="status">{{ status }}</p>

    <section class="camera-panel">
      <video ref="videoRef" playsinline autoplay muted></video>
      <canvas ref="canvasRef" class="hidden"></canvas>
      <div class="actions">
        <button type="button" @click="capturePhoto">Capture</button>
        <button type="button" @click="runOCR" :disabled="loading">
          {{ loading ? `OCR ${ocrProgress}%` : 'Extract Text' }}
        </button>
      </div>
    </section>

    <section v-if="capturedImage" class="preview-panel">
      <h2>Captured Image</h2>
      <img :src="capturedImage" alt="Captured business card" />
    </section>

    <section class="form-panel">
      <h2>Extracted Data</h2>
      <form @submit.prevent="submitCard">
        <label>
          Full Name
          <input v-model="form.fullName" type="text" />
        </label>
        <label>
          Company
          <input v-model="form.company" type="text" />
        </label>
        <label>
          Job Title
          <input v-model="form.jobTitle" type="text" />
        </label>
        <label>
          Email
          <input v-model="form.email" type="email" />
        </label>
        <label>
          Phone
          <input v-model="form.phone" type="tel" />
        </label>
        <label>
          Website
          <input v-model="form.website" type="url" />
        </label>
        <label>
          Address
          <textarea v-model="form.address" rows="3"></textarea>
        </label>

        <button class="submit" type="submit">Save To API</button>
      </form>
      <p v-if="submitMessage" class="submit-message">{{ submitMessage }}</p>
    </section>

    <section v-if="extractedText" class="raw-text">
      <h2>Raw OCR Text</h2>
      <pre>{{ extractedText }}</pre>
    </section>
  </main>
</template>
