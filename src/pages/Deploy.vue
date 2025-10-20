// src/pages/Deploy.vue (refactor: fetch ABI/WASM from CORS-friendly mirrors with fallback)
<template>
  <section id="deploy" class="container mx-auto px-4 py-6">
    <Toast />
    <ConfirmDialog />

    <div class="flex items-center justify-between">
      <div>
        <h1 class="text-2xl font-bold">Smart Contract Manager</h1>
        <p class="mt-1 text-slate-600">
          Antelope-native operations using VSR (VSR URI). Manage code & ABI safely from your browser.
        </p>
      </div>
    </div>

    <div class="mt-5 card rounded-xl border border-surface-200/70 p-3 sm:p-4">
      <Tabs v-model:value="tab" class="w-full">
        <TabList>
          <Tab value="deploy">Deploy Contract</Tab>
          <Tab value="clear">Clear Contract</Tab>
        </TabList>

        <TabPanels>
          <TabPanel value="deploy">
            <div class="grid gap-4 md:grid-cols-2">
              <div class="rounded-xl border border-surface-200/70 p-4 space-y-4">
                <div class="flex items-center justify-between">
                  <div class="text-sm">
                    <div class="text-slate-600 mb-0.5">Contract Account</div>
                    <div class="font-medium">
                      <span v-if="isConnected">{{ actorLabel }}</span>
                      <span v-else class="opacity-60">Not connected</span>
                    </div>
                  </div>
                  <Badge :value="sourceMode === 'github' ? githubBadge : `${uploadedCount}/2 files`" :severity="badgeSeverity" />
                </div>

                <div class="rounded-lg border border-surface-200/70 p-3">
                  <div class="text-sm font-semibold mb-2">Artifacts Source</div>
                  <div class="flex flex-col gap-3">
                    <div class="flex items-center gap-2">
                      <RadioButton inputId="srcGithub" value="github" v-model="sourceMode" />
                      <label for="srcGithub" class="cursor-pointer">From Mirrors (Pages/Raw/jsDelivr)</label>
                    </div>
                    <div v-if="sourceMode === 'github'" class="grid gap-3">
                      <div class="text-xs text-slate-600">
                        Default mirrors try <code>GitHub Pages → Raw → jsDelivr</code>. You may override below.
                      </div>
                      <div>
                        <label class="text-xs text-slate-600">ABI override (optional)</label>
                        <InputText v-model.trim="abiUrlOverride" placeholder="https://…" class="w-full" />
                      </div>
                      <div>
                        <label class="text-xs text-slate-600">WASM override (optional)</label>
                        <InputText v-model.trim="wasmUrlOverride" placeholder="https://…" class="w-full" />
                      </div>
                      <div class="flex items-center gap-2">
                        <Button :label="loadingGithub ? 'Loading…' : 'Load from Mirrors'" icon="pi pi-download"
                          :loading="loadingGithub" :disabled="busy" @click="loadFromGithub" />
                        <small class="opacity-70">No upload required.</small>
                      </div>
                    </div>

                    <Divider class="!my-2" />

                    <div class="flex items-center gap-2">
                      <RadioButton inputId="srcFiles" value="files" v-model="sourceMode" />
                      <label for="srcFiles" class="cursor-pointer">From Files</label>
                    </div>
                    <div v-if="sourceMode === 'files'" class="flex flex-col gap-3">
                      <FileUpload mode="basic" name="contract[]" :multiple="true" :auto="true" :maxFileSize="20000000"
                        :customUpload="true" accept=".wasm,.abi,application/wasm,application/json,text/json"
                        chooseLabel="Choose WASM & ABI" @uploader="onCustomUpload" @select="onSelectedFiles"
                        @clear="onClear" @error="onUploadError" />
                      <ProgressBar :value="totalSizePercent" :showValue="false" class="h-1" />
                      <div class="text-xs text-slate-600">{{ totalSizeLabel }}</div>
                    </div>
                  </div>
                </div>

                <div class="grid gap-1 text-xs text-slate-700">
                  <div v-if="wasmInfo"><b>WASM</b> — {{ wasmInfo.size }} • sha256 {{ wasmInfo.sha }}</div>
                  <div v-if="abiInfo"><b>ABI</b> — {{ abiInfo.size }} • sha256 {{ abiInfo.sha }}</div>
                </div>

                <div class="flex justify-end gap-2">
                  <Button :label="busy ? 'Deploying…' : 'Deploy'" icon="pi pi-upload" :loading="busy"
                    :disabled="!canDeploy || busy" @click="confirmDeploy" />
                  <Button v-if="(abiJson || wasmBytes)" label="Clear Selection" severity="secondary" outlined @click="onClear" />
                </div>
              </div>

              <div class="rounded-xl border border-surface-200/70 p-4">
                <div class="text-lg font-semibold">Deployment Notes</div>
                <ul class="list-disc pl-5 text-sm text-slate-600 space-y-2 mt-3">
                  <li><code>setabi</code> packs <code>ABI.from(json)</code> via <code>Serializer.encode</code>.</li>
                  <li><code>setcode</code> sends <code>Bytes.from(wasm)</code> (vmtype=0, vmversion=0).</li>
                  <li>VSR requests: <b>setabi → setcode</b>.</li>
                  <li>Artifacts fetched from mirrors (CORS-friendly): Pages → Raw → jsDelivr.</li>
                </ul>
              </div>
            </div>
          </TabPanel>

          <TabPanel value="clear">
            <div class="grid gap-4 md:grid-cols-2">
              <div class="rounded-xl border border-surface-200/70 p-4">
                <div class="space-y-2">
                  <div class="text-lg font-semibold">Clear Contract (Reset Code & ABI)</div>
                  <p class="text-sm text-slate-600">
                    Sets empty ABI and zero-length wasm (no contract).
                  </p>
                  <ul class="list-disc pl-5 text-sm text-slate-600">
                    <li><code>setabi</code> → empty ABI</li>
                    <li><code>setcode</code> → empty code</li>
                    <li>No file upload required</li>
                  </ul>
                </div>
                <Divider class="my-5" />
                <div class="flex justify-end gap-2">
                  <Button :label="busy ? 'Clearing…' : 'Clear Contract'" icon="pi pi-trash" severity="danger"
                    :loading="busy" :disabled="!isConnected || busy" @click="confirmClear" />
                </div>
              </div>

              <div class="rounded-xl border border-surface-200/70 p-4">
                <div class="text-lg font-semibold">Safety Notes</div>
                <ul class="list-disc pl-5 text-sm text-slate-600 space-y-2 mt-3">
                  <li>Ensure no production traffic relies on this contract before clearing.</li>
                  <li>Re-deploy any time to restore functionality.</li>
                </ul>
              </div>
            </div>
          </TabPanel>
        </TabPanels>
      </Tabs>
    </div>
  </section>
</template>

<script setup>
import { ref, computed } from 'vue'
import Button from 'primevue/button'
import FileUpload from 'primevue/fileupload'
import ProgressBar from 'primevue/progressbar'
import Badge from 'primevue/badge'
import Toast from 'primevue/toast'
import Divider from 'primevue/divider'
import ConfirmDialog from 'primevue/confirmdialog'
import RadioButton from 'primevue/radiobutton'
import InputText from 'primevue/inputtext'
import { useToast } from 'primevue/usetoast'
import { useConfirm } from 'primevue/useconfirm'

import { Action, Bytes, ABI, Serializer, Checksum256 } from '@wharfkit/antelope'
import { PlaceholderAuth, PlaceholderName } from '@wharfkit/signing-request'

import Wallet from '@/core/wallet.js'
import { abiCache } from '@/core/nodes.js'
import { getErrorMessage } from '@/core/utils.js'

defineOptions({ name: 'DeployPage' })

const tab = ref('deploy')
const toast = useToast()
const confirm = useConfirm()

const sourceMode = ref('github')

const ABI_MIRRORS = [
  'https://windvex.github.io/token.wind/token.wind/latest/token.abi',
  'https://raw.githubusercontent.com/windvex/token.wind/main/dist/token.wind/latest/token.abi',
  'https://cdn.jsdelivr.net/gh/windvex/token.wind@main/dist/token.wind/latest/token.abi',
]
const WASM_MIRRORS = [
  'https://windvex.github.io/token.wind/token.wind/latest/token.wasm',
  'https://raw.githubusercontent.com/windvex/token.wind/main/dist/token.wind/latest/token.wasm',
  'https://cdn.jsdelivr.net/gh/windvex/token.wind@main/dist/token.wind/latest/token.wasm',
]

const abiUrlOverride = ref('')
const wasmUrlOverride = ref('')

const loadingGithub = ref(false)

const totalSize = ref(0)
const totalSizePercent = ref(0)

const wasmBytes = ref(/** @type {Uint8Array|null} */(null))
const abiJson   = ref(/** @type {any|null} */(null))
const wasmInfo  = ref(null)
const abiInfo   = ref(null)

const uploadedCount = computed(() => Number(!!wasmBytes.value) + Number(!!abiJson.value))
const busy = ref(false)

const isConnected = computed(() => {
  try { return Wallet?.isConnected?.() === true } catch { return false }
})
const actorLabel = computed(() => {
  try { return Wallet.getPermission()?.actor?.toString?.() || '' } catch { return '' }
})
const canDeploy = computed(() => isConnected.value && actorLabel.value && (wasmBytes.value instanceof Uint8Array) && !!abiJson.value)

const githubBadge = computed(() => (abiJson.value && wasmBytes.value) ? 'Mirrors ready' : 'Not loaded')
const badgeSeverity = computed(() => {
  if (sourceMode.value === 'github') return (abiJson.value && wasmBytes.value) ? 'success' : 'secondary'
  return uploadedCount.value === 2 ? 'success' : (uploadedCount.value === 1 ? 'warn' : 'secondary')
})

function formatSize(bytes) {
  if (bytes == null) return '—'
  const k = 1024
  const i = Math.floor(Math.log(bytes) / Math.log(k))
  const sizes = ['B', 'KB', 'MB', 'GB']
  const val = (bytes / Math.pow(k, i)).toFixed(i === 0 ? 0 : 2)
  return `${val} ${sizes[i]}`
}
const totalSizeLabel = computed(() => `${formatSize(totalSize.value)} / 20 MB`)
function prettySize(len) { return len == null ? '—' : formatSize(len) }
function sha256Hex(u8) { return Checksum256.hash(Bytes.from(u8)).hexString }

async function firstOk(urls, as = 'json') {
  let lastErr
  for (const url of urls) {
    try {
      const res = await fetch(url, { redirect: 'follow' })
      if (!res.ok) throw new Error(`HTTP ${res.status}`)
      if (as === 'json') return await res.json()
      const buf = await res.arrayBuffer()
      return new Uint8Array(buf)
    } catch (e) { lastErr = e }
  }
  throw lastErr || new Error('No URL worked')
}

async function loadFromGithub() {
  try {
    loadingGithub.value = true
    onClear()

    const abiUrls = abiUrlOverride.value ? [abiUrlOverride.value] : ABI_MIRRORS
    const wasmUrls = wasmUrlOverride.value ? [wasmUrlOverride.value] : WASM_MIRRORS

    const [json, bin] = await Promise.all([
      firstOk(abiUrls, 'json'),
      firstOk(wasmUrls, 'bin'),
    ])

    abiJson.value = json
    const rawAbi = new TextEncoder().encode(JSON.stringify(json))
    abiInfo.value = { size: prettySize(rawAbi.byteLength), sha: sha256Hex(rawAbi) }

    wasmBytes.value = bin
    wasmInfo.value = { size: prettySize(bin.byteLength), sha: sha256Hex(bin) }

    toast.add({ severity: 'success', summary: 'Artifacts loaded from mirrors', life: 2600 })
  } catch (err) {
    console.error('[loadFromGithub]', err)
    toast.add({ severity: 'error', summary: 'Failed loading artifacts', detail: getErrorMessage(err), life: 5200 })
  } finally {
    loadingGithub.value = false
  }
}

function onClear() {
  totalSize.value = 0
  totalSizePercent.value = 0
  wasmBytes.value = null
  abiJson.value = null
  wasmInfo.value = null
  abiInfo.value = null
}
function onSelectedFiles(event) {
  const files = event?.files || []
  totalSize.value = files.reduce((a, f) => a + (f?.size || 0), 0)
  totalSizePercent.value = Math.min(100, Math.round((totalSize.value / (20 * 1024 * 1024)) * 100))
}
function onUploadError(e) {
  console.error('[onUploadError]', e)
  toast.add({ severity: 'error', summary: 'Upload error', detail: 'Unable to process files.', life: 4000 })
}
async function onCustomUpload(event) {
  try {
    const selected = Array.isArray(event.files) ? event.files : [event.files]
    if (!selected.length) return

    onClear()
    event.options?.progress?.(10)

    for (const f of selected) {
      try {
        const name = f?.name?.toLowerCase?.() || ''
        if (name.endsWith('.wasm') || f.type === 'application/wasm') {
          const u8 = new Uint8Array(await f.arrayBuffer())
          wasmBytes.value = u8
          wasmInfo.value = { size: prettySize(u8.byteLength), sha: sha256Hex(u8) }
        } else if (name.endsWith('.abi') || name.endsWith('.json') || f.type === 'application/json' || f.type === 'text/json') {
          const text = await f.text()
          const json = JSON.parse(text)
          abiJson.value = json
          const raw = new TextEncoder().encode(JSON.stringify(json))
          abiInfo.value = { size: prettySize(raw.byteLength), sha: sha256Hex(raw) }
        }
      } catch (fe) {
        toast.add({ severity: 'error', summary: `Failed reading ${f?.name}`, detail: getErrorMessage(fe), life: 4500 })
      }
    }

    event.options?.progress?.(100)
    totalSizePercent.value = 100
    event.options?.complete?.()

    toast.add({
      severity: (abiJson.value && wasmBytes.value) ? 'success' : 'warn',
      summary: (abiJson.value && wasmBytes.value) ? 'WASM & ABI loaded' : 'Please select both .wasm and .abi',
      life: 2800
    })
  } catch (err) {
    console.error('[onCustomUpload] error', err)
    event.options?.progress?.(0)
    toast.add({ severity: 'error', summary: 'Failed to process files', detail: getErrorMessage(err), life: 4500 })
  }
}

function login() {
  if (typeof Wallet?.show === 'function') {
    try { Wallet.show() } catch (e) { console.error('[login] Wallet.show error', e) }
  } else {
    window.dispatchEvent(new CustomEvent('wind:wallet:connect'))
  }
}

function confirmDeploy() {
  if (!canDeploy.value) return
  confirm.require({
    key: 'deploy',
    header: 'Deploy Smart Contract',
    message: `Deploy WASM & ABI to ${actorLabel.value}?`,
    icon: 'pi pi-exclamation-triangle',
    rejectProps: { label: 'Cancel', severity: 'secondary', outlined: true },
    acceptProps: { label: 'Deploy' },
    accept: deployNow,
  })
}

async function deployNow() {
  if (!canDeploy.value) return
  if (!Wallet.session) { login(); return }

  busy.value = true
  try {
    try { Wallet.session.setABICache?.(abiCache) } catch {}

    const system = 'vexcore'
    const sysAbi = await abiCache.getAbi(system)

    const abiDef    = ABI.from(abiJson.value)
    const abiPacked = Serializer.encode({ object: abiDef, type: ABI })
    const setabi = Action.from({
      account: system,
      name: 'setabi',
      authorization: [PlaceholderAuth],
      data: { account: PlaceholderName, abi: abiPacked }
    }, sysAbi)

    const setcode = Action.from({
      account: system,
      name: 'setcode',
      authorization: [PlaceholderAuth],
      data: { account: PlaceholderName, vmtype: 0, vmversion: 0, code: Bytes.from(wasmBytes.value) }
    }, sysAbi)

    const req = await Wallet.createSigningRequest({ actions: [setabi, setcode] })
    const vsr = req.encode?.(true, true, 'vsr:') ?? req.encodeUri?.('vsr:')
    if (!vsr) throw new Error('Unable to encode VSR (VSR URI).')
    await Wallet.session.signingRequest(vsr)

    onClear()
    toast.add({ severity: 'success', summary: 'Contract deployed (setabi → setcode)', life: 3200 })
  } catch (err) {
    toast.add({ severity: 'error', summary: 'Deploy failed', detail: getErrorMessage(err), life: 5200 })
  } finally {
    busy.value = false
  }
}

function confirmClear() {
  if (!isConnected.value) { login(); return }
  confirm.require({
    key: 'clear',
    header: 'Clear Contract',
    message: `Reset ABI and Code for ${actorLabel.value}? This action will remove the deployed contract.`,
    icon: 'pi pi-exclamation-triangle',
    rejectProps: { label: 'Cancel', severity: 'secondary', outlined: true },
    acceptProps: { label: 'Clear Contract', severity: 'danger' },
    accept: clearNow,
  })
}

async function clearNow() {
  if (!Wallet.session) { login(); return }

  busy.value = true
  try {
    try { Wallet.session.setABICache?.(abiCache) } catch {}

    const system = 'vexcore'
    const sysAbi = await abiCache.getAbi(system)

    const emptyAbiDef = ABI.from({
      version: 'eosio::abi/1.3',
      types: [], variants: [], structs: [], actions: [], tables: [],
      ricardian_clauses: [], action_results: []
    })
    const emptyAbiPacked = Serializer.encode({ object: emptyAbiDef, type: ABI })

    const setabiEmpty = Action.from({
      account: system,
      name: 'setabi',
      authorization: [PlaceholderAuth],
      data: { account: PlaceholderName, abi: emptyAbiPacked }
    }, sysAbi)

    const setcodeEmpty = Action.from({
      account: system,
      name: 'setcode',
      authorization: [PlaceholderAuth],
      data: { account: PlaceholderName, vmtype: 0, vmversion: 0, code: Bytes.from(new Uint8Array()) }
    }, sysAbi)

    const req = await Wallet.createSigningRequest({ actions: [setabiEmpty, setcodeEmpty] })
    const vsr = req.encode?.(true, true, 'vsr:') ?? req.encodeUri?.('vsr:')
    if (!vsr) throw new Error('Unable to encode VSR (VSR URI).')
    await Wallet.session.signingRequest(vsr)

    toast.add({ severity: 'success', summary: 'Contract cleared (ABI & Code reset)', life: 3200 })
  } catch (err) {
    toast.add({ severity: 'error', summary: 'Clear failed', detail: getErrorMessage(err), life: 5200 })
  } finally {
    busy.value = false
  }
}
</script>
