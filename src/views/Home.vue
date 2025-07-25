<template>
  <el-card class="home-card" shadow="hover">
    <div class="logo-container">
      <img src="@/assets/images/logo.png" alt="Logo" class="logo" />
      <h2>BlueArchiveGM 控制台</h2>
      <p class="description">欢迎使用 BlueArchiveGM 控制台，请配置服务器连接信息</p>
    </div>

    <div class="form-container">
      <!-- Lite 开关 -->
      <div class="input-container">
        <el-switch
          v-model="isLite"
          active-text="Lite 版本"
          inactive-text="标准 版本"
          active-color="#13ce66"
          inactive-color="#ff4949"
        />
      </div>

      <div class="input-container">
        <el-input
          v-model="serverAddress"
          placeholder="办公室地址（例：http://127.0.0.1:5000）"
          clearable
        >
          <template #prefix>
            <el-icon><Link /></el-icon>
          </template>
        </el-input>
      </div>

      <!-- 仅标准版本显示 Key 输入 -->
      <div class="input-container" v-if="!isLite">
        <el-input v-model="serverAuthKey" placeholder="老师的认证密钥" show-password clearable>
          <template #prefix>
            <el-icon><Key /></el-icon>
          </template>
        </el-input>
      </div>

      <div class="action-group">
        <el-button
          type="primary"
          @click="saveConfig"
          :loading="saving"
          class="save-button"
          :disabled="!isFormValid"
        >
          <el-icon class="icon"><Upload /></el-icon>
          {{ hasSaved ? '更新配置' : '保存配置' }}
        </el-button>

        <el-button
          type="success"
          @click="manualUpdate"
          :loading="updating"
          class="update-button"
          :disabled="!isFormValid || !hasSaved"
        >
          <el-icon class="icon"><Refresh /></el-icon>
          {{ updating ? '更新中...' : '更新服务器状态' }}
        </el-button>
      </div>
    </div>

    <transition name="fade-slide" mode="out-in">
      <!-- 错误状态 -->
      <div v-if="serverError" key="error">
        <el-card class="server-status-card">
          <h3 class="server-title">服务离线</h3>
          <div class="offline-status">
            <p>错误原因：{{ serverError }}</p>
          </div>
          <div class="update-time">最后更新：{{ lastUpdate || '--:--:--' }}</div>
        </el-card>
      </div>

      <!-- 正常状态 -->
      <div v-else-if="hasData" key="content">
        <el-card class="server-status-card">
          <h3 class="server-title">服务器状态（{{ autoUpdateStatus }}）</h3>
          <div class="timer-bar-container">
            <div class="timer-bar" :key="progressKey"></div>
            <span class="timer-text">{{ countdown }}s</span>
          </div>

          <el-skeleton :loading="loading" animated :rows="6">
            <el-row :gutter="20">
              <el-col :span="12" v-for="item in metrics" :key="item.label">
                <div class="metric">
                  <span class="metric-label">{{ item.label }}：</span>
                  <span class="metric-value">{{ item.value }}</span>
                </div>
              </el-col>
            </el-row>
            <div class="update-time">最后更新：{{ lastUpdate || '--:--:--' }}</div>
          </el-skeleton>
        </el-card>
      </div>

      <!-- 无数据 -->
      <el-card v-else key="empty" class="server-status-card empty-placeholder">
        <div class="empty-text">暂无服务器状态数据</div>
      </el-card>
    </transition>
  </el-card>
</template>

<script>
import axios from 'axios'
import { Key, Link, Refresh, Upload } from '@element-plus/icons-vue'

export default {
  name: 'Home',
  components: { Key, Link, Refresh, Upload },
  data() {
    return {
      isLite: localStorage.getItem('isLite') === 'false',
      serverAddress: localStorage.getItem('serverAddress') || '',
      serverAuthKey: localStorage.getItem('serverAuthKey') || '',
      hasSaved: !!localStorage.getItem('serverAddress'),
      saving: false,
      updating: false,
      serverData: null,
      serverError: '',
      autoUpdateTimer: null,
      autoUpdateErrorShown: false,
      lastUpdate: null,
      isPageVisible: true,
      loading: false,
      countdown: 30,
      timerInterval: null,
      progressKey: 0,
      updateQueue: null,
    }
  },
  computed: {
    isFormValid() {
      return this.serverAddress.startsWith('http')
    },
    autoUpdateStatus() {
      return this.isPageVisible ? '每30秒自动同步' : '后台暂停更新'
    },
    hasData() {
      return !!this.serverData || this.loading
    },
    metrics() {
      if (!this.serverData) return []
      return [
        { label: '在线的老师数量', value: this.serverData.playerNum },
        { label: 'TPS', value: this.serverData.tps },
        { label: 'RT', value: this.serverData.rt },
        { label: '客户端版本', value: this.serverData.clientVersion },
        { label: 'API版本', value: this.serverData.apiVersion },
        { label: '服务器版本', value: this.serverData.serverVersion },
        { label: '提交 ID', value: this.serverData.commit || '无' },
        { label: 'CPU 占用率', value: this.serverData.cpuOc },
        { label: '内存占用', value: this.serverData.memoryOc },
        { label: 'BAPS 内存占用', value: this.serverData.baPsMemoryOc },
      ]
    },
  },
  methods: {
    async saveConfig() {
      if (!this.serverAddress.startsWith('http')) {
        this.$message.error('地址需包含 http/https 协议头')
        return
      }
      this.saving = true
      try {
        localStorage.setItem('isLite', this.isLite)
        localStorage.setItem('serverAddress', this.serverAddress)
        localStorage.setItem('serverAuthKey', this.serverAuthKey)
        if (!this.hasSaved) {
          this.hasSaved = true
          const success = await this.manualUpdate(true)
          if (!success) {
            this.$message.error('无法连接到服务器，请检查配置')
          } else {
            this.startAutoUpdate()
            this.$message.success('配置保存成功')
          }
        } else {
          const success = await this.manualUpdate(true)
          if (!success) {
            this.$message.error('无法连接到服务器，请检查配置')
          } else {
            this.restartAutoUpdate()
            this.$message.success('配置更新成功')
          }
        }
      } catch (error) {
        this.$message.error('操作失败: ' + error.message)
      } finally {
        this.saving = false
      }
    },
    async manualUpdate(silent = false) {
      if (this.updating) return false
      this.loading = true
      this.updating = true
      try {
        await new Promise((resolve) => setTimeout(resolve, 300))
        const headers = {}
        if (!this.isLite && this.serverAuthKey) headers.Authorization = this.serverAuthKey
        const response = await axios.get(`${this.serverAddress}/cdq/api?cmd=ping`, {
          headers,
          timeout: 5000,
        })
        if (response.status === 200 && response.data.code === 0) {
          const pd = JSON.parse(response.data.msg)
          this.applyDataUpdate(pd)
          this.serverError = ''
          if (!silent) this.$message.success('状态更新成功')
          this.autoUpdateErrorShown = false
          return true
        } else {
          this.handleUpdateError(new Error(response.data.message || '服务器返回错误'), silent)
          return false
        }
      } catch (error) {
        this.handleUpdateError(error, silent)
        return false
      } finally {
        this.loading = false
        this.updating = false
      }
    },
    async silentUpdate() {
      try {
        const headers = {}
        if (!this.isLite && this.serverAuthKey) headers.Authorization = this.serverAuthKey
        const response = await axios.get(`${this.serverAddress}/cdq/api?cmd=ping`, {
          headers,
          timeout: 5000,
        })
        if (response.status === 200 && response.data.code === 0) {
          const pd = JSON.parse(response.data.msg)
          this.applyDataUpdate(pd)
          this.serverError = ''
          this.autoUpdateErrorShown = false
        }
      } catch (error) {
        if (!this.autoUpdateErrorShown) {
          this.$message.warning(`自动同步失败: ${error.message}`)
          this.autoUpdateErrorShown = true
        }
        this.handleUpdateError(error, true)
      }
    },
    applyDataUpdate(parsedData) {
      this.serverData = {
        playerNum: parsedData.playerNum || 0,
        tps: parsedData.tps || 0,
        rt: parsedData.rt || '0s',
        clientVersion: parsedData.clientVersion || '未知',
        serverVersion: parsedData.serverVersion || '未知',
        apiVersion: parsedData.apiVersion || '未知',
        commit: parsedData.commit || '',
        cpuOc: parsedData.cpuOc ? parsedData.cpuOc.toFixed(2) + '%' : '未知',
        memoryOc: parsedData.memoryOc || '未知',
        baPsMemoryOc: parsedData.baPsMemoryOc || '未知',
      }
      this.lastUpdate = this.getCurrentTime()
      this.clearCountdown()
      this.progressKey++
      this.countdown = 30
      this.startCountdown()
    },
    handleUpdateError(error, silent) {
      let message = ''
      if (error.response && error.response.status === 401) {
        message = '老师！您貌似没有认证密钥哦，请检查首页认证密钥是否输入正确！'
      } else {
        message = '状态更新失败: '
        if (error.response) message += `服务器错误 (${error.response.status})`
        else if (error.request) message += '无法连接到服务器'
        else message += error.message
      }
      this.serverError = message
      this.clearCountdown()
      if (!silent) this.$message.error(message)
    },
    startAutoUpdate() {
      this.clearAutoUpdate()
      if (!this.hasSaved) return
      this.updateQueue = setTimeout(() => {
        if (!this.isPageVisible) return
        this.autoUpdateTimer = setInterval(async () => {
          if (document.visibilityState === 'visible') await this.silentUpdate()
        }, 30000)
        this.silentUpdate()
      }, 500)
    },
    restartAutoUpdate() {
      this.clearAutoUpdate()
      this.startAutoUpdate()
    },
    clearAutoUpdate() {
      clearTimeout(this.updateQueue)
      if (this.autoUpdateTimer) clearInterval(this.autoUpdateTimer)
    },
    clearCountdown() {
      if (this.timerInterval) clearInterval(this.timerInterval)
    },
    startCountdown() {
      if (this.timerInterval) return
      this.timerInterval = setInterval(() => {
        if (this.countdown > 0) this.countdown--
        else {
          this.countdown = 30
          this.progressKey++
        }
      }, 1000)
    },
    getCurrentTime() {
      const now = new Date()
      return `${now.toLocaleDateString()} ${now.toLocaleTimeString()}`
    },
    handleVisibilityChange() {
      this.isPageVisible = document.visibilityState === 'visible'
      if (this.isPageVisible) setTimeout(this.startAutoUpdate, 300)
      else this.clearAutoUpdate()
    },
  },
  mounted() {
    document.addEventListener('visibilitychange', this.handleVisibilityChange)
    if (this.hasSaved) this.handleVisibilityChange()
  },
  beforeUnmount() {
    document.removeEventListener('visibilitychange', this.handleVisibilityChange)
    this.clearAutoUpdate()
    this.clearCountdown()
  },
}
</script>

<style scoped>
.home-card {
  max-width: 680px;
  margin: 1rem auto;
  padding: 2rem;
  border-radius: 16px;
  background: rgba(255, 255, 255, 0.45);
  backdrop-filter: blur(24px) saturate(140%);
  -webkit-backdrop-filter: blur(24px) saturate(140%);
  border: 1px solid rgba(255, 255, 255, 0.3);
  box-shadow:
    0 12px 40px -12px rgba(0, 0, 0, 0.12),
    0 4px 24px -4px rgba(0, 0, 0, 0.08),
    inset 0 0 12px rgba(255, 255, 255, 0.4);
  transition: all 0.3s ease;
  overflow: hidden;
  animation: fadeIn 0.3s ease-out both;
}
.home-card:hover {
  transform: translateY(-2px);
  box-shadow:
    0 16px 48px -12px rgba(0, 0, 0, 0.16),
    0 6px 32px -4px rgba(0, 0, 0, 0.12),
    inset 0 0 16px rgba(255, 255, 255, 0.5);
}
.logo-container {
  text-align: center;
  margin-bottom: 1.5rem;
}
.logo {
  width: 370px;
  height: 90px;
  margin-bottom: 1rem;
  object-fit: contain;
}
.description {
  color: #4a5568;
  margin-top: 0.5rem;
}
.form-container {
  margin-bottom: 2rem;
}
.input-container {
  margin: 1.2rem 0;
}
.action-group {
  margin-top: 2rem;
  display: flex;
  gap: 12px;
  align-items: center;
}
.save-button,
.update-button {
  min-width: 120px;
  padding: 0 20px;
  flex-shrink: 0;
  transition: transform 0.3s;
}
.update-button {
  flex: 1;
  min-width: 160px;
}
.server-status-card {
  margin-top: 2rem;
  padding: 1.5rem;
  border-radius: 16px;
  background: rgba(255, 255, 255, 0.4);
  backdrop-filter: blur(24px) saturate(140%);
  -webkit-backdrop-filter: blur(24px) saturate(140%);
  border: 1px solid rgba(255, 255, 255, 0.3);
  box-shadow:
    0 12px 40px -12px rgba(0, 0, 0, 0.12),
    0 4px 24px -4px rgba(0, 0, 0, 0.08),
    inset 0 0 12px rgba(255, 255, 255, 0.4);
  transition: all 0.3s ease;
  overflow: hidden;
}
.server-status-card:hover {
  transform: translateY(-2px);
  box-shadow:
    0 16px 48px -12px rgba(0, 0, 0, 0.16),
    0 6px 32px -4px rgba(0, 0, 0, 0.12),
    inset 0 0 16px rgba(255, 255, 255, 0.5);
}
.server-title {
  margin: 0 0 1.2rem;
  font-size: 1.1rem;
  font-weight: 600;
  color: #2c3e50;
  text-align: center;
  padding-bottom: 12px;
  border-bottom: 1px solid #e2e8f0;
}
.timer-bar-container {
  position: relative;
  height: 4px;
  background: #e2e8f0;
  border-radius: 2px;
  overflow: hidden;
  margin: 8px auto 16px;
  width: 80%;
}
.timer-bar {
  height: 100%;
  background: linear-gradient(90deg, #4facfe 0%, #00f2fe 100%);
  animation: progressAnimation 30s linear forwards;
}
@keyframes progressAnimation {
  from {
    width: 0%;
  }
  to {
    width: 100%;
  }
}
.timer-text {
  position: absolute;
  top: -24px;
  right: 0;
  font-size: 0.8em;
  color: #4a5568;
}
.metric {
  display: flex;
  justify-content: space-between;
  padding: 6px 0;
}
.metric-label {
  font-weight: 500;
  color: #4a5568;
}
.metric-value {
  font-weight: 600;
}
.update-time {
  margin-top: 1.5rem;
  text-align: center;
  color: #666;
  font-size: 0.9em;
}
.empty-placeholder {
  text-align: center;
  color: #999;
  padding: 2rem;
}
.offline-status {
  text-align: center;
  color: #f56c6c;
  font-size: 1rem;
  padding: 1rem 0;
}
.fade-slide-enter-active,
.fade-slide-leave-active {
  transition: all 0.3s cubic-bezier(0.55, 0, 0.1, 1);
}
.fade-slide-enter-from,
.fade-slide-leave-to {
  opacity: 0;
  transform: translateY(10px);
}
@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(16px) scale(0.98);
  }
  to {
    opacity: 1;
    transform: translateY(0) scale(1);
  }
}
@media (max-width: 480px) {
  .home-card {
    margin: 1rem;
    padding: 1.5rem;
  }
  .action-group {
    flex-direction: column;
  }
  .save-button,
  .update-button {
    width: 100%;
    min-width: unset;
  }
}
</style>
