<script setup lang="ts">
import type QRCodeStyling from 'qr-code-styling'
import type { CornerDotType, CornerSquareType, DotType, ErrorCorrectionLevel, Options } from 'qr-code-styling'

useHead({
  meta: [{ name: 'viewport', content: 'width=device-width, initial-scale=1' }],
  // GitHub Pages serves the site under /<repo>/, so prefix the base URL
  link: [{ rel: 'icon', href: `${useRuntimeConfig().app.baseURL}favicon.ico` }],
  htmlAttrs: { lang: 'en' }
})
useSeoMeta({ title: 'QR Code Generator', description: 'Make styled QR codes and download them as PNG, JPG or SVG.' })

const dotTypes: DotType[] = ['square', 'dots', 'rounded', 'extra-rounded', 'classy', 'classy-rounded']
const cornerSquareTypes: CornerSquareType[] = ['square', 'dot', 'extra-rounded', 'rounded', 'dots', 'classy', 'classy-rounded']
const cornerDotTypes: CornerDotType[] = ['square', 'dot', 'rounded', 'dots', 'classy', 'classy-rounded', 'extra-rounded']
const eclItems: { label: string, value: ErrorCorrectionLevel }[] = [
  { label: 'L (7%)', value: 'L' },
  { label: 'M (15%)', value: 'M' },
  { label: 'Q (25%)', value: 'Q' },
  { label: 'H (30%)', value: 'H' }
]
const formats = [
  { label: 'PNG', value: 'png' },
  { label: 'JPG', value: 'jpeg' },
  { label: 'SVG', value: 'svg' }
] as const

const form = reactive({
  url: 'https://example.com',
  size: 300,
  margin: 10,
  ecl: 'Q' as ErrorCorrectionLevel,
  dotsType: 'rounded' as DotType,
  dotsColor: '#000000',
  csType: 'extra-rounded' as CornerSquareType,
  csColor: '#000000',
  cdType: 'dot' as CornerDotType,
  cdColor: '#000000',
  bgColor: '#FFFFFF',
  transparent: false
})
const format = ref<(typeof formats)[number]['value']>('png')
const flat = ref(false)

// number fields are empty while the user clears them
const size = () => form.size || 300
const margin = () => form.margin || 0

function options(): Options {
  return {
    type: 'svg',
    data: form.url || ' ',
    width: size(),
    height: size(),
    margin: margin(),
    qrOptions: { errorCorrectionLevel: form.ecl },
    dotsOptions: { type: form.dotsType, color: form.dotsColor },
    cornersSquareOptions: { type: form.csType, color: form.csColor },
    cornersDotOptions: { type: form.cdType, color: form.cdColor },
    backgroundOptions: { color: form.transparent ? 'transparent' : form.bgColor }
  }
}

// preview is always drawn at a fixed size; scaling the SVG in the browser shows seams between modules
const PREVIEW = 256
function previewOptions(): Options {
  const o = options()
  return { ...o, width: PREVIEW, height: PREVIEW, margin: (margin() * PREVIEW) / size() }
}

const preview = useTemplateRef('preview')
let QRCode: typeof QRCodeStyling
let qr: QRCodeStyling

// the library needs the DOM, so load it in the browser only
onMounted(async () => {
  QRCode = (await import('qr-code-styling')).default
  qr = new QRCode(previewOptions())
  qr.append(preview.value!)
})
watch(form, () => qr?.update(previewOptions()))

// Affinity and some other design apps ignore clip-path and draw each clipped rect in full,
// so replace every clipped rect with its clip shapes, filled directly
function flattenSvg(svg: string) {
  const doc = new DOMParser().parseFromString(svg, 'image/svg+xml')
  const root = doc.documentElement
  for (const el of [...root.children]) {
    const id = el.getAttribute('clip-path')?.match(/#([^')]+)/)?.[1]
    const clip = id && doc.getElementById(id)
    if (!clip) continue
    clip.remove()
    const fill = el.getAttribute('fill')
    // some apps draw fill="transparent" as black; leaving the shape out is the same picture
    if (fill === 'transparent') {
      el.remove()
      continue
    }
    const g = doc.createElementNS(root.namespaceURI, 'g')
    g.setAttribute('fill', fill!)
    // touching shapes show hairline seams when anti-aliased; a thin same-color outline closes them
    g.setAttribute('stroke', fill!)
    g.setAttribute('stroke-width', String(Number(root.getAttribute('width')) / 1200))
    for (const shape of [...clip.children]) {
      const rule = shape.getAttribute('clip-rule')
      if (rule) shape.setAttribute('fill-rule', rule)
      shape.removeAttribute('clip-rule')
      g.append(shape)
    }
    el.replaceWith(g)
  }
  return new XMLSerializer().serializeToString(doc)
}

async function download() {
  const opts = options()
  // JPG has no alpha channel; transparent pixels would turn black, so use white
  if (format.value === 'jpeg' && form.transparent) opts.backgroundOptions = { color: '#FFFFFF' }
  const code = new QRCode(opts)
  if (format.value !== 'svg' || !flat.value) return code.download({ name: 'qrcode', extension: format.value })

  const raw = await code.getRawData('svg') as Blob
  const a = document.createElement('a')
  a.href = URL.createObjectURL(new Blob([flattenSvg(await raw.text())], { type: 'image/svg+xml' }))
  a.download = 'qrcode.svg'
  a.click()
  URL.revokeObjectURL(a.href)
}
</script>

<template>
  <UApp>
    <UHeader title="QR Code Generator">
      <template #right>
        <UColorModeButton />
      </template>
    </UHeader>

    <UMain>
      <UContainer class="py-8 grid gap-6 lg:grid-cols-[1fr_340px] items-start">
        <UCard :ui="{ body: 'space-y-8' }">
          <UFormField
            label="URL"
            size="xl"
          >
            <UInput
              v-model="form.url"
              type="url"
              icon="i-lucide-link"
              placeholder="https://..."
              class="w-full"
            />
          </UFormField>

          <section class="space-y-3">
            <h2 class="font-semibold">
              Size and error correction
            </h2>
            <div class="grid gap-4 sm:grid-cols-3">
              <UFormField label="Size (px)">
                <UInputNumber
                  v-model="form.size"
                  :min="100"
                  :max="2000"
                  :step="10"
                  class="w-full"
                />
              </UFormField>
              <UFormField label="Margin (px)">
                <UInputNumber
                  v-model="form.margin"
                  :min="0"
                  :max="200"
                  class="w-full"
                />
              </UFormField>
              <UFormField label="Error correction">
                <USelect
                  v-model="form.ecl"
                  :items="eclItems"
                  class="w-full"
                />
              </UFormField>
            </div>
          </section>

          <section class="space-y-3">
            <h2 class="font-semibold">
              Dots
            </h2>
            <div class="grid gap-4 sm:grid-cols-2">
              <UFormField label="Style">
                <USelect
                  v-model="form.dotsType"
                  :items="dotTypes"
                  class="w-full"
                />
              </UFormField>
              <UFormField label="Color">
                <ColorField v-model="form.dotsColor" />
              </UFormField>
            </div>
          </section>

          <section class="space-y-3">
            <h2 class="font-semibold">
              Corner squares
            </h2>
            <div class="grid gap-4 sm:grid-cols-2">
              <UFormField label="Style">
                <USelect
                  v-model="form.csType"
                  :items="cornerSquareTypes"
                  class="w-full"
                />
              </UFormField>
              <UFormField label="Color">
                <ColorField v-model="form.csColor" />
              </UFormField>
            </div>
          </section>

          <section class="space-y-3">
            <h2 class="font-semibold">
              Corner dots
            </h2>
            <div class="grid gap-4 sm:grid-cols-2">
              <UFormField label="Style">
                <USelect
                  v-model="form.cdType"
                  :items="cornerDotTypes"
                  class="w-full"
                />
              </UFormField>
              <UFormField label="Color">
                <ColorField v-model="form.cdColor" />
              </UFormField>
            </div>
          </section>

          <section class="space-y-3">
            <h2 class="font-semibold">
              Background
            </h2>
            <div class="grid gap-4 sm:grid-cols-2 items-end">
              <UFormField label="Color">
                <ColorField
                  v-model="form.bgColor"
                  :disabled="form.transparent"
                />
              </UFormField>
              <USwitch
                v-model="form.transparent"
                label="Transparent"
                class="pb-2"
              />
            </div>
          </section>
        </UCard>

        <UCard
          class="lg:sticky lg:top-(--ui-header-height) max-lg:-order-1"
          :ui="{ body: 'space-y-4' }"
        >
          <!-- checkerboard so transparency is visible -->
          <div class="grid place-items-center rounded-md p-4 bg-[repeating-conic-gradient(#e5e7eb_0_25%,#fff_0_50%)] bg-size-[16px_16px]">
            <div ref="preview" />
          </div>

          <UFieldGroup class="w-full">
            <USelect
              v-model="format"
              :items="[...formats]"
              aria-label="File format"
              class="w-28"
            />
            <UButton
              label="Download"
              icon="i-lucide-download"
              block
              @click="download"
            />
          </UFieldGroup>
          <USwitch
            v-if="format === 'svg'"
            v-model="flat"
            label="Flat SVG"
            description="For Affinity and other design apps that show the code as a blank square"
          />
        </UCard>
      </UContainer>
    </UMain>
  </UApp>
</template>
