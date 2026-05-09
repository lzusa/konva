<script setup>
import { reactive, ref } from 'vue'

const props = defineProps({
  existingIds: {
    type: Array,
    default: () => []
  }
})

const emit = defineEmits(['add'])

const form = reactive({
  id: '',
  x: 0,
  y: 0,
  width: 5000,
  height: 3000
})

const error = ref('')

const resetForm = () => {
  form.id = ''
  form.x = 0
  form.y = 0
  form.width = 5000
  form.height = 3000
}

const validate = () => {
  if (!form.id.trim()) {
    return 'Device ID is required.'
  }
  if (props.existingIds.includes(form.id.trim())) {
    return 'Device ID already exists.'
  }

  const numericFields = ['x', 'y', 'width', 'height']
  for (const field of numericFields) {
    const value = Number(form[field])
    if (Number.isNaN(value)) {
      return 'Coordinates and size must be valid numbers.'
    }
  }

  if (Number(form.x) < 0 || Number(form.y) < 0) {
    return 'Coordinates must be zero or positive.'
  }
  if (Number(form.width) <= 0 || Number(form.height) <= 0) {
    return 'Width and height must be greater than 0.'
  }

  return ''
}

const submit = () => {
  const validationError = validate()
  error.value = validationError
  if (validationError) {
    return
  }

  emit('add', {
    id: form.id.trim(),
    x: Number(form.x),
    y: Number(form.y),
    width: Number(form.width),
    height: Number(form.height)
  })

  resetForm()
}
</script>

<template>
  <form class="form" @submit.prevent="submit">
    <label class="form__field">
      <span>Device ID</span>
      <input v-model="form.id" type="text" placeholder="EQ-2001" />
    </label>

    <div class="form__row">
      <label class="form__field">
        <span>X</span>
        <input v-model.number="form.x" type="number" min="0" step="any" placeholder="如 1000.5" />
      </label>
      <label class="form__field">
        <span>Y</span>
        <input v-model.number="form.y" type="number" min="0" step="any" placeholder="如 500.25" />
      </label>
    </div>

    <div class="form__row">
      <label class="form__field">
        <span>Width</span>
        <input v-model.number="form.width" type="number" min="0.1" step="any" placeholder="如 5000.5" />
      </label>
      <label class="form__field">
        <span>Height</span>
        <input v-model.number="form.height" type="number" min="0.1" step="any" placeholder="如 3000.25" />
      </label>
    </div>

    <p v-if="error" class="form__error">{{ error }}</p>

    <button class="button" type="submit">Add Equipment</button>
  </form>
</template>
