<script setup>
import {ref, watch} from 'vue'
import { MS_PER_DAY_MINUS_ONE } from '../utils/Constants'
import ApiService from '../utils/ApiService'

const props = defineProps({
    subscription: {
        type: Object,
        default: null
    }
})

const emit = defineEmits(['save', 'cancel'])

const editedSub = ref(null)
const activeTab = ref('basic')

const availableChannels = ref([])
const availableTopics = ref([])

const Dashboard = window.Dashboard ?? null

async function loadAutocompleteData() {
    try {
        const [channels, topics] = await Promise.all([
            ApiService.getChannels(),
            ApiService.getTopics()
        ])
        availableChannels.value = channels || []
        availableTopics.value = topics || []
    } catch (e) {
        console.error('Failed to load autocomplete data', e)
    }
}

loadAutocompleteData()

watch(() => props.subscription, (newVal) => {
    if (newVal) {
        // Deep copy
        const copy = JSON.parse(JSON.stringify(newVal))
        // Ensure nested objects are initialized to prevent template crashes
        copy.Search = copy.Search || {}
        copy.Search.Criteria = copy.Search.Criteria || []
        copy.Download = copy.Download || {}
        copy.Series = copy.Series || {}
        copy.Metadata = copy.Metadata || {}
        copy.Accessibility = copy.Accessibility || {}
        editedSub.value = copy
        // Reset active tab when a new subscription is opened
        activeTab.value = 'basic'
    } else {
        editedSub.value = null
    }
}, {immediate: true, deep: true})

function addQuery() {
    editedSub.value.Search.Criteria.push({
        Fields: ['Title', 'Topic'],
        Query: '',
        IsExclude: false
    })
}

function removeQuery(index) {
    editedSub.value.Search.Criteria.splice(index, 1)
}

function toggleField(query, field) {
    const index = query.Fields.indexOf(field)
    if (index > -1) {
        if (query.Fields.length > 1) {
            query.Fields.splice(index, 1)
        }
    } else {
        query.Fields.push(field)
    }
}

async function save() {
    emit('save', editedSub.value)
}

function cancel() {
    emit('cancel')
}

function selectPath() {
    if (!Dashboard) return
    const picker = new Dashboard.DirectoryBrowser()
    picker.show({
        header: 'Select Subscription Path',
        includeDirectories: true,
        includeFiles: false,
        callback: (path) => {
            if (path) {
                editedSub.value.Download.DownloadPath = path
            }
            picker.close()
        }
    })
}

// Utility to format date for input[type=date]
function formatDate(dateStr) {
    if (!dateStr) return ''
    return dateStr.split('T')[0]
}

function updateDate(target, field, value) {
    if (!value) {
        target[field] = null
        return
    }
    let date = new Date(value)
    if (field === 'MaxBroadcastDate') {
        date = new Date(date.getTime() + MS_PER_DAY_MINUS_ONE)
    }
    target[field] = date.toISOString()
}
</script>

<template>
    <div v-if="editedSub" class="editor-overlay">
        <div class="editor-modal card">
            <header class="editor-header">
                <h2>{{ editedSub.Id ? 'Edit Subscription' : 'New Subscription' }}</h2>
                <div class="header-actions">
                    <button @click="cancel" class="btn-icon">✕</button>
                </div>
            </header>

            <div class="editor-tabs">
                <button class="tab-btn" :class="{ active: activeTab === 'basic' }" @click="activeTab = 'basic'">General</button>
                <button class="tab-btn" :class="{ active: activeTab === 'search' }" @click="activeTab = 'search'">Search</button>
                <button class="tab-btn" :class="{ active: activeTab === 'download' }" @click="activeTab = 'download'">Download</button>
                <button class="tab-btn" :class="{ active: activeTab === 'series' }" @click="activeTab = 'series'">Series</button>
                <button class="tab-btn" :class="{ active: activeTab === 'metadata' }" @click="activeTab = 'metadata'">Metadata</button>
                <button class="tab-btn" :class="{ active: activeTab === 'accessibility' }" @click="activeTab = 'accessibility'">Accessibility</button>
            </div>

            <div class="editor-content">
                <!-- Allgemein Tab -->
                <div v-if="activeTab === 'basic'" class="tab-pane">
                    <div class="field">
                        <label>Name (Series Name)</label>
                        <input v-model="editedSub.Name" type="text" class="field-input" placeholder="e.g. Tatort" required>
                    </div>
                    <div class="checkbox-field">
                        <label>
                            <input v-model="editedSub.IsEnabled" type="checkbox"> Enabled
                        </label>
                    </div>

                    <div class="checkbox-field" hidden>
                        <label>
                            <input v-model="editedSub.IgnoreLocalFiles" type="checkbox"> Ignore Local Files
                        </label>
                        <p class="field-desc">Forces the download even if the file already exists locally.</p>
                    </div>
                    <div class="checkbox-field" hidden>
                        <label>
                            <input v-model="editedSub.IgnoreHistory" type="checkbox"> Ignore Download History
                        </label>
                        <p class="field-desc">Forces the download even if the program was previously downloaded.</p>
                    </div>
                </div>

                <!-- Suche Tab -->
                <div v-if="activeTab === 'search'" class="tab-pane">
                    <h3>Search Queries</h3>
                    <div v-for="(query, idx) in editedSub.Search.Criteria" :key="idx" class="query-row">
                        <div class="query-fields">
                            <button
                                v-for="f in ['Title', 'Topic', 'Description', 'Channel']"
                                :key="f"
                                @click="toggleField(query, f)"
                                class="field-tag"
                                :class="{ active: query.Fields.includes(f) }"
                            >
                                {{ f === 'Title' ? 'Title' : f === 'Topic' ? 'Topic' : f === 'Description' ? 'Description' : 'Channel' }}
                            </button>
                        </div>
                        <div class="query-input-row">
                            <input
                                v-model="query.Query"
                                type="text"
                                class="field-input"
                                :placeholder="query.IsExclude ? 'Exclude...' : 'Search...'"
                                :list="query.Fields.includes('Channel') && !query.Fields.includes('Topic') ? 'sub-channels' : (query.Fields.includes('Topic') ? 'sub-topics' : null)"
                            >
                            <button @click="query.IsExclude = !query.IsExclude" class="btn-small" :class="{ 'btn-danger': query.IsExclude }">
                                {{ query.IsExclude ? 'NOT' : 'SEARCH' }}
                            </button>
                            <button @click="removeQuery(idx)" class="btn-icon">🗑️</button>
                        </div>
                    </div>
                    <datalist id="sub-channels">
                        <option v-for="channel in availableChannels" :key="channel" :value="channel" />
                    </datalist>
                    <datalist id="sub-topics">
                        <option v-for="topic in availableTopics" :key="topic" :value="topic" />
                    </datalist>
                    <button @click="addQuery" class="btn btn-secondary">Add Query</button>

                    <hr>
                    <div class="grid-2">
                        <div class="field">
                            <label>Min. Duration (minutes)</label>
                            <input v-model="editedSub.Search.MinDurationMinutes" type="number" class="field-input">
                        </div>
                        <div class="field">
                            <label>Max. Duration (minutes)</label>
                            <input v-model="editedSub.Search.MaxDurationMinutes" type="number" class="field-input">
                        </div>
                    </div>
                    <div class="grid-2">
                        <div class="field">
                            <label>Min. Broadcast Date</label>
                            <input :value="formatDate(editedSub.Search.MinBroadcastDate)" @input="updateDate(editedSub.Search, 'MinBroadcastDate', $event.target.value)" type="date" class="field-input">
                        </div>
                        <div class="field">
                            <label>Max. Broadcast Date</label>
                            <input :value="formatDate(editedSub.Search.MaxBroadcastDate)" @input="updateDate(editedSub.Search, 'MaxBroadcastDate', $event.target.value)" type="date" class="field-input">
                        </div>
                    </div>
                </div>

                <!-- Download Tab -->
                <div v-if="activeTab === 'download'" class="tab-pane">
                    <div class="field">
                        <label>Download Path (Optional)</label>
                        <div class="input-with-btn">
                            <input v-model="editedSub.Download.DownloadPath" type="text" class="field-input" placeholder="Leave blank to use the default paths">
                            <button @click="selectPath" class="btn btn-secondary">Select</button>
                        </div>
                        <p class="field-desc">Leave blank to use the default path. For series, a subfolder using the subscription name is created automatically. For movies, a subscription-name subfolder is created only when the "Create a topic folder for movie downloads" setting is enabled.</p>
                    </div>
                    <div class="checkbox-field">
                        <label>
                            <input v-model="editedSub.Download.UseStreamingUrlFiles" type="checkbox"> Use Streaming URL Files (.strm)
                        </label>
                        <p class="field-desc">Uses streaming URL files (.strm) instead of downloading the actual video files. No video files are stored; videos are streamed directly from ARD/ZDF. Subtitles are unaffected.</p>
                    </div>
                    <div v-if="!editedSub.Download.UseStreamingUrlFiles" class="sub-options">
                        <div class="checkbox-field">
                            <label>
                                <input v-model="editedSub.Download.DownloadFullVideoForSecondaryAudio" type="checkbox"> Download Full Video for Secondary Audio Languages
                            </label>
                            <p class="field-desc">When enabled, the full video is downloaded even when it contains an audio language other than German. Otherwise, only that language's audio track is extracted.</p>
                        </div>
                    </div>
                    <div class="checkbox-field">
                        <label>
                            <input v-model="editedSub.Download.AlwaysCreateSubfolder" type="checkbox"> Create Subfolder for This Subscription
                        </label>
                        <p class="field-desc">Always creates a subfolder using the subscription name, including for movies when the global "Create a topic folder for movie downloads" setting is disabled.</p>
                    </div>
                    <div class="checkbox-field">
                        <label>
                            <input v-model="editedSub.Download.EnhancedDuplicateDetection" type="checkbox"> Enhanced Duplicate Detection
                        </label>
                        <p class="field-desc">Scans the destination directory for existing files with matching SxxExx patterns (or absolute numbering) to prevent duplicate downloads, even when filenames differ.</p>
                    </div>
                    <div class="checkbox-field">
                        <label>
                            <input v-model="editedSub.Download.AllowFallbackToLowerQuality" type="checkbox"> Allow Fallback to Lower Quality
                        </label>
                        <p class="field-desc">When enabled, the downloader checks whether a lower-quality version is available if the HD URL is unavailable.</p>
                    </div>
                    <div v-if="editedSub.Download.AllowFallbackToLowerQuality" class="sub-options">
                        <div class="checkbox-field">
                            <label>
                                <input v-model="editedSub.Download.QualityCheckWithUrl" type="checkbox"> Verify That URLs Are Valid
                            </label>
                            <p class="field-desc">When enabled, MediathekView URLs are also checked for availability and the next lower quality is attempted if necessary. HD → Default → SD</p>
                        </div>
                    </div>
                </div>

                <!-- Serien Tab -->
                <div v-if="activeTab === 'series'" class="tab-pane">
                    <div class="checkbox-field">
                        <label>
                            <input v-model="editedSub.Series.EnforceSeriesParsing" type="checkbox"> Download Series Only
                        </label>
                        <p class="field-desc">Download only videos recognized as series.</p>
                    </div>
                    <div v-if="editedSub.Series.EnforceSeriesParsing" class="sub-options">
                        <div class="checkbox-field">
                            <label>
                                <input v-model="editedSub.Series.AllowAbsoluteEpisodeNumbering" type="checkbox"> Allow Absolute Episode Numbering
                            </label>
                            <p class="field-desc">Download episodes even when only absolute episode numbering is available (e.g. "Episode 5" instead of "Season 1, Episode 5").</p>
                        </div>
                    </div>
                    <div v-else class="sub-options">
                        <div class="checkbox-field">
                            <label>
                                <input v-model="editedSub.Series.TreatNonEpisodesAsExtras" type="checkbox"> Treat Non-Episodes as Extras
                            </label>
                            <p class="field-desc">Treat videos not recognized as episodes as extras.</p>
                        </div>
                        <div v-if="editedSub.Series.TreatNonEpisodesAsExtras" class="sub-options">
                            <div class="checkbox-field">
                                <label><input v-model="editedSub.Series.SaveTrailers" type="checkbox"> Save Trailers</label>
                                <p class="field-desc">Trailers are saved.</p>
                            </div>
                            <div class="checkbox-field">
                                <label><input v-model="editedSub.Series.SaveInterviews" type="checkbox"> Save Interviews</label>
                                <p class="field-desc">Interviews are saved.</p>
                            </div>
                            <div class="checkbox-field">
                                <label><input v-model="editedSub.Series.SaveGenericExtras" type="checkbox"> Save Generic Extras</label>
                                <p class="field-desc">All other extras except trailers and interviews are saved.</p>
                            </div>
                            <div class="checkbox-field">
                                <label><input v-model="editedSub.Series.SaveExtrasAsStrm" type="checkbox"> Save Extras as Streams (.strm)</label>
                                <p class="field-desc">Extras are saved as .strm files to conserve disk space.</p>
                            </div>
                        </div>
                    </div>
                </div>

                <!-- Metadaten Tab -->
                <div v-if="activeTab === 'metadata'" class="tab-pane">
                    <div class="field">
                        <label>Original Language (ISO code, e.g. 'eng')</label>
                        <input v-model="editedSub.Metadata.OriginalLanguage" type="text" class="field-input" placeholder="e.g. eng or fra">
                        <p class="field-desc">When set, this language code is used when content is recognized as an original-language version (OV/OmU), instead of 'und'.</p>
                    </div>
                    <div class="checkbox-field">
                        <label>
                            <input v-model="editedSub.Metadata.CreateNfo" type="checkbox"> Create NFO Files
                        </label>
                        <p class="field-desc">Creates an .nfo file containing metadata (description and episode number) alongside the video file.</p>
                    </div>
                    <div class="checkbox-field">
                        <label>
                            <input v-model="editedSub.Metadata.AppendDateToTitle" type="checkbox"> Append Date to Title
                        </label>
                        <p class="field-desc">Appends the broadcast date to the title (e.g. "Title - 2026-01-01") and forces series detection. Useful for programs such as "Tagesschau in 100 Sekunden" that do not include a release date in the title.</p>
                    </div>
                    <div v-if="editedSub.Metadata.AppendDateToTitle" class="sub-options">
                        <div class="checkbox-field">
                            <label>
                                <input v-model="editedSub.Metadata.AppendTimeToTitle" type="checkbox"> Append Time to Title
                            </label>
                            <p class="field-desc">Appends the broadcast time to the title (e.g. "Title - 2026-01-01 20-00").</p>
                        </div>
                    </div>
                    <div class="checkbox-field">
                        <label>
                            <input v-model="editedSub.Metadata.KeepOriginalTitle" type="checkbox"> Keep Original Title
                        </label>
                        <p class="field-desc">Keeps the original title and does not remove information such as (AD), sign language, or episode numbers.</p>
                    </div>
                </div>

                <!-- Barrierefreiheit Tab -->
                <div v-if="activeTab === 'accessibility'" class="tab-pane">
                    <div class="checkbox-field">
                        <label>
                            <input v-model="editedSub.Accessibility.AllowAudioDescription" type="checkbox"> Download Audio-Described Versions
                        </label>
                        <p class="field-desc">Also downloads audio-described content when available.</p>
                    </div>
                    <div class="checkbox-field">
                        <label>
                            <input v-model="editedSub.Accessibility.AllowSignLanguage" type="checkbox"> Download Sign-Language Versions
                        </label>
                        <p class="field-desc">Also downloads sign-language versions when available.</p>
                    </div>
                </div>
            </div>

            <footer class="editor-footer">
                <button @click="cancel" class="btn btn-secondary">Cancel</button>
                <button @click="$emit('test', editedSub)" class="btn btn-secondary">Test Subscription (Dry Run)</button>
                <button @click="save" class="btn btn-primary">Save Subscription</button>
            </footer>
        </div>
    </div>
</template>

<style scoped>
.editor-overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(0, 0, 0, 0.8);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 9999;
    padding: 20px;
}

.editor-modal {
    width: 100%;
    max-width: 800px;
    height: 80vh;
    min-height: 500px;
    display: flex;
    flex-direction: column;
    overflow: hidden;
}

.editor-header {
    padding: 20px;
    border-bottom: 1px solid #3f3f46;
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.editor-tabs {
    display: flex;
    background: #27272a;
    border-bottom: 1px solid #3f3f46;
    overflow-x: auto;
}

.editor-content {
    padding: 20px;
    overflow-y: auto;
    flex: 1;
}

.editor-footer {
    padding: 20px;
    border-top: 1px solid #3f3f46;
    display: flex;
    justify-content: flex-end;
    gap: 15px;
}

.tab-btn {
    padding: 12px 20px;
    background: none;
    border: none;
    color: #a1a1aa;
    cursor: pointer;
    white-space: nowrap;
}

.tab-btn.active {
    color: #7c3aed;
    background: #18181b;
    border-bottom: 2px solid #7c3aed;
}

.grid-2 {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 20px;
}

.query-row {
    background: #27272a;
    padding: 15px;
    border-radius: 8px;
    margin-bottom: 15px;
    border: 1px solid #3f3f46;
}

.query-fields {
    display: flex;
    gap: 8px;
    margin-bottom: 10px;
}

.field-tag {
    padding: 4px 10px;
    border-radius: 12px;
    background: #3f3f46;
    border: none;
    color: #a1a1aa;
    font-size: 0.75rem;
    cursor: pointer;
}

.field-tag.active {
    background: #7c3aed;
    color: white;
}

.query-input-row {
    display: flex;
    gap: 10px;
    align-items: center;
}

.input-with-btn {
    display: flex;
    gap: 10px;
}

.sub-options {
    margin-left: 25px;
    border-left: 2px solid #3f3f46;
    padding-left: 15px;
    margin-top: 10px;
    margin-bottom: 10px;
}

.btn-small {
    padding: 5px 10px;
    border-radius: 4px;
    border: 1px solid #3f3f46;
    background: #27272a;
    color: white;
    cursor: pointer;
    font-size: 0.75rem;
}

.btn-danger {
    background: #ef4444;
    border-color: #ef4444;
}
</style>
