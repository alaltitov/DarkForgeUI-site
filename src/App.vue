<script setup lang="ts">
import { ref } from 'vue'
// Импортируем классы из esptool-js
import { ESPLoader, Transport } from 'esptool-js'

// Типизируем реактивные переменные
const isConnected = ref<boolean>(false)
const progress = ref<number>(0)
const statusText = ref<string>('Ожидание подключения...')

// Переменная для хранения экземпляра загрузчика
let esploader: any = null

// Функция для скачивания бинарника из папки public/firmware/
async function fetchBinary(url: string): Promise<ArrayBuffer> {
  const response = await fetch(url)
  if (!response.ok) throw new Error(`Не удалось загрузить файл: ${url}`)
  return await response.arrayBuffer()
}

// Главная асинхронная функция прошивки
async function connectAndFlash(): Promise<void> {
  try {
    // 1. Запрашиваем порт (используем any для обхода отсутствия типов Web Serial в базовом TS)
    const nav = navigator as any
    if (!nav.serial) {
      throw new Error('Web Serial API не поддерживается вашим браузером. Используйте Chrome/Edge.')
    }
    
    const port = await nav.serial.requestPort()
    const transport = new Transport(port)
    
    // 2. Инициализируем загрузчик
    esploader = new ESPLoader({
      transport,
      baudrate: 115200,
      terminal: {
        clean: () => {},
        writeLine: (data: string) => console.log(data),
        write: (data: string) => console.log(data)
      }
    } as any)


    statusText.value = 'Подключение к ESP32-P4...'
    await esploader.main()
    isConnected.value = true

    // 3. Загружаем бинарные файлы в память браузера
    statusText.value = 'Загрузка файлов прошивки...'
    const appBuffer = await fetchBinary('/DarkForgeUI-site/firmware/app.bin')
    // Пример для других разделов:
    // const spiffsBuffer = await fetchBinary('/DarkForgeUI-site/firmware/spiffs.bin')

    // 4. Формируем массив для записи
    const fileArray = [
      { data: new Uint8Array(appBuffer), address: 0x10000 },
      // { data: new Uint8Array(spiffsBuffer), address: 0x200000 }
    ]

    // 5. Начинаем прошивку
    statusText.value = 'Идет прошивка...'
    await esploader.write_flash({
      fileArray,
      flash_size: 'keep',
      flash_mode: 'keep',
      flash_freq: 'keep',
      erase_all: false,
      compress: true,
      reportProgress: (fileIndex: number, written: number, total: number) => {
        progress.value = Math.round((written / total) * 100)
      }
    })

    statusText.value = 'Успешно прошито! 🚀'
  } catch (error: any) {
    statusText.value = `Ошибка: ${error.message}`
    console.error(error)
  } finally {
    if (esploader) {
      await esploader.transport.disconnect()
      isConnected.value = false
    }
  }
}
</script>

<template>
  <div class="darkforge-container">
    <div class="card">
      <div class="logo">DarkForge UI</div>
      <p class="status">{{ statusText }}</p>
      
      <!-- Прогресс-бар -->
      <div class="progress-bg" v-if="progress > 0 && progress < 100">
        <div class="progress-fill" :style="{ width: progress + '%' }"></div>
      </div>
      <p class="progress-text" v-if="progress > 0 && progress < 100">{{ progress }}%</p>

      <button @click="connectAndFlash" :disabled="isConnected">
        {{ isConnected ? 'Идет процесс...' : 'Подключить и Прошить' }}
      </button>
    </div>
  </div>
</template>

<style>
/* Глобальный сброс и темный фон */
body {
  margin: 0;
  background-color: #0d1117;
  color: #c9d1d9;
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
}

.darkforge-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
}

.card {
  background: #161b22;
  padding: 40px;
  border-radius: 16px;
  border: 1px solid #30363d;
  text-align: center;
  width: 380px;
  box-shadow: 0 12px 32px rgba(0, 0, 0, 0.6);
}

.logo {
  font-size: 28px;
  font-weight: 800;
  color: #58a6ff;
  margin-bottom: 8px;
  letter-spacing: -0.5px;
}

.status {
  margin-bottom: 24px;
  font-size: 14px;
  color: #8b949e;
  min-height: 40px; /* Чтобы интерфейс не прыгал при смене текста */
}

button {
  background: #238636;
  color: #ffffff;
  border: none;
  padding: 14px 24px;
  border-radius: 8px;
  font-size: 16px;
  font-weight: 600;
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
  color: #484f58;
  cursor: not-allowed;
  transform: none;
}

.progress-bg {
  background: #21262d;
  border-radius: 6px;
  height: 10px;
  margin-bottom: 8px;
  overflow: hidden;
}

.progress-fill {
  background: #58a6ff;
  height: 100%;
  transition: width 0.1s linear;
  border-radius: 6px;
}

.progress-text {
  font-size: 13px;
  color: #58a6ff;
  margin-bottom: 16px;
  font-weight: 600;
}
</style>
