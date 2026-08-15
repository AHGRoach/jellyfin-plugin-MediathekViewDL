<script setup>
import { ref, computed, onMounted } from 'vue'
import ApiService from "../../utils/ApiService.js";

const Dashboard = window.Dashboard ?? null
const PLUGIN_ID = 'a31b415a-5264-419d-b152-8c8192a54994'
const PLUGIN_NAME = 'MediathekViewDL'

const emit = defineEmits(['config-saved'])

// --- State ---
const loading = ref(false)
const saving = ref(false)
const lastRun = ref(null)

// Paths
const useTopicForMoviePath = ref(false)
const defaultDownloadPath = ref('')
const subscriptionShowPath = ref('')
const subscriptionMoviePath = ref('')
const manualShowPath = ref('')
const manualMoviePath = ref('')
const tempDownloadPath = ref('')

// Download
const downloadSubtitles = ref(false)
const readRate = ref(0)
const scanLibraryAfterDownload = ref(true)
const minFreeDiskSpaceMiB = ref('')

// Search
const fetchStreamSizes = ref(false)
const searchInFutureBroadcasts = ref(false)
const searchPageSize = ref(50)
const searchMaxPages = ref(5)

// Network
const allowUnknownDomains = ref(false)
const allowHttp = ref(false)

// Subscription Defaults - Search
const defMinDuration = ref('')
const defMaxDuration = ref('')

// Subscription Defaults - Download
const defUseStreamingUrlFiles = ref(false)
const defDownloadFullVideoSecondaryAudio = ref(false)
const defAlwaysCreateSubfolder = ref(false)
const defEnhancedDuplicateDetection = ref(false)
const defAllowFallbackToLowerQuality = ref(true)
const defQualityCheckWithUrl = ref(false)

// Subscription Defaults - Series
const defEnforceSeries = ref(false)
const defAllowAbsoluteEpisodeNumbering = ref(false)
const defTreatNonEpisodesAsExtras = ref(false)
const defSaveExtrasAsStrm = ref(false)
const defSaveTrailers = ref(true)
const defSaveInterviews = ref(true)
const defSaveGenericExtras = ref(true)

// Subscription Defaults - Metadata
const defOriginalLanguage = ref('')
const defCreateNfo = ref(false)
const defAppendDateToTitle = ref(false)
const defKeepOriginalTitle = ref(false)
const defAppendTimeToTitle = ref(false)

// Subscription Defaults - Accessibility
const defAllowAudioDesc = ref(false)
const defAllowSignLanguage = ref(false)

// Maintenance
const enableStrmCleanup = ref(false)
const allowDownloadOnUnknownDiskSpace = ref(false)

// Computed
const searchTotalItems = computed(() => {
  const ps = parseInt(searchPageSize.value) || 0
  const mp = parseInt(searchMaxPages.value) || 0
  return ps * mp
})

// --- API ---
async function loadConfig() {
  loading.value = true
  try {
    const config = await ApiService.getPluginConfig(PLUGIN_ID)

    lastRun.value = config.LastRun ? new Date(config.LastRun).toLocaleString() : 'Never'

    // Paths
    useTopicForMoviePath.value = config.Paths?.UseTopicForMoviePath ?? false
    defaultDownloadPath.value = config.Paths?.DefaultDownloadPath ?? ''
    subscriptionShowPath.value = config.Paths?.DefaultSubscriptionShowPath ?? ''
    subscriptionMoviePath.value = config.Paths?.DefaultSubscriptionMoviePath ?? ''
    manualShowPath.value = config.Paths?.DefaultManualShowPath ?? ''
    manualMoviePath.value = config.Paths?.DefaultManualMoviePath ?? ''
    tempDownloadPath.value = config.Paths?.TempDownloadPath ?? ''

    // Download
    downloadSubtitles.value = config.Download?.DownloadSubtitles ?? false
    readRate.value = config.Download?.ReadRate ?? 0
    scanLibraryAfterDownload.value = config.Download?.ScanLibraryAfterDownload ?? true
    minFreeDiskSpaceMiB.value = config.Download?.MinFreeDiskSpaceBytes
      ? (config.Download.MinFreeDiskSpaceBytes / (1024 * 1024)).toString()
      : ''

    // Search
    fetchStreamSizes.value = config.Search?.FetchStreamSizes ?? false
    searchInFutureBroadcasts.value = config.Search?.SearchInFutureBroadcasts ?? false
    searchPageSize.value = config.Search?.PageSize ?? 50
    searchMaxPages.value = config.Search?.MaxPages ?? 5

    // Network
    allowUnknownDomains.value = config.Network?.AllowUnknownDomains ?? false
    allowHttp.value = config.Network?.AllowHttp ?? false

    // Maintenance
    enableStrmCleanup.value = config.Maintenance?.EnableStrmCleanup ?? false
    allowDownloadOnUnknownDiskSpace.value = config.Maintenance?.AllowDownloadOnUnknownDiskSpace ?? false

    // Subscription Defaults
    const def = config.SubscriptionDefaults ?? {}
    const defDl = def.DownloadSettings ?? {}
    const defSearch = def.SearchSettings ?? {}
    const defSeries = def.SeriesSettings ?? {}
    const defMeta = def.MetadataSettings ?? {}
    const defAccess = def.AccessibilitySettings ?? {}

    defMinDuration.value = defSearch.MinDurationMinutes ?? ''
    defMaxDuration.value = defSearch.MaxDurationMinutes ?? ''

    defUseStreamingUrlFiles.value = defDl.UseStreamingUrlFiles ?? false
    defDownloadFullVideoSecondaryAudio.value = defDl.DownloadFullVideoForSecondaryAudio ?? false
    defAlwaysCreateSubfolder.value = defDl.AlwaysCreateSubfolder ?? false
    defEnhancedDuplicateDetection.value = defDl.EnhancedDuplicateDetection ?? false
    defAllowFallbackToLowerQuality.value = defDl.AllowFallbackToLowerQuality !== undefined ? defDl.AllowFallbackToLowerQuality : true
    defQualityCheckWithUrl.value = defDl.QualityCheckWithUrl ?? false

    defEnforceSeries.value = defSeries.EnforceSeriesParsing ?? false
    defAllowAbsoluteEpisodeNumbering.value = defSeries.AllowAbsoluteEpisodeNumbering ?? false
    defTreatNonEpisodesAsExtras.value = defSeries.TreatNonEpisodesAsExtras ?? false
    defSaveExtrasAsStrm.value = defSeries.SaveExtrasAsStrm ?? false
    defSaveTrailers.value = defSeries.SaveTrailers !== undefined ? defSeries.SaveTrailers : true
    defSaveInterviews.value = defSeries.SaveInterviews !== undefined ? defSeries.SaveInterviews : true
    defSaveGenericExtras.value = defSeries.SaveGenericExtras !== undefined ? defSeries.SaveGenericExtras : true

    defOriginalLanguage.value = defMeta.OriginalLanguage ?? ''
    defCreateNfo.value = defMeta.CreateNfo ?? false
    defAppendDateToTitle.value = defMeta.AppendDateToTitle ?? false
    defKeepOriginalTitle.value = defMeta.KeepOriginalTitle ?? false
    defAppendTimeToTitle.value = defMeta.AppendTimeToTitle ?? false

    defAllowAudioDesc.value = defAccess.AllowAudioDescription ?? false
    defAllowSignLanguage.value = defAccess.AllowSignLanguage ?? false

  } catch (e) {
    console.error('Failed to load config', e)
  } finally {
    loading.value = false
  }
}

async function saveConfig() {
  saving.value = true
  try {
    const config = await ApiService.getPluginConfig(PLUGIN_ID)

    // Paths
    if (!config.Paths) config.Paths = {}
    config.Paths.UseTopicForMoviePath = useTopicForMoviePath.value
    config.Paths.DefaultDownloadPath = defaultDownloadPath.value
    config.Paths.DefaultSubscriptionShowPath = subscriptionShowPath.value
    config.Paths.DefaultSubscriptionMoviePath = subscriptionMoviePath.value
    config.Paths.DefaultManualShowPath = manualShowPath.value
    config.Paths.DefaultManualMoviePath = manualMoviePath.value
    config.Paths.TempDownloadPath = tempDownloadPath.value

    // Download
    if (!config.Download) config.Download = {}
    config.Download.DownloadSubtitles = downloadSubtitles.value
    config.Download.ScanLibraryAfterDownload = scanLibraryAfterDownload.value
    config.Download.ReadRate = readRate.value
    const minFree = parseInt(minFreeDiskSpaceMiB.value)
    config.Download.MinFreeDiskSpaceBytes = isNaN(minFree) ? (1.5 * 1024 * 1024 * 1024) : (minFree * 1024 * 1024)

    // Search
    if (!config.Search) config.Search = {}
    config.Search.FetchStreamSizes = fetchStreamSizes.value
    config.Search.SearchInFutureBroadcasts = searchInFutureBroadcasts.value
    config.Search.PageSize = parseInt(searchPageSize.value) || 50
    config.Search.MaxPages = parseInt(searchMaxPages.value) || 5

    // Network
    if (!config.Network) config.Network = {}
    config.Network.AllowUnknownDomains = allowUnknownDomains.value
    config.Network.AllowHttp = allowHttp.value

    // Maintenance
    if (!config.Maintenance) config.Maintenance = {}
    config.Maintenance.EnableStrmCleanup = enableStrmCleanup.value
    config.Maintenance.AllowDownloadOnUnknownDiskSpace = allowDownloadOnUnknownDiskSpace.value

    // Subscription Defaults
    config.SubscriptionDefaults = {
      SearchSettings: {
        MinDurationMinutes: defMinDuration.value !== '' ? parseInt(defMinDuration.value) : null,
        MaxDurationMinutes: defMaxDuration.value !== '' ? parseInt(defMaxDuration.value) : null
      },
      DownloadSettings: {
        UseStreamingUrlFiles: defUseStreamingUrlFiles.value,
        DownloadFullVideoForSecondaryAudio: defDownloadFullVideoSecondaryAudio.value,
        AlwaysCreateSubfolder: defAlwaysCreateSubfolder.value,
        EnhancedDuplicateDetection: defEnhancedDuplicateDetection.value,
        AllowFallbackToLowerQuality: defAllowFallbackToLowerQuality.value,
        QualityCheckWithUrl: defQualityCheckWithUrl.value
      },
      SeriesSettings: {
        EnforceSeriesParsing: defEnforceSeries.value,
        AllowAbsoluteEpisodeNumbering: defAllowAbsoluteEpisodeNumbering.value,
        TreatNonEpisodesAsExtras: defTreatNonEpisodesAsExtras.value,
        SaveExtrasAsStrm: defSaveExtrasAsStrm.value,
        SaveTrailers: defSaveTrailers.value,
        SaveInterviews: defSaveInterviews.value,
        SaveGenericExtras: defSaveGenericExtras.value
      },
      MetadataSettings: {
        OriginalLanguage: defOriginalLanguage.value,
        CreateNfo: defCreateNfo.value,
        AppendDateToTitle: defAppendDateToTitle.value,
        KeepOriginalTitle: defKeepOriginalTitle.value,
        AppendTimeToTitle: defAppendTimeToTitle.value
      },
      AccessibilitySettings: {
        AllowAudioDescription: defAllowAudioDesc.value,
        AllowSignLanguage: defAllowSignLanguage.value
      }
    }

     await ApiService.updatePluginConfig(PLUGIN_ID, config)
     if (Dashboard) Dashboard.alert('Settings saved.')
     // Notify parent to refresh config
     emit('config-saved')
  } catch (e) {
    console.error('Failed to save config', e)
    if (Dashboard) Dashboard.alert('Failed to save settings.')
  } finally {
    saving.value = false
  }
}

async function copyConfig() {
  try {
    const config = await ApiService.getPluginConfig(PLUGIN_ID)
    const text = JSON.stringify(config, null, 2)
    if (window.isSecureContext) {
      await navigator.clipboard.writeText(text)
      if (Dashboard) Dashboard.alert('Configuration copied to the clipboard.')
    } else {
      prompt('Copy manually:', text)
    }
  } catch (e) {
    console.error('Failed to copy config', e)
  }
}

function selectPath(targetRef, header) {
  if (!Dashboard) return
  try {
    const picker = new Dashboard.DirectoryBrowser()
    picker.show({
      header: header,
      includeDirectories: true,
      includeFiles: false,
      callback: (path) => {
        if (path) targetRef.value = path
        picker.close()
      }
    })
  } catch (e) {
    const newPath = prompt(header + '\nCurrent path: ' + targetRef.value, targetRef.value)
    if (newPath !== null && newPath.trim() !== '') targetRef.value = newPath.trim()
  }
}

async function setupTuner() {
  try {
    if (Dashboard) Dashboard.showLoadingMsg()
    await ApiService.addTunerHost({ Type: 'zapp', Url: 'zapp', FriendlyName: 'Zapp (MediathekView)', TunerCount: 0 })
    if (Dashboard) { Dashboard.hideLoadingMsg(); Dashboard.alert('Zapp tuner added successfully.') }
  } catch (e) {
    if (Dashboard) Dashboard.hideLoadingMsg()
    console.error('Error adding tuner', e)
    if (Dashboard) Dashboard.alert('Failed to add the tuner.')
  }
}

async function setupGuide() {
  try {
    if (Dashboard) Dashboard.showLoadingMsg()
    await ApiService.addListingProvider({ Type: 'zapp', Id: 'zapp_guide', Name: 'Zapp (MediathekView)' })
    if (Dashboard) { Dashboard.hideLoadingMsg(); Dashboard.alert('Zapp guide provider added successfully.') }
  } catch (e) {
    if (Dashboard) Dashboard.hideLoadingMsg()
    console.error('Error adding guide', e)
    if (Dashboard) Dashboard.alert('Failed to add the guide provider.')
  }
}

onMounted(() => {
  loadConfig()
})
</script>

<template>
  <div class="card settings-card">
    <div v-if="loading" class="state-msg"><div class="spinner"></div> Loading settings...</div>

    <form v-else @submit.prevent="saveConfig">

      <!-- ===== ALLGEMEIN (hidden, no active options) ===== -->
      <details hidden class="settings-section">
        <summary class="section-title">General Settings</summary>
        <div class="section-body"></div>
      </details>

      <!-- ===== PFADE ===== -->
      <details class="settings-section">
        <summary class="section-title">Path Settings</summary>
        <div class="section-body">
          <div class="checkbox-field">
            <label>
              <input v-model="useTopicForMoviePath" type="checkbox"> Create a topic folder for movie downloads
            </label>
            <p class="field-desc">
              For movie downloads, create a folder for the topic (or subscription name for subscriptions).<br>
              e.g. when enabled: /Filme/Filme im Ersten/Filmname/Filmname.mkv<br>
              when disabled: /Filme/Filmname/Filmname.mkv
            </p>
          </div>

          <div class="field">
            <label class="field-label">Global Default Download Path <span class="badge-deprecated">Deprecated</span></label>
            <div class="path-row">
              <input v-model="defaultDownloadPath" type="text" class="field-input" placeholder="Leave blank to use default">
              <button type="button" class="btn btn-secondary btn-sm" @click="selectPath(defaultDownloadPath, 'Select Global Default Download Path')" title="Select folder">📁</button>
            </div>
            <p class="field-desc">The global fallback folder for all downloads. <span style="color:#f87171;font-weight:bold;">Note: This path is deprecated and will be removed in a future version.</span></p>
          </div>

          <div class="field">
            <label class="field-label">Default Series Path (Subscriptions)</label>
            <div class="path-row">
              <input v-model="subscriptionShowPath" type="text" class="field-input" placeholder="Leave blank to use default">
              <button type="button" class="btn btn-secondary btn-sm" @click="selectPath(subscriptionShowPath, 'Select Default Series Path (Subscriptions)')" title="Select folder">📁</button>
            </div>
            <p class="field-desc">Default folder for series downloads from subscriptions.</p>
          </div>

          <div class="field">
            <label class="field-label">Default Movie Path (Subscriptions)</label>
            <div class="path-row">
              <input v-model="subscriptionMoviePath" type="text" class="field-input" placeholder="Leave blank to use default">
              <button type="button" class="btn btn-secondary btn-sm" @click="selectPath(subscriptionMoviePath, 'Select Default Movie Path (Subscriptions)')" title="Select folder">📁</button>
            </div>
            <p class="field-desc">Default folder for movie downloads from subscriptions.</p>
          </div>

          <div class="field">
            <label class="field-label">Default Series Path (Manual)</label>
            <div class="path-row">
              <input v-model="manualShowPath" type="text" class="field-input" placeholder="Leave blank to use default">
              <button type="button" class="btn btn-secondary btn-sm" @click="selectPath(manualShowPath, 'Select Default Series Path (Manual)')" title="Select folder">📁</button>
            </div>
            <p class="field-desc">Default folder for manual series downloads.</p>
          </div>

          <div class="field">
            <label class="field-label">Default Movie Path (Manual)</label>
            <div class="path-row">
              <input v-model="manualMoviePath" type="text" class="field-input" placeholder="Leave blank to use default">
              <button type="button" class="btn btn-secondary btn-sm" @click="selectPath(manualMoviePath, 'Select Default Movie Path (Manual)')" title="Select folder">📁</button>
            </div>
            <p class="field-desc">Default folder for manual movie downloads.</p>
          </div>

          <div class="field">
            <label class="field-label">Temporary Download Path</label>
            <div class="path-row">
              <input v-model="tempDownloadPath" type="text" class="field-input" placeholder="Leave blank for direct download">
              <button type="button" class="btn btn-secondary btn-sm" @click="selectPath(tempDownloadPath, 'Select Temporary Download Path')" title="Select folder">📁</button>
            </div>
            <p class="field-desc">An optional folder used to temporarily store downloads. Leave blank to download directly to the destination folder.</p>
          </div>
        </div>
      </details>

      <!-- ===== DOWNLOAD ===== -->
      <details class="settings-section">
        <summary class="section-title">Download Settings</summary>
        <div class="section-body">
          <div class="checkbox-field">
            <label><input v-model="downloadSubtitles" type="checkbox"> Download subtitles (when available)</label>
            <p class="field-desc">Downloads a separate VTT or TTML subtitle file when provided by the broadcaster.</p>
          </div>

          <div class="checkbox-field">
            <label><input v-model="scanLibraryAfterDownload" type="checkbox"> Scan Library After Download</label>
            <p class="field-desc">Automatically scans the media library after new content is downloaded.</p>
          </div>
          <div class="grid-2">
            <div class="field">
              <label class="field-label">Minimum Free Disk Space (MiB)</label>
              <input v-model="minFreeDiskSpaceMiB" type="number" class="field-input" placeholder="e.g. 1536">
              <p class="field-desc">Minimum free disk space required to start a new download.</p>
            </div>
            <div class="field">
              <label class="field-label">Download Speed</label>
              <input v-model="readRate" type="number" class="field-input" min="0" step="0.1" placeholder="0 = unlimited">
              <p class="field-desc">FFmpeg readrate: 1 = real time, 2 = double speed, 0 = unlimited.</p>
            </div>
          </div>
        </div>
      </details>

      <!-- ===== SUCHE ===== -->
      <details class="settings-section">
        <summary class="section-title">Search Settings</summary>
        <div class="section-body">
          <div class="checkbox-field">
            <label><input v-model="fetchStreamSizes" type="checkbox"> Fetch Stream Size</label>
            <p class="field-desc">When enabled, video file sizes are retrieved during searches. This significantly slows down searching.</p>
          </div>
          <div class="checkbox-field">
            <label><input v-model="searchInFutureBroadcasts" type="checkbox"> Search Future Broadcasts</label>
            <p class="field-desc">Sometimes videos are available in the media library before their scheduled TV broadcast.</p>
          </div>
          <div class="grid-2">
            <div class="field">
              <label class="field-label">Page Size (API Requests)</label>
              <input v-model="searchPageSize" type="number" class="field-input" min="1" max="100">
              <p class="field-desc">Number of results requested per page from the API.</p>
            </div>
            <div class="field">
              <label class="field-label">Maximum Number of Pages</label>
              <input v-model="searchMaxPages" type="number" class="field-input" min="1" max="100">
              <p class="field-desc">Maximum number of pages per search/subscription run.</p>
            </div>
          </div>
          <div v-if="searchTotalItems > 0" class="info-msg">
            Current configuration: Up to <strong>{{ searchTotalItems }}</strong> media items can be found per search/subscription run.
          </div>
        </div>
      </details>

      <!-- ===== NETZWERK & SICHERHEIT ===== -->
      <details class="settings-section">
        <summary class="section-title">Network &amp; Security</summary>
        <div class="section-body">
          <div class="checkbox-field">
            <label><input v-model="allowUnknownDomains" type="checkbox"> Allow Downloads from Unknown Domains</label>
            <p class="field-desc">Allows downloading content from domains that are not on the whitelist. This can be useful if ARD or ZDF add new CDNs. <strong>Security risk — use with caution.</strong></p>
          </div>
          <div class="checkbox-field">
            <label><input v-model="allowHttp" type="checkbox"> Allow HTTP Downloads</label>
            <p class="field-desc">This may be necessary because some URLs do not support HTTPS. It is recommended to leave this disabled.</p>
          </div>
        </div>
      </details>

      <!-- ===== ABO-STANDARDWERTE ===== -->
      <details class="settings-section">
        <summary class="section-title">Subscription Defaults</summary>
        <div class="section-body">
          <p class="field-desc" style="margin-bottom:15px;">These values are used as defaults for new subscriptions.</p>

          <div class="sub-section-title">Search</div>
          <div class="grid-2">
            <div class="field">
              <label class="field-label">Min. Duration (minutes)</label>
              <input v-model="defMinDuration" type="number" class="field-input" placeholder="No limit">
            </div>
            <div class="field">
              <label class="field-label">Max. Duration (minutes)</label>
              <input v-model="defMaxDuration" type="number" class="field-input" placeholder="No limit">
            </div>
          </div>

          <div class="sub-section-title">Download</div>
          <div class="checkbox-field">
            <label><input v-model="defUseStreamingUrlFiles" type="checkbox"> Use streaming URL files (.strm)</label>
          </div>
          <div v-if="!defUseStreamingUrlFiles" class="sub-options">
            <div class="checkbox-field">
              <label><input v-model="defDownloadFullVideoSecondaryAudio" type="checkbox"> Full video for secondary audio languages</label>
            </div>
          </div>
          <div class="checkbox-field">
            <label><input v-model="defAlwaysCreateSubfolder" type="checkbox"> Create subfolder for subscription</label>
          </div>
          <div class="checkbox-field">
            <label><input v-model="defEnhancedDuplicateDetection" type="checkbox"> Enhanced Duplicate Detection</label>
          </div>
          <div class="checkbox-field">
            <label><input v-model="defAllowFallbackToLowerQuality" type="checkbox"> Allow fallback to lower quality</label>
          </div>
          <div v-if="defAllowFallbackToLowerQuality" class="sub-options">
            <div class="checkbox-field">
              <label><input v-model="defQualityCheckWithUrl" type="checkbox"> Validate URL</label>
            </div>
          </div>

          <div class="sub-section-title">Series</div>
          <div class="checkbox-field">
            <label><input v-model="defEnforceSeries" type="checkbox"> Download series only</label>
          </div>
          <div v-if="defEnforceSeries" class="sub-options">
            <div class="checkbox-field">
              <label><input v-model="defAllowAbsoluteEpisodeNumbering" type="checkbox"> Allow absolute episode numbering</label>
            </div>
          </div>
          <div v-if="!defEnforceSeries">
            <div class="checkbox-field">
              <label><input v-model="defTreatNonEpisodesAsExtras" type="checkbox"> Treat non-episodes as extras</label>
            </div>
            <div v-if="defTreatNonEpisodesAsExtras" class="sub-options">
              <div class="checkbox-field">
                <label><input v-model="defSaveExtrasAsStrm" type="checkbox"> Save extras as streams (.strm)</label>
              </div>
              <div class="checkbox-field">
                <label><input v-model="defSaveTrailers" type="checkbox"> Save trailers</label>
              </div>
              <div class="checkbox-field">
                <label><input v-model="defSaveInterviews" type="checkbox"> Save interviews</label>
              </div>
              <div class="checkbox-field">
                <label><input v-model="defSaveGenericExtras" type="checkbox"> Save generic extras</label>
              </div>
            </div>
          </div>

          <div class="sub-section-title">Metadata</div>
          <div class="field">
            <label class="field-label">Original Language (ISO code, e.g. 'eng')</label>
            <input v-model="defOriginalLanguage" type="text" class="field-input" placeholder="e.g. eng or fra">
          </div>
          <div class="checkbox-field">
            <label><input v-model="defCreateNfo" type="checkbox"> Create NFO files</label>
          </div>
          <div class="checkbox-field">
            <label><input v-model="defAppendDateToTitle" type="checkbox"> Append date to title</label>
          </div>
          <div v-if="defAppendDateToTitle" class="sub-options">
            <div class="checkbox-field">
              <label><input v-model="defAppendTimeToTitle" type="checkbox"> Append time to title</label>
            </div>
          </div>
          <div class="checkbox-field">
            <label><input v-model="defKeepOriginalTitle" type="checkbox"> Keep original title</label>
          </div>

          <div class="sub-section-title">Accessibility</div>
          <div class="checkbox-field">
            <label><input v-model="defAllowAudioDesc" type="checkbox"> Allow audio description</label>
          </div>
          <div class="checkbox-field">
            <label><input v-model="defAllowSignLanguage" type="checkbox"> Allow sign language</label>
          </div>
        </div>
      </details>

      <!-- ===== WARTUNG ===== -->
      <details class="settings-section">
        <summary class="section-title">Maintenance</summary>
        <div class="section-body">
          <div class="checkbox-field">
            <label><input v-model="enableStrmCleanup" type="checkbox"> Enable cleanup of invalid streaming files (.strm)</label>
            <p class="field-desc">Regularly validates links in all generated .strm files. If a link is no longer available (e.g. 404), the file is deleted.</p>
          </div>
          <div class="checkbox-field">
            <label><input v-model="allowDownloadOnUnknownDiskSpace" type="checkbox"> Allow downloads when disk space is unknown</label>
            <p class="field-desc">Allows downloads even when available disk space cannot be determined (for example, on some network shares).</p>
          </div>
        </div>
      </details>

      <!-- ===== LIVE TV ===== -->
      <details class="settings-section">
        <summary class="section-title">Live TV Integration</summary>
        <div class="section-body">
          <p class="field-desc">Configure the Live TV integration here. Because Jellyfin does not automatically expose plugins as guide providers, you can add them manually here.</p>
          <div class="btn-row">
            <button type="button" class="btn btn-secondary" @click="setupTuner">Add Zapp Tuner</button>
            <button type="button" class="btn btn-secondary" @click="setupGuide">Add Zapp Guide Provider</button>
          </div>
          <p class="field-desc" style="margin-top:10px;"><strong>Note:</strong> After adding these providers, you may need to reload the page or run Jellyfin's Guide Refresh task before the data appears.</p>
        </div>
      </details>

      <!-- ===== LETZTER LAUF + BUTTONS ===== -->
      <div class="footer-row">
        <div class="last-run">
          <span class="field-label">Last Run:</span>
          <span>{{ lastRun ?? 'Never' }}</span>
        </div>
        <div class="action-row">
          <button type="button" class="btn btn-secondary btn-sm" @click="copyConfig" title="Copy configuration">📋 Copy</button>
          <button type="submit" class="btn btn-save" :disabled="saving">
            {{ saving ? 'Saving...' : 'Save' }}
          </button>
        </div>
      </div>

    </form>
  </div>
</template>

<style scoped>
.settings-card {
  padding: 0;
}

.settings-section {
  border-bottom: 1px solid #3f3f46;
}

.settings-section:last-of-type {
  border-bottom: none;
}

.section-title {
  list-style: none;
  padding: 16px 20px;
  font-weight: 600;
  font-size: 1rem;
  color: #e4e4e7;
  cursor: pointer;
  user-select: none;
  display: flex;
  align-items: center;
  gap: 8px;
}

.section-title::before {
  content: '▶';
  font-size: 0.7em;
  color: #7c3aed;
  transition: transform 0.2s;
}

details[open] > .section-title::before {
  transform: rotate(90deg);
}

.section-title::-webkit-details-marker {
  display: none;
}

.section-body {
  padding: 10px 20px 20px 20px;
  background: #1c1c1f;
  border-top: 1px solid #3f3f46;
}

.grid-2 {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 15px;
}

@media (max-width: 600px) {
  .grid-2 { grid-template-columns: 1fr; }
}

.path-row {
  display: flex;
  gap: 8px;
  align-items: center;
}

.path-row .field-input {
  flex: 1;
}

.sub-options {
  margin-left: 25px;
  border-left: 2px solid #3f3f46;
  padding-left: 15px;
  margin-top: 5px;
  margin-bottom: 5px;
}

.sub-section-title {
  font-size: 0.9rem;
  font-weight: 700;
  color: #a1a1aa;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin: 20px 0 10px;
  padding-bottom: 5px;
  border-bottom: 1px solid #3f3f46;
}

.badge-deprecated {
  background: #7f1d1d;
  color: #fca5a5;
  font-size: 0.7rem;
  padding: 2px 6px;
  border-radius: 4px;
  margin-left: 8px;
  font-weight: 600;
}

.info-msg {
  background: #1e3a5f;
  border: 1px solid #2563eb;
  color: #93c5fd;
  border-radius: 6px;
  padding: 10px 14px;
  font-size: 0.875rem;
  margin-top: 10px;
}

.btn-row {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
  margin-top: 10px;
}

.footer-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 20px;
  background: #18181b;
  border-top: 1px solid #3f3f46;
  flex-wrap: wrap;
  gap: 10px;
}

.last-run {
  display: flex;
  gap: 8px;
  align-items: center;
  font-size: 0.875rem;
  color: #a1a1aa;
}

.action-row {
  display: flex;
  gap: 10px;
  align-items: center;
}

.state-msg {
  text-align: center;
  padding: 40px;
  color: #a1a1aa;
}
</style>
