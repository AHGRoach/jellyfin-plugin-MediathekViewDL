<script setup>
import { ref, onMounted } from 'vue'
import ApiService from '../../utils/ApiService'

const props = defineProps({
  onEdit: { type: Function, required: true }
})

const Dashboard = window.Dashboard ?? null

const subscriptions = ref([])
const loading = ref(false)
const error = ref(null)

async function fetchSubscriptions() {
  loading.value = true
  error.value = null
  try {
    subscriptions.value = await ApiService.getSubscriptions()
  } catch (e) {
    error.value = 'Failed to load subscriptions.'
    console.error('Failed to fetch subscriptions', e)
  } finally {
    loading.value = false
  }
}

async function deleteSubscription(id) {
  if (!Dashboard) return
  Dashboard.confirm('Are you sure you want to delete this subscription?', 'Confirm Delete', async (result) => {
    if (result) {
      try {
        await ApiService.deleteSubscription(id)
        await fetchSubscriptions()
        Dashboard.alert('Subscription deleted.')
      } catch (e) {
        console.error('Delete failed', e)
        Dashboard.alert('Failed to delete the subscription.')
      }
    }
  })
}

async function resetProcessedItems(id) {
  if (!Dashboard) return
  Dashboard.confirm('Are you sure you want to reset the processed-item history for this subscription?', 'Confirm Reset', async (result) => {
    if (result) {
      try {
        await ApiService.resetSubscriptionHistory(id)
        Dashboard.alert('History has been reset.')
        await fetchSubscriptions()
      } catch (e) {
        console.error('Reset failed', e)
        Dashboard.alert('Failed to reset history.')
      }
    }
  })
}

async function processSubscription(id) {
  if (!Dashboard) return
  try {
    const response = await ApiService.processSubscription(id)
    Dashboard.alert(response + ' new items found.')
  } catch (e) {
    console.error('Processing failed', e)
    Dashboard.alert('Failed to process the subscription.')
  }
}

async function toggleActive(sub) {
  const newState = !sub.IsEnabled
  try {
    const result = await ApiService.setSubscriptionActive(sub.Id, newState)
    sub.IsEnabled = result === true || result === 'true'
  } catch (e) {
    console.error('Toggle failed', e)
    if (Dashboard) Dashboard.alert('Failed to change subscription status.')
  }
}

async function triggerDownloads() {
  if (!Dashboard) return
  loading.value = true
  try {
    const tasks = await ApiService.getScheduledTasks()
    const task = tasks.find(t => t.Key === 'MediathekViewDL-MediathekAboDownloader')

    if (!task) {
      Dashboard.alert('Scheduled task "Mediathek Subscription Downloader" was not found.')
      return
    }

    if (task.State !== 'Idle') {
      Dashboard.alert('The subscription downloader is already running.')
      return
    }

    await ApiService.startScheduledTask(task.Id)

    Dashboard.alert('Download task started.')
  } catch (e) {
    console.error('Failed to trigger downloads', e)
    Dashboard.alert('Failed to start the download task.')
  } finally {
    loading.value = false
  }
}

onMounted(() => {
  fetchSubscriptions()
})

// Expose refresh to parent if needed
defineExpose({ refresh: fetchSubscriptions })
</script>

<template>
  <div class="card">
    <div class="header-row">
      <h2>Subscription Management</h2>
      <div class="header-actions">
        <button class="btn btn-secondary" @click="triggerDownloads" :disabled="loading">Start Downloads Manually</button>
        <button class="btn btn-primary" @click="onEdit()" :disabled="loading">New Subscription</button>
      </div>
    </div>

    <div v-if="loading" class="state-msg">
      <div class="spinner"></div>
      Loading subscriptions...
    </div>

    <div v-else-if="error" class="error-container">
      <div class="error-msg">{{ error }}</div>
      <button @click="fetchSubscriptions" class="btn btn-secondary">Try Again</button>
    </div>

    <div v-else-if="subscriptions.length > 0" class="subscriptions-list">
      <div v-for="sub in subscriptions" :key="sub.Id" class="subscription-item" :class="{ disabled: !sub.IsEnabled }">
        <div class="sub-left">
          <label class="switch" title="Enable/disable subscription">
            <input type="checkbox" :checked="sub.IsEnabled" @change="toggleActive(sub)">
            <span class="slider round"></span>
          </label>

          <div class="sub-info">
            <div class="sub-name">
              {{ sub.Name }}
            </div>
            <div class="sub-meta">
              Last Download: {{ sub.LastDownloadedTimestamp ? new Date(sub.LastDownloadedTimestamp).toLocaleString() : 'Never' }}
            </div>
          </div>
        </div>
        <div class="sub-actions">
          <button @click="resetProcessedItems(sub.Id)" class="btn-icon" title="Reset history">↩️</button>
          <button @click="processSubscription(sub.Id)" class="btn-icon" title="Process now">🔄</button>
          <button @click="onEdit(sub)" class="btn-icon" title="Edit">✏️</button>
          <button @click="deleteSubscription(sub.Id)" class="btn-icon btn-delete" title="Delete">🗑️</button>
        </div>
      </div>
    </div>

    <div v-else class="no-data">
      No subscriptions configured.
    </div>
  </div>
</template>

<style scoped>
.header-row { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; }
.header-actions { display: flex; gap: 10px; }
.subscriptions-list { display: grid; gap: 10px; }
.subscription-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px;
}
.subscription-item.disabled { opacity: 0.6; border-style: dashed; }
.sub-left { display: flex; align-items: center; gap: 20px; }
.sub-name { font-weight: bold; font-size: 1.1rem; display: flex; align-items: center; gap: 10px; }
.sub-meta { font-size: 0.85rem; color: #a1a1aa; margin-top: 4px; }
.sub-actions { display: flex; gap: 15px; }
.btn-icon { background: none; border: none; cursor: pointer; font-size: 1.4rem; padding: 5px; border-radius: 4px; filter: grayscale(1); color: white; }
.btn-icon:hover { background: #3f3f46; filter: none; }
.btn-delete:hover { color: #ef4444; }
.state-msg { text-align: center; padding: 40px; color: #a1a1aa; }
.error-container { text-align: center; padding: 30px; background: rgba(239, 68, 68, 0.1); border: 1px solid #ef4444; border-radius: 8px; color: #ef4444; }
.error-msg { margin-bottom: 10px; font-weight: bold; }
.no-data { text-align: center; color: #a1a1aa; padding: 40px; }

/* Switch Toggle Styles */
.switch {
  position: relative;
  display: inline-block;
  width: 44px;
  height: 24px;
}
.switch input { opacity: 0; width: 0; height: 0; }
.slider {
  position: absolute;
  cursor: pointer;
  top: 0; left: 0; right: 0; bottom: 0;
  background-color: #3f3f46;
  transition: .4s;
}
.slider:before {
  position: absolute;
  content: "";
  height: 18px; width: 18px;
  left: 3px; bottom: 3px;
  background-color: white;
  transition: .4s;
}
input:checked + .slider { background-color: #7c3aed; }
input:focus + .slider { box-shadow: 0 0 1px #7c3aed; }
input:checked + .slider:before { transform: translateX(20px); }
.slider.round { border-radius: 24px; }
.slider.round:before { border-radius: 50%; }

</style>
