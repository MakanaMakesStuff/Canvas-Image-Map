<script lang="ts">
import { computed, defineComponent, ref } from 'vue'
import { imageMaps } from '../includes/imageMaps'
import { useRouter } from 'vue-router'

export default defineComponent({
  name: 'ImageMap'
})
</script>

<script setup lang="ts">
const router = useRouter()
const mapSlug = ref('')
const props = withDefaults(
  defineProps<{
    slug: string
    selectionOpacity: number
  }>(),
  { selectionOpacity: 0.5 }
)

const selected = computed(() => imageMaps.find((item) => item.name === props.slug))

const mappedCoords = ref<number[] | null>([])
const selectedMapCoords = ref<number[]>([])
const canvas = ref()
const image = ref()
const hovering = ref(false)

function initCanvas() {
  if (!canvas.value || !image.value) return console.info('No image or canvas found')
  canvas.value.width = image.value.naturalWidth
  canvas.value.height = image.value.naturalHeight
  drawBackground()
}

function detectHover(coords: { x: number; y: number }) {
  if (!selected.value) return

  for (const map of Object.values(selected.value.map)) {
    const [x, y, w, h] = map.coords
    const sameAsSelected =
      selectedMapCoords.value?.[0] == x &&
      selectedMapCoords.value?.[1] == y &&
      selectedMapCoords.value?.[2] == w &&
      selectedMapCoords.value?.[3] == h

    if (coords.x >= x && coords.x <= w && coords.y >= y && coords.y <= h && !sameAsSelected) {
      mappedCoords.value = map.coords

      clearCanvas()
      drawBackground()
      drawHover()

      canvas.value.style.cursor = 'pointer'
      mapSlug.value = map.slug
      return
    } else {
      canvas.value.style.cursor = 'unset'

      clearCanvas()
      drawBackground()
      drawSelected()
    }
  }
}

function drawBackground() {
  const ctx = canvas.value.getContext('2d')
  const background = new Image()
  background.src = image.value.src
  ctx.drawImage(background, 0, 0)
}

function clearCanvas() {
  if (!hovering.value) return
  hovering.value = false
  if (!canvas.value || !image.value) return console.info('No image or canvas found')
  canvas.value.width = canvas.value.height * (canvas.value.clientWidth / canvas.value.clientHeight)

  const ctx = canvas.value.getContext('2d')
  ctx.clearRect(0, 0, canvas.value.width, canvas.value.height)
  drawBackground()
}

function getMousePos(evt: MouseEvent) {
  const rect = canvas.value.getBoundingClientRect() // abs. size of element
  const scaleX = canvas.value.width / rect.width // relationship bitmap vs. element for x
  const scaleY = canvas.value.height / rect.height // relationship bitmap vs. element for y

  detectHover({
    x: (evt.clientX - rect.left) * scaleX,
    y: (evt.clientY - rect.top) * scaleY
  })
}

function drawHover() {
  if (!mappedCoords.value) return
  const [x, y, w, h] = mappedCoords.value
  const ctx = canvas.value.getContext('2d')
  ctx.globalAlpha = props.selectionOpacity
  ctx.fillStyle = 'lightgray'
  ctx.fillRect(x, y, Math.abs(w - x), Math.abs(h - y))
}

function drawSelected() {
  if (!selectedMapCoords.value) return
  const [x, y, w, h] = selectedMapCoords.value
  const ctx = canvas.value.getContext('2d')
  ctx.globalAlpha = props.selectionOpacity
  ctx.fillStyle = 'yellow'
  ctx.fillRect(x, y, Math.abs(w - x), Math.abs(h - y))
}

function setSelected() {
  if (!mappedCoords.value) return
  selectedMapCoords.value = mappedCoords.value
  const link = selected.value?.map.find((item) => item.slug == mapSlug.value)?.href

  if (link) {
    if (link.startsWith('http')) {
      window.location.href = link
    } else if (link.startsWith('/')) {
      router?.replace(link)
    } else {
      console.info(`Invalid link: ${link}`)
    }
  }

  clearCanvas()
  drawSelected()
}
</script>

<template>
  <canvas
    v-if="selected"
    ref="canvas"
    class="image-map"
    @mousemove="getMousePos($event)"
    @click="setSelected"
  >
    <img ref="image" :src="selected.img" :alt="selected.name" @load="initCanvas()" />
  </canvas>
</template>

<style lang="scss">
.image-map {
  margin: auto;
  width: 100%;
  max-width: 800px;
  height: auto;

  img {
    width: 100%;
    height: 100%;
  }
}
</style>
