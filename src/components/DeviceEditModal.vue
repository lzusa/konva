<script setup>
import { computed, reactive, ref, watch } from 'vue'

const props = defineProps({
  open: {
    type: Boolean,
    default: false
  },
  device: {
    type: Object,
    default: null
  },
  existingIds: {
    type: Array,
    default: () => []
  }
})

const emit = defineEmits(['close', 'save', 'delete'])

const draft = reactive({
  id: '',
  x: 0,
  y: 0,
  width: 0,
  height: 0
})

const originalId = ref('')
const error = ref('')

const isOpen = computed(() => props.open && props.device)

watch(
  () => props.device,
  (device) => {
    if (!device) {
      return
    }
    draft.id = device.id
    draft.x = device.x
    draft.y = device.y
    draft.width = device.width
    draft.height = device.height
    originalId.value = device.id
    error.value = ''
  },
  { immediate: true }
)

const validate = () => {
  if (!draft.id.trim()) {
    return 'Device ID is required.'
  }

  const isDuplicate =
    props.existingIds.includes(draft.id.trim()) &&
    draft.id.trim() !== originalId.value
  if (isDuplicate) {
    return 'Device ID already exists.'
  }

  const numericFields = ['x', 'y', 'width', 'height']
  for (const field of numericFields) {
    const value = Number(draft[field])
    if (Number.isNaN(value)) {
      return 'Coordinates and size must be valid numbers.'
    }
  }

  if (Number(draft.x) < 0 || Number(draft.y) < 0) {
    return 'Coordinates must be zero or positive.'
  }
  if (Number(draft.width) <= 0 || Number(draft.height) <= 0) {
    return 'Width and height must be greater than 0.'
  }
  if (Number(draft.x) + Number(draft.width) > 800000) {
    return 'Device exceeds canvas width (800000).'
  }
  if (Number(draft.y) + Number(draft.height) > 200000) {
    return 'Device exceeds canvas height (200000).'
  }

  return ''
}

const save = () => {
  const validationError = validate()
  error.value = validationError
  if (validationError) {
    return
  }

  emit('save', {
    id: draft.id.trim(),
    x: Number(draft.x),
    y: Number(draft.y),
    width: Number(draft.width),
    height: Number(draft.height)
  })
}
</script>

<template>
  <div v-if="isOpen" class="modal">
    <div class="modal__backdrop" @click="emit('close')"></div>
    <div class="modal__panel" role="dialog" aria-modal="true">
      <header class="modal__header">
        <div>
          <p class="eyebrow">Edit Equipment</p>
          <h3>{{ draft.id }}</h3>
        </div>
        <button class="icon-button" type="button" @click="emit('close')">
          Close
        </button>
      </header>

      <div class="modal__body">
        <label class="form__field">
          <span>Device ID</span>
          <input v-model="draft.id" type="text" />
        </label>

        <div class="form__row">
          <label class="form__field">
            <span>X</span>
            <input v-model.number="draft.x" type="number" min="0" step="any" />
          </label>
          <label class="form__field">
            <span>Y</span>
            <input v-model.number="draft.y" type="number" min="0" step="any" />
          </label>
        </div>

        <div class="form__row">
          <label class="form__field">
            <span>Width</span>
            <input v-model.number="draft.width" type="number" min="0.1" step="any" />
          </label>
          <label class="form__field">
            <span>Height</span>
            <input v-model.number="draft.height" type="number" min="0.1" step="any" />
          </label>
        </div>

        <p v-if="error" class="form__error">{{ error }}</p>

        <!-- 只读信息区 -->
        <div v-if="device && device.info" class="info-section">
          <h4 class="info-section__title">Macro Info</h4>
          <div class="info-section__grid">
            <div
              v-for="(value, key) in device.info"
              :key="key"
              class="info-section__item"
            >
              <span class="info-section__label">{{ key }}</span>
              <span class="info-section__value">{{ value }}</span>
            </div>
          </div>
        </div>
      </div>

      <footer class="modal__footer">
        <button
          class="button button--ghost button--danger"
          type="button"
          @click="emit('delete')"
        >
          Delete
        </button>
        <button class="button button--ghost" type="button" @click="emit('close')">
          Cancel
        </button>
        <button class="button" type="button" @click="save">Save changes</button>
      </footer>
    </div>
  </div>
</template>
