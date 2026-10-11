<script setup lang="ts">
defineProps<{ disabled?: boolean }>()
const color = defineModel<string>({ required: true })
</script>

<template>
  <UPopover>
    <UButton
      :label="color"
      :disabled="disabled"
      color="neutral"
      variant="outline"
      block
      class="justify-start font-mono"
    >
      <template #leading>
        <span
          class="size-4 rounded-sm ring ring-default"
          :style="{ backgroundColor: color }"
        />
      </template>
    </UButton>

    <template #content>
      <UColorPicker
        v-model="color"
        class="p-2"
      />
    </template>
  </UPopover>
</template>
