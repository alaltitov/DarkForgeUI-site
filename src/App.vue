<script setup lang="ts">
import { ref } from 'vue'
import md5 from 'js-md5'
import { ESPLoader, Transport } from 'esptool-js'

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
type FlashFreqValues =
  | 'keep'
  | '80m'
  | '60m'
  | '48m'
  | '40m'
  | '30m'
  | '26m'
  | '24m'
  | '20m'
  | '16m'
  | '15m'
  | '12m'
type FlashSizeValues =
  | 'detect'
  | 'keep'
  | '256KB'
  | '512KB'
  | '1MB'
  | '2MB'
  | '2MB-c1'
  | '4MB'
  | '4MB-c1'
  | '8MB'
  | '16MB'
  | '32MB'
  | '64MB'
  | '128MB'

type FlashFile = {
  data: Uint8Array
  address: number
}

const isFlashing = ref<boolean>(false)
const progress = ref<number>(0)
const statusText = ref<string>('Готово к прошивке')
const logLines = ref<string[]>([])

let esploader: any = null

function appendLog(message: string): void {
  logLines.value.push(message)
  console.log(message)
}

async function fetchBinary(url: string): Promise<ArrayBuffer> {
  const response = await fetch(url)
  if (!response.ok) {
    throw new Error(`Не удалось загрузить бинарник: ${url}`)
  }
  return await response.arrayBuffer()
}

async function fetchJson<T>(url: string): Promise<T> {
  const response = await fetch(url)
  if (!response.ok) {
    throw new Error(`Не удалось загрузить JSON: ${url}`)
  }
  return (await response.json()) as T
}

function md5FromUint8Array(image: Uint8Array): string {
  let binary = ''
  const chunkSize = 0x8000

  for (let i = 0; i < image.length; i += chunkSize) {
    const chunk = image.subarray(i, i + chunkSize)
    binary += String.fromCharCode(...chunk)
  }

  return md5(binary)
}

async function connectAndFlash(): Promise<void> {
  try {
    const nav = navigator as any

    if (!nav.serial) {
      throw new Error('Web Serial API не поддерживается вашим браузером. Используйте Chrome или Edge.')
    }

    isFlashing.value = true
    progress.value = 0
    logLines.value = []
    statusText.value = 'Запрос доступа к serial-порту...'

    appendLog('Проверка Web Serial API...')
    appendLog('Запрос порта у пользователя...')

    const port = await nav.serial.requestPort()
    const transport = new Transport(port, true)

    esploader = new ESPLoader({
      transport,
      baudrate: 115200,
      terminal: {
        clean: () => {},
        writeLine: (data: string) => appendLog(data),
        write: (data: string) => appendLog(data)
      }
    } as any)

    statusText.value = 'Подключение к ESP32-P4...'
    appendLog('Инициализация ESPLoader...')
    const chip = await esploader.main()
    appendLog(`Подключено к чипу: ${chip}`)
    appendLog(`typeof esploader.writeFlash = ${typeof esploader.writeFlash}`)

    statusText.value = 'Загрузка плана прошивки...'
    appendLog('Загрузка flash_args.json...')

    const manifest = await fetchJson<FlashManifest>('/DarkForgeUI-site/firmware/flash_args.json')

    appendLog(`Flash mode: ${manifest.flash_mode}`)
    appendLog(`Flash freq: ${manifest.flash_freq}`)
    appendLog(`Flash size: ${manifest.flash_size}`)
    appendLog(`Количество файлов: ${manifest.files.length}`)

    statusText.value = 'Загрузка бинарников прошивки...'

    const fileArray: FlashFile[] = await Promise.all(
      manifest.files.map(async (file) => {
        appendLog(`Загрузка ${file.path} @ 0x${file.address.toString(16).toUpperCase()}`)
        const buffer = await fetchBinary(`/DarkForgeUI-site/firmware/${file.path}`)

        return {
          data: new Uint8Array(buffer),
          address: file.address
        }
      })
    )

    statusText.value = 'Идет прошивка...'
    appendLog('Начинается запись во flash...')

    await esploader.writeFlash({
      fileArray,
      eraseAll: true,
      compress: true,
      flashMode: manifest.flash_mode as FlashModeValues,
      flashFreq: manifest.flash_freq as FlashFreqValues,
      flashSize: manifest.flash_size as FlashSizeValues,
      reportProgress: (_fileIndex: number, written: number, total: number) => {
        progress.value = Math.round((written / total) * 100)
      },
      calculateMD5Hash: (image: Uint8Array) => {
        return md5FromUint8Array(image)
      }
    })

    appendLog('Запись завершена, выполняется after()...')
    await esploader.after()

    progress.value = 100
    statusText.value = 'Прошивка завершена! 🚀'
    appendLog('Прошивка успешно завершена.')
  } catch (error: any) {
    const message = error?.message || String(error)
    statusText.value = `Ошибка: ${message}`
    appendLog(`Ошибка: ${message}`)
    console.error(error)
  } finally {
    if (esploader?.transport) {
      try {
        appendLog('Отключение транспорта...')
        await esploader.transport.disconnect()
      } catch (disconnectError) {
        appendLog(`Ошибка отключения: ${String(disconnectError)}`)
      }
    }

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

      <div class="progress-wrap" v-if="progress > 0">
        <div class="progress-bg">
          <div class="progress-fill" :style="{ width: progress + '%' }"></div>
        </div>
        <p class="progress-text">{{ progress }}%</p>
      </div>

      <button @click="connectAndFlash" :disabled="isFlashing">
        {{ isFlashing ? 'Идет прошивка...' : 'Подключить и прошить' }}
      </button>

      <div class="log-panel" v-if="logLines.length > 0">
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
body {
  margin: 0;
  background-color: #0d1117;
  color: #c9d1d9;
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
}

* {
  box-sizing: border-box;
}

.darkforge-container {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
  padding: 24px;
}

.card {
  background: #161b22;
  padding: 32px;
  border-radius: 16px;
  border: 1px solid #30363d;
  text-align: center;
  width: 100%;
  max-width: 460px;
  box-shadow: 0 12px 32px rgba(0, 0, 0, 0.45);
}

.logo {
  font-size: 30px;
  font-weight: 800;
  color: #58a6ff;
  margin-bottom: 6px;
  letter-spacing: -0.5px;
}

.subtitle {
  margin: 0 0 18px;
  color: #8b949e;
  font-size: 14px;
}

.status {
  margin-bottom: 20px;
  font-size: 14px;
  color: #c9d1d9;
  min-height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.progress-wrap {
  margin-bottom: 18px;
}

.progress-bg {
  background: #21262d;
  border-radius: 8px;
  height: 12px;
  overflow: hidden;
}

.progress-fill {
  background: linear-gradient(90deg, #1f6feb, #58a6ff);
  height: 100%;
  transition: width 0.15s linear;
  border-radius: 8px;
}

.progress-text {
  font-size: 13px;
  color: #58a6ff;
  margin: 8px 0 0;
  font-weight: 700;
}

button {
  background: #238636;
  color: #ffffff;
  border: none;
  padding: 14px 18px;
  border-radius: 10px;
  font-size: 16px;
  font-weight: 700;
  cursor: pointer;
  width: 100%;
  transition: all 0.2s ease;
}

button:hover:not(:disabled) {
  background: #2ea043;
  transform: translateY(-1px);
}

button:disabled {
  background: #21262d;
  color: #6e7681;
  cursor: not-allowed;
  transform: none;
}

.log-panel {
  margin-top: 22px;
  text-align: left;
  border: 1px solid #30363d;
  border-radius: 12px;
  overflow: hidden;
  background: #0d1117;
}

.log-title {
  padding: 10px 12px;
  font-size: 13px;
  font-weight: 700;
  color: #c9d1d9;
  background: #161b22;
  border-bottom: 1px solid #30363d;
}

.log-content {
  max-height: 220px;
  overflow: auto;
  padding: 10px 12px;
}

.log-line {
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace;
  font-size: 12px;
  color: #8b949e;
  line-height: 1.5;
  word-break: break-word;
  margin-bottom: 6px;
}
</style>
