<script setup lang="ts">
import { SecLabIcon } from '@seclab-dev/vue'
</script>

<template>
  <div class="app-shell" data-page="api-governance">
    <div class="app-bar" data-ui="app-bar">
      <div class="brand">
        <div class="brand-mark"><SecLabIcon name="braces" :size="19" /></div>
        <div>
          <div class="brand-name">SecLab API</div>
          <div class="brand-caption">接口治理中心</div>
        </div>
      </div>
      <div class="app-meta">OpenAPI 3.1</div>
    </div>
    <RouterView />
  </div>
</template>
