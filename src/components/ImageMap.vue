<script lang="ts">
import {
	computed,
	defineComponent,
	onBeforeMount,
	onMounted,
	reactive,
	ref,
	watch,
} from 'vue'
import { imageMaps } from '../includes/imageMaps'
import { useRouter } from 'vue-router';
import { cloneDeep } from 'lodash'

export default defineComponent({
	name: 'ULPublicImageMap',
})
</script>

<script setup lang="ts">
const router = useRouter()
const mapSlug = ref('')
const props = defineProps<{
	slug: string
}>()

const selected = computed(() =>
	cloneDeep(imageMaps.find(item => item.name === props.slug))
)

const canvas = ref()
const image = ref()
const hovering = ref(false)

function initCanvas() {
	if (!canvas.value || !image.value)
		return console.info('No image or canvas found')
	canvas.value.width = image.value.naturalWidth
	canvas.value.height = image.value.naturalHeight
	drawBackground()
}

function detectHover(coords: { x: number; y: number }) {
	if (!selected.value) return
	for (const map of Object.values(selected.value.map)) {
		const [x, y, w, h] = map.coords
		if (coords.x >= x && coords.x <= w && coords.y >= y && coords.y <= h) {
			clearCanvas()
			drawRectToCanvas(map.coords)
			canvas.value.style.cursor = 'pointer'
			mapSlug.value = map.slug
			return
		} else {
			canvas.value.style.cursor = 'unset'
			clearCanvas()
		}
	}
}

function setQuery() {}

function drawRectToCanvas(coords: number[]) {
	if (hovering.value) return
	hovering.value = true
	if (!canvas.value || !image.value)
		return console.info('No image or canvas found')
	canvas.value.width =
		canvas.value.height * (canvas.value.clientWidth / canvas.value.clientHeight)
	canvas.value.height = image.value.height
	const ctx = canvas.value.getContext('2d')
	const [x, y, w, h] = coords
	ctx.fillStyle = 'yellow'
	ctx.fillRect(x, y, Math.abs(w - x), Math.abs(h - y))
	drawBackground()
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
	if (!canvas.value || !image.value)
		return console.info('No image or canvas found')
	canvas.value.width =
		canvas.value.height * (canvas.value.clientWidth / canvas.value.clientHeight)

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
		y: (evt.clientY - rect.top) * scaleY,
	})
}

function setMapSlug() {
	if (!mapSlug.value || mapSlug.value === '') return
	router.push({ query: { mapslug: mapSlug.value } })
}
</script>

<template>
	<canvas
		v-if="selected"
		ref="canvas"
		class="ul-public-image-map"
		@mousemove="getMousePos($event)"
		@click="setMapSlug"
	>
		<img ref="image" :src="selected.img" :alt="selected.name" @load="initCanvas()" />
	</canvas>
</template>

<style lang="scss">
.ul-public-image-map {
	width: 100%;
	max-width: 800px;
	height: auto;
	background-color: white;

	img {
		width: 100%;
		height: 100%;
	}
}
</style>
