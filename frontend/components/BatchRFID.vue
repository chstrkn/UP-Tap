<script setup lang="ts">
const toast = useToast()
const modal = useModal()
interface Props {
  /** List of student_id values to scan in sequence */
  students: Student[]
}
const props = defineProps<Props>()
const studentIds = computed(() => props.students.map((s) => s.student_id))

/** Which student in the array we're on */
const currentIndex = ref(0)

/** Buffer for the “typed” RFID from the scanner */
const buffer = ref('')

/** Expose the current student_id for display */
const currentStudentId = ref<number | null>(null)

/** Whenever the prop or index changes, update currentStudentId */
watch(
  () => [studentIds, currentIndex.value],
  () => {
    if (studentIds.value.length === 0 || currentIndex.value >= studentIds.value.length) {
      currentStudentId.value = null
      modal.close()
    } else {
      currentStudentId.value = studentIds.value[currentIndex.value]
    }
    // clear buffer for the next scan
    buffer.value = ''
  },
  { immediate: true }
)

/** Handle each key event from the scanner */
async function onKeydown(e: KeyboardEvent) {
  // digits/letters accumulate in buffer
  if (/^[0-9A-Za-z]$/.test(e.key)) {
    buffer.value += e.key
    return
  }
  // Enter triggers the POST
  if (e.key === 'Enter') {
    const rfid = buffer.value.trim()
    if (!rfid || currentStudentId.value === null) {
      buffer.value = ''
      return
    }

    // Send to your API
    await $fetch('/api/students/rfid', {
      method: 'POST',
        body: {
          student_id: currentStudentId.value,
          rfid_num: Number(rfid),
        },
    }).then((res) => {
        toast.add({
            id: 'rfid-assigned',
            color: 'green',
            icon: 'i-heroicons-check-circle',
            title: `RFID ${rfid} assigned to student ${props.students[currentIndex.value].first_name} ${props.students[currentIndex.value].last_name}`,
        })
        currentIndex.value += 1
        if (currentIndex.value >= studentIds.value.length) {
          toast.add({
            id: 'batch-complete',
            color: 'green',
            icon: 'i-heroicons-check-circle',
            title: 'RFID batch assignment complete',
          })
          modal.close()
        }
      })
      .catch((err) => {
        toast.add({
          id: 'rfid-error',
          color: 'red',
          icon: 'i-heroicons-x-circle',
          title: `Error assigning RFID`,
          description: err.data.data.detail,
        })
      }).finally(() => {
        // clear buffer for the next scan
        buffer.value = ''
      })
  }
}

onMounted(() => {
  window.addEventListener('keydown', onKeydown)
})
onBeforeUnmount(() => {
  window.removeEventListener('keydown', onKeydown)
})
</script>

<template>
    <UModal>
    <div class="space-y-2 p-4 border rounded">
        <div class="font-medium">
        <!-- Show the student we're scanning for -->
        Scan RFID for student:
        <span v-if="currentStudentId">{{ students[currentIndex].first_name }} {{ students[currentIndex].last_name }}</span>
        <span v-else>— no students defined —</span>
        </div>
        <div class="text-sm text-gray-500">
        Buffer: <code>{{ buffer }}</code>
        </div>
        <p class="text-xs text-gray-400">
        (Use your RFID reader now; it will type into the buffer and send on Enter.)
        </p>
    </div>
</UModal>
</template>
