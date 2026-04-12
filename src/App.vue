<script setup lang="ts">
import { ref, computed, nextTick } from 'vue'
import { ESPLoader, Transport, CustomReset } from 'esptool-js'

type FlashFileEntry = {
  path: string
  address: number
}

type FlashManifest = {
  flash_mode: string
  flash_freq: string
  flash_size: string
  files: FlashFileEntry[]
}

type FlashModeValues = 'qio' | 'qout' | 'dio' | 'dout' | 'keep'
type FlashFreqValues = 'keep' | '80m' | '60m' | '48m' | '40m' | '30m' | '26m' | '24m' | '20m' | '16m' | '15m' | '12m'
type FlashSizeValues =
  | 'detect'
  | 'keep'
  | '256KB'
  | '512KB'
  | '1MB'
  | '2MB'
  | '4MB'
  | '8MB'
  | '16MB'
  | '32MB'
  | '64MB'
  | '128MB'

type FlashFile = {
  data: Uint8Array
  address: number
}

type UiState =
  | 'idle'
  | 'connecting'
  | 'loadingManifest'
  | 'downloadingFiles'
  | 'flashing'
  | 'erasing'
  | 'resetting'
  | 'success'
  | 'error'

type FlashMode = 'update' | 'clean'

const isBusy = ref(false)
const progress = ref(0)
const currentFileName = ref('')
const logLines = ref<string[]>([])
const uiState = ref<UiState>('idle')
const logBoxRef = ref<HTMLElement | null>(null)
const selectedFlashMode = ref<FlashMode>('update')

let esploader: ESPLoader | null = null
let transport: Transport | null = null

const statusText = computed(() => {
  switch (uiState.value) {
    case 'idle':
      return 'Устройство не подключено'
    case 'connecting':
      return 'Подключение к устройству...'
    case 'loadingManifest':
      return 'Чтение манифеста прошивки...'
    case 'downloadingFiles':
      return 'Загрузка файлов прошивки...'
    case 'flashing':
      return selectedFlashMode.value === 'update'
        ? 'Идёт обновление прошивки без очистки NVS...'
        : 'Идёт полная прошивка с полной очисткой flash...'
    case 'erasing':
      return 'Выполняется полное стирание flash...'
    case 'resetting':
      return 'Перезагрузка устройства...'
    case 'success':
      return 'Операция успешно завершена'
    case 'error':
      return 'Произошла ошибка'
    default:
      return 'Готово'
  }
})

const modeDescription = computed(() => {
  return selectedFlashMode.value === 'update'
    ? 'Обновление сохраняет NVS и клиентские настройки устройства.'
    : 'Полная прошивка полностью очищает flash перед записью и удаляет все настройки.'
})

function stateClass(state: UiState): string {
  if (state === 'success') return 'is-success'
  if (state === 'error') return 'is-error'
  if (state === 'flashing' || state === 'erasing' || state === 'connecting') return 'is-active'
  return ''
}

function formatHex(address: number): string {
  return `0x${address.toString(16).toUpperCase()}`
}

async function scrollLogToBottom(): Promise<void> {
  await nextTick()
  if (logBoxRef.value) {
    logBoxRef.value.scrollTop = logBoxRef.value.scrollHeight
  }
}

function appendLog(message: string): void {
  logLines.value.push(message)
  console.log(message)
  void scrollLogToBottom()
}

function resetUiForOperation(): void {
  isBusy.value = true
  progress.value = 0
  currentFileName.value = ''
  logLines.value = []
  uiState.value = 'idle'
}

async function fetchJson<T>(url: string): Promise<T> {
  const response = await fetch(url, { cache: 'no-store' })
  if (!response.ok) throw new Error(`Не удалось загрузить JSON: ${url}`)
  return (await response.json()) as T
}

async function fetchBinary(path: string): Promise<Uint8Array> {
  const url = `/DarkForgeUI-site/firmware/${path}`
  const response = await fetch(url, { cache: 'no-store' })
  if (!response.ok) throw new Error(`Не удалось загрузить бинарник: ${url}`)
  const buffer = await response.arrayBuffer()
  return new Uint8Array(buffer)
}

async function openConnection(): Promise<void> {
  const nav = navigator as Navigator & {
    serial?: {
      requestPort: () => Promise<unknown>
    }
  }

  if (!nav.serial) {
    throw new Error('Web Serial API не поддерживается вашим браузером. Используйте Chrome или Edge.')
  }

  uiState.value = 'connecting'
  appendLog('Запрос доступа к последовательному порту...')

  const port = await nav.serial.requestPort()
  transport = new Transport(port as any, true)

  esploader = new ESPLoader({
    transport,
    baudrate: 460800,
    terminal: {
      clean: () => {},
      writeLine: (data: string) => appendLog(data),
      write: (data: string) => appendLog(data)
    }
  } as any)

  const chip = await (esploader as any).main()
  appendLog(`Подключено к чипу: ${chip}`)
}

async function closeConnection(): Promise<void> {
  if (transport) {
    appendLog('Отключение от порта...')
    await transport.disconnect().catch(() => {})
    transport = null
  }
  esploader = null
}

async function hardResetDevice(): Promise<void> {
  if (!transport) return
  uiState.value = 'resetting'
  appendLog('Выполняется аппаратный сброс устройства...')
  const resetStrategy = new CustomReset(transport, 'D0|R1|W200|R0|W1000')
  await resetStrategy.reset()
}

async function eraseDeviceFlash(): Promise<void> {
  try {
    resetUiForOperation()
    uiState.value = 'connecting'

    await openConnection()

    uiState.value = 'erasing'
    appendLog('Запуск полного стирания flash...')

    if (!(esploader as any)?.eraseFlash) {
      throw new Error('Текущая версия esptool-js не поддерживает eraseFlash().')
    }

    await (esploader as any).eraseFlash()

    appendLog('Flash-память успешно очищена.')
    progress.value = 100
    currentFileName.value = 'Flash очищена'

    await hardResetDevice()
    await closeConnection()

    uiState.value = 'success'
    appendLog('Устройство полностью очищено и готово к новой установке.')
  } catch (error: unknown) {
    const message = error instanceof Error ? error.message : String(error)
    uiState.value = 'error'
    appendLog(`Ошибка: ${message}`)
    console.error(error)
  } finally {
    await closeConnection()
    isBusy.value = false
  }
}

async function connectAndFlash(mode: FlashMode): Promise<void> {
  try {
    selectedFlashMode.value = mode
    resetUiForOperation()

    await openConnection()

    uiState.value = 'loadingManifest'
    appendLog('Чтение flash_args.json...')
    const manifest = await fetchJson<FlashManifest>('/DarkForgeUI-site/firmware/flash_args.json')

    uiState.value = 'downloadingFiles'
    appendLog('Загрузка бинарных файлов...')

    const fileArray: FlashFile[] = await Promise.all(
      manifest.files.map(async (file) => {
        appendLog(`Загрузка: ${file.path} → ${formatHex(file.address)}`)
        return {
          data: await fetchBinary(file.path),
          address: file.address
        }
      })
    )

    uiState.value = 'flashing'

    const eraseAll = mode === 'clean'

    appendLog(
      eraseAll
        ? 'Режим: полная прошивка с очисткой flash.'
        : 'Режим: обновление без полной очистки flash. Раздел NVS не стирается.'
    )

    appendLog('Запуск writeFlash...')
    appendLog(`Параметры: mode=${manifest.flash_mode}, freq=${manifest.flash_freq}, size=${manifest.flash_size}`)

    await (esploader as any).writeFlash({
      fileArray,
      eraseAll,
      compress: true,
      flashMode: manifest.flash_mode as FlashModeValues,
      flashFreq: manifest.flash_freq as FlashFreqValues,
      flashSize: manifest.flash_size as FlashSizeValues,
      reportProgress: (fileIndex: number, written: number, total: number) => {
        if (manifest.files[fileIndex]) {
          const file = manifest.files[fileIndex]
          currentFileName.value = `${file.path} · ${formatHex(file.address)}`
        }
        progress.value = total > 0 ? Math.round((written / total) * 100) : 0
      }
    })

    appendLog(
      eraseAll
        ? 'Полная прошивка успешно завершена.'
        : 'Обновление прошивки успешно завершено.'
    )

    progress.value = 100
    currentFileName.value = 'Готово'

    await hardResetDevice()
    await closeConnection()

    uiState.value = 'success'
    appendLog('Устройство успешно запущено.')
  } catch (error: unknown) {
    const message = error instanceof Error ? error.message : String(error)
    uiState.value = 'error'
    appendLog(`Ошибка: ${message}`)
    console.error(error)
  } finally {
    await closeConnection()
    isBusy.value = false
  }
}
</script>

<template>
  <div class="darkforge-container">
    <div class="card">
      <div class="hero">
        <div class="hero-badge">ESP32-P4 Web Flasher</div>
        <div class="logo">DarkForge UI</div>
        <div class="subtitle">
          Профессиональный веб-инструмент для обновления, полной прошивки и восстановления устройства
        </div>
      </div>

      <div class="status-panel" :class="stateClass(uiState)">
        <div class="status-dot"></div>
        <div class="status-content">
          <div class="status-label">Статус</div>
          <div class="status-text">{{ statusText }}</div>
        </div>
      </div>

      <div class="mode-panel">
        <div class="mode-header">
          <div class="mode-title">Режим прошивки</div>
          <div class="mode-subtitle">{{ modeDescription }}</div>
        </div>

        <div class="mode-options">
          <label class="mode-option" :class="{ active: selectedFlashMode === 'update' }">
            <input v-model="selectedFlashMode" type="radio" value="update" :disabled="isBusy" />
            <div class="mode-option-content">
              <div class="mode-option-title">Обновление</div>
              <div class="mode-option-text">Без очистки NVS, настройки клиента сохраняются</div>
            </div>
          </label>

          <label class="mode-option" :class="{ active: selectedFlashMode === 'clean' }">
            <input v-model="selectedFlashMode" type="radio" value="clean" :disabled="isBusy" />
            <div class="mode-option-content">
              <div class="mode-option-title">Полная прошивка</div>
              <div class="mode-option-text">Полное стирание flash перед записью, все данные удаляются</div>
            </div>
          </label>
        </div>
      </div>

      <div class="warning-box" :class="{ danger: selectedFlashMode === 'clean' }">
        <div class="warning-title">
          {{ selectedFlashMode === 'update' ? 'Сохранение настроек' : 'Полная очистка данных' }}
        </div>
        <div class="warning-text">
          {{
            selectedFlashMode === 'update'
              ? 'В режиме обновления раздел NVS не стирается, поэтому клиентские настройки обычно сохраняются.'
              : 'В режиме полной прошивки будет выполнено полное стирание flash перед записью. Все пользовательские данные и настройки будут удалены.'
          }}
        </div>
      </div>

      <div v-if="progress > 0 || currentFileName" class="progress-area">
        <div class="progress-header">
          <span class="file-badge">{{ currentFileName || 'Подготовка...' }}</span>
          <span class="percent">{{ progress }}%</span>
        </div>

        <div class="progress-track">
          <div class="progress-fill" :style="{ width: progress + '%' }"></div>
        </div>
      </div>

      <div class="actions">
        <button
          class="btn btn-primary"
          :disabled="isBusy"
          @click="connectAndFlash(selectedFlashMode)"
        >
          {{
            isBusy && uiState === 'flashing'
              ? selectedFlashMode === 'update'
                ? 'Идёт обновление...'
                : 'Идёт полная прошивка...'
              : selectedFlashMode === 'update'
                ? 'Запустить обновление'
                : 'Запустить полную прошивку'
          }}
        </button>

        <button class="btn btn-secondary" :disabled="isBusy" @click="eraseDeviceFlash">
          {{ isBusy && uiState === 'erasing' ? 'Стирание...' : 'Полная очистка flash' }}
        </button>
      </div>

      <div class="info-grid">
        <div class="info-card">
          <div class="info-label">Активный режим</div>
          <div class="info-value">
            {{ selectedFlashMode === 'update' ? 'Обновление без очистки NVS' : 'Полная прошивка с eraseAll' }}
          </div>
        </div>

        <div class="info-card">
          <div class="info-label">Recovery</div>
          <div class="info-value">Отдельная полная очистка flash доступна</div>
        </div>
      </div>

      <div v-if="logLines.length" class="log-box">
        <div class="log-header">
          <span>Журнал операций</span>
          <span class="log-count">{{ logLines.length }} записей</span>
        </div>

        <div ref="logBoxRef" class="log-list">
          <div v-for="(line, idx) in logLines" :key="idx" class="log-entry">
            {{ line }}
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
:global(html, body, #app) {
  min-height: 100%;
  margin: 0;
}

:global(body) {
  background:
    radial-gradient(circle at top, rgba(255, 191, 71, 0.12), transparent 25%),
    radial-gradient(circle at bottom left, rgba(59, 130, 246, 0.08), transparent 25%),
    linear-gradient(180deg, #0b0f14 0%, #0f141b 100%);
  color: #e8edf2;
  font-family: Inter, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
}

* {
  box-sizing: border-box;
}

.darkforge-container {
  min-height: 100vh;
  display: grid;
  place-items: center;
  padding: 32px 20px;
}

.card {
  width: 100%;
  max-width: 720px;
  padding: 28px;
  border-radius: 24px;
  background: rgba(17, 22, 29, 0.94);
  border: 1px solid #232c36;
  box-shadow:
    0 24px 80px rgba(0, 0, 0, 0.45),
    inset 0 1px 0 rgba(255, 255, 255, 0.03);
  backdrop-filter: blur(16px);
}

.hero {
  margin-bottom: 24px;
}

.hero-badge {
  display: inline-flex;
  align-items: center;
  min-height: 28px;
  padding: 0 10px;
  border-radius: 999px;
  background: rgba(255, 191, 71, 0.08);
  border: 1px solid rgba(255, 191, 71, 0.2);
  color: #ffbf47;
  font-size: 12px;
  font-weight: 700;
  letter-spacing: 0.02em;
  margin-bottom: 14px;
}

.logo {
  font-size: 34px;
  font-weight: 800;
  letter-spacing: -0.04em;
  color: #f8fafc;
  margin-bottom: 8px;
}

.subtitle {
  color: #94a3b8;
  font-size: 15px;
  line-height: 1.6;
  max-width: 620px;
}

.status-panel {
  display: flex;
  align-items: center;
  gap: 14px;
  padding: 16px 18px;
  border-radius: 18px;
  background: #151c24;
  border: 1px solid #283240;
  margin-bottom: 16px;
}

.status-panel.is-active {
  border-color: rgba(255, 191, 71, 0.35);
  box-shadow: 0 0 0 1px rgba(255, 191, 71, 0.08);
}

.status-panel.is-success {
  border-color: rgba(34, 197, 94, 0.35);
  background: rgba(34, 197, 94, 0.08);
}

.status-panel.is-error {
  border-color: rgba(239, 68, 68, 0.35);
  background: rgba(239, 68, 68, 0.08);
}

.status-dot {
  width: 12px;
  height: 12px;
  border-radius: 999px;
  background: #64748b;
  flex: 0 0 auto;
}

.status-panel.is-active .status-dot {
  background: #ffbf47;
  box-shadow: 0 0 14px rgba(255, 191, 71, 0.5);
}

.status-panel.is-success .status-dot {
  background: #22c55e;
  box-shadow: 0 0 14px rgba(34, 197, 94, 0.45);
}

.status-panel.is-error .status-dot {
  background: #ef4444;
  box-shadow: 0 0 14px rgba(239, 68, 68, 0.45);
}

.status-content {
  min-width: 0;
}

.status-label {
  font-size: 12px;
  color: #94a3b8;
  margin-bottom: 4px;
}

.status-text {
  font-size: 14px;
  font-weight: 700;
  color: #e8edf2;
  word-break: break-word;
}

.mode-panel {
  padding: 18px;
  border-radius: 18px;
  background: #121820;
  border: 1px solid #232c36;
  margin-bottom: 16px;
}

.mode-header {
  margin-bottom: 14px;
}

.mode-title {
  color: #f8fafc;
  font-size: 15px;
  font-weight: 800;
  margin-bottom: 6px;
}

.mode-subtitle {
  color: #94a3b8;
  font-size: 13px;
  line-height: 1.5;
}

.mode-options {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
}

.mode-option {
  display: block;
  cursor: pointer;
}

.mode-option input {
  display: none;
}

.mode-option-content {
  height: 100%;
  padding: 14px 14px;
  border-radius: 16px;
  border: 1px solid #2a3440;
  background: #151c24;
  transition: 0.18s ease;
}

.mode-option.active .mode-option-content {
  border-color: rgba(255, 191, 71, 0.45);
  background: rgba(255, 191, 71, 0.08);
  box-shadow: 0 0 0 1px rgba(255, 191, 71, 0.08);
}

.mode-option-title {
  font-size: 14px;
  font-weight: 800;
  color: #f8fafc;
  margin-bottom: 6px;
}

.mode-option-text {
  font-size: 12px;
  line-height: 1.5;
  color: #a8b4c3;
}

.warning-box {
  padding: 14px 16px;
  border-radius: 16px;
  background: rgba(59, 130, 246, 0.08);
  border: 1px solid rgba(59, 130, 246, 0.22);
  margin-bottom: 16px;
}

.warning-box.danger {
  background: rgba(239, 68, 68, 0.08);
  border-color: rgba(239, 68, 68, 0.22);
}

.warning-title {
  color: #dbeafe;
  font-size: 13px;
  font-weight: 700;
  margin-bottom: 6px;
}

.warning-box.danger .warning-title {
  color: #fecaca;
}

.warning-text {
  color: #d9e1ea;
  font-size: 13px;
  line-height: 1.55;
}

.progress-area {
  padding: 16px;
  border-radius: 16px;
  background: #121820;
  border: 1px solid #232c36;
  margin-bottom: 16px;
}

.progress-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
  margin-bottom: 10px;
}

.file-badge {
  max-width: 78%;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  display: inline-flex;
  align-items: center;
  min-height: 30px;
  padding: 0 12px;
  border-radius: 999px;
  font-family: "JetBrains Mono", "Fira Code", monospace;
  font-size: 12px;
  color: #ffbf47;
  background: rgba(255, 191, 71, 0.08);
  border: 1px solid rgba(255, 191, 71, 0.22);
}

.percent {
  font-size: 13px;
  font-weight: 700;
  color: #e8edf2;
}

.progress-track {
  height: 10px;
  width: 100%;
  background: #0d1319;
  border-radius: 999px;
  overflow: hidden;
  border: 1px solid #1e2731;
}

.progress-fill {
  height: 100%;
  width: 0%;
  border-radius: inherit;
  background: linear-gradient(90deg, #f59e0b 0%, #ffbf47 100%);
  box-shadow: 0 0 16px rgba(255, 191, 71, 0.28);
  transition: width 0.2s ease;
}

.actions {
  display: grid;
  grid-template-columns: 2fr 1fr;
  gap: 12px;
  margin-bottom: 16px;
}

.btn {
  min-height: 52px;
  border-radius: 14px;
  border: none;
  padding: 0 16px;
  font-size: 14px;
  font-weight: 800;
  letter-spacing: 0.01em;
  cursor: pointer;
  transition:
    transform 0.16s ease,
    box-shadow 0.16s ease,
    background 0.16s ease,
    border-color 0.16s ease,
    opacity 0.16s ease;
}

.btn:hover:not(:disabled) {
  transform: translateY(-1px);
}

.btn:active:not(:disabled) {
  transform: translateY(0);
}

.btn:disabled {
  opacity: 0.65;
  cursor: not-allowed;
}

.btn-primary {
  color: #111827;
  background: linear-gradient(180deg, #ffbf47 0%, #f59e0b 100%);
  box-shadow: 0 10px 24px rgba(245, 158, 11, 0.28);
}

.btn-primary:hover:not(:disabled) {
  filter: brightness(1.03);
}

.btn-secondary {
  color: #e5edf5;
  background: #151c24;
  border: 1px solid #2b3644;
}

.btn-secondary:hover:not(:disabled) {
  background: #1a222c;
  border-color: #364152;
}

.info-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 12px;
  margin-bottom: 18px;
}

.info-card {
  padding: 14px 16px;
  border-radius: 16px;
  background: #121820;
  border: 1px solid #232c36;
}

.info-label {
  font-size: 12px;
  color: #94a3b8;
  margin-bottom: 6px;
}

.info-value {
  font-size: 14px;
  font-weight: 700;
  color: #e8edf2;
}

.log-box {
  overflow: hidden;
  border-radius: 18px;
  background: #0f141a;
  border: 1px solid #232c36;
}

.log-header {
  min-height: 46px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  padding: 0 16px;
  background: #151c24;
  border-bottom: 1px solid #232c36;
  color: #cbd5e1;
  font-size: 12px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.04em;
}

.log-count {
  color: #94a3b8;
  font-weight: 600;
}

.log-list {
  max-height: 320px;
  overflow-y: auto;
  padding: 14px 16px;
  scroll-behavior: smooth;
}

.log-entry {
  font-family: "JetBrains Mono", "Fira Code", monospace;
  font-size: 12px;
  line-height: 1.65;
  color: #d6dee7;
  margin-bottom: 8px;
  white-space: pre-wrap;
  word-break: break-word;
}

.log-entry:last-child {
  margin-bottom: 0;
}

.log-list::-webkit-scrollbar {
  width: 8px;
}

.log-list::-webkit-scrollbar-track {
  background: transparent;
}

.log-list::-webkit-scrollbar-thumb {
  background: #334155;
  border-radius: 999px;
}

@media (max-width: 680px) {
  .card {
    padding: 20px;
    border-radius: 20px;
  }

  .logo {
    font-size: 28px;
  }

  .mode-options,
  .info-grid,
  .actions {
    grid-template-columns: 1fr;
  }

  .progress-header {
    align-items: flex-start;
    flex-direction: column;
  }

  .file-badge {
    max-width: 100%;
  }
}
</style>
