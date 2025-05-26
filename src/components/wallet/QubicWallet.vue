<template>
  <div class="qubic-wallet">
    <div class="wallet-container">
      <div v-if="!isConnected" class="connect-section">
        <button @click="connectWallet" class="connect-button">
          Подключить QUBIC кошелек
        </button>
      </div>
      <div v-else class="wallet-info">
        <div class="address">
          <span>Адрес: {{ shortAddress }}</span>
          <button @click="copyAddress" class="copy-button">
            <i class="fas fa-copy"></i>
          </button>
        </div>
        <div class="balance">
          <span>Баланс: {{ balance }} QUBIC</span>
        </div>
        <button @click="disconnectWallet" class="disconnect-button">
          Отключить
        </button>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'QubicWallet',
  data() {
    return {
      isConnected: false,
      address: '',
      balance: '0',
    }
  },
  computed: {
    shortAddress() {
      if (!this.address) return ''
      return `${this.address.slice(0, 6)}...${this.address.slice(-4)}`
    }
  },
  methods: {
    async connectWallet() {
      try {
        // TODO: Реализовать подключение к QUBIC кошельку
        this.isConnected = true
        this.address = '0x123...' // Временный адрес для демонстрации
        this.balance = '1000'
      } catch (error) {
        console.error('Ошибка при подключении кошелька:', error)
      }
    },
    disconnectWallet() {
      this.isConnected = false
      this.address = ''
      this.balance = '0'
    },
    async copyAddress() {
      try {
        await navigator.clipboard.writeText(this.address)
        // TODO: Добавить уведомление об успешном копировании
      } catch (error) {
        console.error('Ошибка при копировании адреса:', error)
      }
    }
  }
}
</script>

<style scoped>
.qubic-wallet {
  padding: 1rem;
}

.wallet-container {
  background: rgba(255, 255, 255, 0.1);
  border-radius: 8px;
  padding: 1rem;
}

.connect-section {
  text-align: center;
}

.connect-button {
  background: #4CAF50;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 4px;
  cursor: pointer;
  font-size: 1rem;
}

.connect-button:hover {
  background: #45a049;
}

.address {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  margin-bottom: 0.5rem;
}

.copy-button {
  background: none;
  border: none;
  color: #666;
  cursor: pointer;
}

.copy-button:hover {
  color: #333;
}

.balance {
  margin-bottom: 1rem;
}

.disconnect-button {
  background: #f44336;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 4px;
  cursor: pointer;
}

.disconnect-button:hover {
  background: #da190b;
}
</style> 