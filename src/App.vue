<script setup lang="ts">
import index from './components/index.vue'
import { NNotificationProvider } from 'naive-ui'
</script>

<template>
  <n-notification-provider>
    <index />
  </n-notification-provider>
</template>

