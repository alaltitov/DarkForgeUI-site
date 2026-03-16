<script setup lang="ts">
import { ref } from 'vue'
import { ESPLoader, Transport, CustomReset } from 'esptool-js'

// --- Типизация ---
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
type FlashSizeValues = 'detect' | 'keep' | '256KB' | '512KB' | '1MB' | '2MB' | '4MB' | '8MB' | '16MB' | '32MB' | '64MB' | '128MB'

type FlashFile = {
  data: Uint8Array
  address: number
}

// --- Состояние UI ---
const isFlashing = ref(false)
const progress = ref(0)
const statusText = ref('Готово к прошивке')
const currentFileName = ref('')
const logLines = ref<string[]>([])

let esploader: ESPLoader | null = null
let transport: Transport | null = null // Сохраняем ссылку для блока finally

// --- Вспомогательные функции ---
function appendLog(message: string): void {
  logLines.value.push(message)
  console.log(message)
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

// --- Главная функция прошивки ---
async function connectAndFlash(): Promise<void> {
  try {
    const nav = navigator as Navigator & { serial?: { requestPort: () => Promise<unknown> } }
    if (!nav.serial) throw new Error('Web Serial API не поддерживается вашим браузером.')

    // Сброс состояния перед новой прошивкой
    isFlashing.value = true
    progress.value = 0
    currentFileName.value = ''
    logLines.value = []
    statusText.value = 'Подключение...'

    // Запрос порта и инициализация транспорта
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

    // Подключение к ROM-загрузчику
    const chip = await (esploader as any).main()
    appendLog(`Подключено к чипу: ${chip}`)

    // Чтение манифеста
    statusText.value = 'Чтение манифеста...'
    const manifest = await fetchJson<FlashManifest>('/DarkForgeUI-site/firmware/flash_args.json')

    // Загрузка всех бинарников в память
    statusText.value = 'Загрузка файлов...'
    const fileArray: FlashFile[] = await Promise.all(
      manifest.files.map(async (file) => ({
        data: await fetchBinary(file.path),
        address: file.address
      }))
    )

    // Старт записи во flash
    statusText.value = 'Идет прошивка...'
    appendLog('Запуск writeFlash...')

    await (esploader as any).writeFlash({
      fileArray,
      eraseAll: true,
      compress: true,
      flashMode: manifest.flash_mode as FlashModeValues,
      flashFreq: manifest.flash_freq as FlashFreqValues,
      flashSize: manifest.flash_size as FlashSizeValues,
      reportProgress: (fileIndex: number, written: number, total: number) => {
        if (manifest.files[fileIndex]) {
          currentFileName.value = manifest.files[fileIndex].path
        }
        progress.value = Math.round((written / total) * 100)
      }
    })

    // --- Процедура финализации и перезагрузки (Через CustomReset) ---
    appendLog('Прошивка завершена. Запуск CustomReset...')
    statusText.value = 'Перезагрузка устройства...'
    
    // Используем встроенный класс с нашей идеальной строкой таймингов
    // D0 (IO0=HIGH), R1 (EN=LOW), W200 (пауза 200мс), R0 (EN=HIGH), W1000 (пауза 1с)
    const resetStrategy = new CustomReset(transport, 'D0|R1|W200|R0|W1000')
    await resetStrategy.reset()

    // Безопасное отключение
    appendLog('Отключение от порта...')
    await transport.disconnect()
    
    // Обнуляем ссылки, чтобы finally не дублировал отключение
    transport = null
    esploader = null 

    // Финальное обновление UI
    progress.value = 100
    currentFileName.value = 'Готово!'
    statusText.value = 'Устройство успешно прошито и запущено! 🚀'
    
  } catch (error: unknown) {
    const message = error instanceof Error ? error.message : String(error)
    statusText.value = `Ошибка: ${message}`
    appendLog(`Ошибка: ${message}`)
    console.error(error)
  } finally {
    // Гарантированная очистка ресурсов при ошибках
    if (transport) {
      appendLog('Экстренное освобождение порта...')
      await transport.disconnect().catch(() => {})
      transport = null
    }
    esploader = null
    isFlashing.value = false
  }
}
</script>

<template>
  <div class="darkforge-container">
    <div class="card">
      <div class="logo">DarkForge UI</div>
      <p class="subtitle">ESP32-P4 Web Flasher</p>

      <p class="status">{{ statusText }}</p>

      <!-- Блок прогресса с именем файла -->
      <div v-if="progress > 0" class="progress-wrap">
        <div class="progress-header">
          <span class="file-name">{{ currentFileName || 'Подготовка...' }}</span>
          <span class="progress-text">{{ progress }}%</span>
        </div>
        <div class="progress-bg">
          <div class="progress-fill" :style="{ width: progress + '%' }"></div>
        </div>
      </div>

      <button :disabled="isFlashing" @click="connectAndFlash">
        {{ isFlashing ? 'Идет прошивка...' : 'Подключить и прошить' }}
      </button>

      <!-- Логи -->
      <div v-if="logLines.length > 0" class="log-panel">
        <div class="log-title">Лог</div>
        <div class="log-content">
          <div v-for="(line, index) in logLines" :key="index" class="log-line">
            {{ line }}
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style>
body { margin: 0; background-color: #0d1117; color: #c9d1d9; font-family: 'Inter', sans-serif; }
* { box-sizing: border-box; }

.darkforge-container { display: flex; justify-content: center; align-items: center; min-height: 100vh; padding: 24px; }
.card { background: #161b22; padding: 32px; border-radius: 16px; border: 1px solid #30363d; text-align: center; width: 100%; max-width: 520px; box-shadow: 0 12px 32px rgba(0, 0, 0, 0.45); }
.logo { font-size: 30px; font-weight: 800; color: #58a6ff; margin-bottom: 6px; letter-spacing: -0.5px; }
.subtitle { margin: 0 0 18px; color: #8b949e; font-size: 14px; }
.status { margin-bottom: 20px; font-size: 14px; color: #c9d1d9; min-height: 40px; display: flex; align-items: center; justify-content: center; text-align: center; }

.progress-wrap { margin-bottom: 24px; text-align: left; }
.progress-header { display: flex; justify-content: space-between; align-items: flex-end; margin-bottom: 8px; }
.file-name { font-size: 13px; color: #8b949e; font-family: monospace; }
.progress-text { font-size: 14px; color: #58a6ff; font-weight: 700; }

.progress-bg { background: #21262d; border-radius: 8px; height: 12px; overflow: hidden; }
.progress-fill { background: linear-gradient(90deg, #1f6feb, #58a6ff); height: 100%; transition: width 0.15s linear; border-radius: 8px; }

button { background: #238636; color: #ffffff; border: none; padding: 14px 18px; border-radius: 10px; font-size: 16px; font-weight: 700; cursor: pointer; width: 100%; transition: all 0.2s ease; }
button:hover:not(:disabled) { background: #2ea043; transform: translateY(-1px); }
button:disabled { background: #21262d; color: #6e7681; cursor: not-allowed; transform: none; }

.log-panel { margin-top: 22px; text-align: left; border: 1px solid #30363d; border-radius: 12px; overflow: hidden; background: #0d1117; }
.log-title { padding: 10px 12px; font-size: 13px; font-weight: 700; color: #c9d1d9; background: #161b22; border-bottom: 1px solid #30363d; }
.log-content { max-height: 320px; overflow: auto; padding: 10px 12px; }
.log-line { font-family: monospace; font-size: 12px; color: #8b949e; line-height: 1.5; word-break: break-word; margin-bottom: 6px; }
</style>
