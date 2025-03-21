<template>
  <el-card class="function-card" shadow="hover">
    <h2>给予玩家单个物品</h2>
    <p>Tips：一次只能获取一个物品哦~</p>
    <el-form :model="form" label-width="100px">
      <el-form-item label="老师UID">
        <el-input
          v-model="form.uid"
          placeholder="请输入的老师UID"
          @input="form.uid = form.uid.replace(/\D/g, '')"
        >
          <template #prefix>
            <el-icon><User /></el-icon>
          </template>
        </el-input>
      </el-form-item>
      <el-form-item label="物品类型">
        <el-select v-model="form.t" placeholder="请选择物品类型">
          <el-option
            v-for="option in typeOptions"
            :key="option.value"
            :label="option.label"
            :value="option.value"
          />
        </el-select>
      </el-form-item>
      <el-form-item label="物品ID">
        <el-input
          v-model="form.id"
          placeholder="请输入物品ID"
          :disabled="!form.t"
          @input="form.id = form.id.replace(/\D/g, '')"
        >
          <template #prefix>
            <el-icon><Box /></el-icon>
          </template>
        </el-input>
      </el-form-item>
      <el-form-item label="数量">
        <el-input-number v-model="form.num" :min="1">
          <template #prefix>
            <el-icon><Coin /></el-icon>
          </template>
        </el-input-number>
      </el-form-item>
      <el-form-item>
        <el-button type="primary" @click="handleGive">提交</el-button>
      </el-form-item>
    </el-form>
    <el-alert
      v-if="response"
      title="响应"
      :type="responseType"
      :description="response"
      show-icon
      style="margin-top: 20px"
    ></el-alert>
    <el-collapse class="documentation">
      <el-collapse-item title="使用方法（必看！）">
        <ul class="guide-list">
          <li>提示：玩家不在线或未注册</li>
          <li><code>解决方法：注册或登录当前用户</code></li>
          <li>不知道物品ID？</li>
          <li><code>前往ID列表查看</code></li>
          <li>物品类型不知道？</li>
          <li>以下是物品类别：</li>
          <li><code>"Material": // 材料</code></li>
          <li><code>"Character": // 学生</code></li>
          <li><code>"Equipment": // 装备</code></li>
          <li><code>"Furniture": // 家具</code></li>
          <li><code>"Favor": // 礼物</code></li>
          <li><code>"Emblem": // 称号</code></li>
        </ul>
      </el-collapse-item>
    </el-collapse>
  </el-card>
</template>

<script>
import axios from 'axios'
import { User, Box, CollectionTag, Coin } from '@element-plus/icons-vue'

export default {
  name: 'Give',
  components: {
    User,
    Box,
    CollectionTag,
    Coin,
  },
  data() {
    return {
      form: {
        uid: '',
        id: '',
        t: '',
        num: 1,
      },
      response: '',
      responseType: '', // 成功时设置为 'success'，错误时设置为 'error'
      typeOptions: [
        // { value: 'None', label: '无' },
        // { value: 'Character', label: '学生' },
        // { value: 'Currency', label: '货币' },
        // { value: 'Equipment', label: '装备' },
        // { value: 'Item', label: '物品' },
        // { value: 'GachaGroup', label: '卡池' },
        // { value: 'Product', label: '商品' },
        // { value: 'Shop', label: '商店' },
        // { value: 'MemoryLobby', label: '记忆大厅' },
        // { value: 'AccountExp', label: '账号经验' },
        // { value: 'CharacterExp', label: '学生经验' },
        // { value: 'FavorExp', label: '好感经验' },
        // { value: 'TSS', label: 'TSS' },
        // { value: 'Furniture', label: '家具' },
        // { value: 'ShopRefresh', label: '商店刷新' },
        // { value: 'LocationExp', label: '地点经验' },
        // { value: 'Recipe', label: '配方' },
        // { value: 'CharacterWeapon', label: '学生武器' },
        // { value: 'ProductMonthly', label: '月度商品' },
        // { value: 'CharacterGear', label: '学生装备' },
        // { value: 'IdCardBackground', label: '资料背景' },
        // { value: 'Emblem', label: '徽章' },
        // { value: 'Sticker', label: '贴纸' },
        // { value: 'Costume', label: '服饰' },
        { value: 'Material', label: '材料' },
        { value: 'Character', label: '学生' },
        { value: 'Equipment', label: '装备' },
        { value: 'Furniture', label: '家具' },
        { value: 'Favor', label: '礼物' },
        { value: 'Emblem', label: '称号' },
      ],
    }
  },
  methods: {
    async handleGive() {
      const baseURL = localStorage.getItem('serverAddress')
      const authKey = localStorage.getItem('serverAuthKey')

      if (!baseURL) {
        this.$message.error('请先在首页保存服务器地址')
        return
      }

      try {
        const url = `${baseURL}/cdq/api?cmd=g&uid=${this.form.uid}&id=${this.form.id}&t=${this.form.t}&num=${this.form.num}`
        // 仅在 authKey 存在时添加 Authorization 请求头
        let headers = {}
        if (authKey) {
          headers.Authorization = authKey
        }
        const res = await axios.get(url, { headers })

        if (res.data.code === 0) {
          this.responseType = 'success'
          this.$message.success('物品授予成功！')
        } else {
          this.responseType = 'error'
          this.$message.error('操作失败：' + (res.data.message || '请查看响应获取具体错误'))
        }
        this.response = JSON.stringify(res.data, null, 2)
      } catch (error) {
        this.responseType = 'error'
        const errorMsg = error.response?.data?.message || error.message
        this.response = '请求错误：' + errorMsg
        this.$message.error('操作失败，请检查配置')
      }
    },
  },
}
</script>

<style scoped>
.function-card {
  max-width: 600px;
  margin: 40px auto;
  animation: fadeIn 0.6s cubic-bezier(0.23, 1, 0.32, 1);
  background: rgba(255, 255, 255, 0.86) !important;
  backdrop-filter: blur(24px) saturate(140%);
  -webkit-backdrop-filter: blur(24px) saturate(140%);
  border: 1px solid rgba(255, 255, 255, 0.3);
  border-radius: 16px;
  box-shadow:
    0 12px 40px -12px rgba(0, 0, 0, 0.12),
    0 4px 24px -4px rgba(0, 0, 0, 0.08),
    inset 0 0 12px rgba(255, 255, 255, 0.4);
  transition: all 0.3s ease;
  overflow: hidden;
}

.function-card:hover {
  transform: translateY(-2px);
  box-shadow:
    0 16px 48px -12px rgba(0, 0, 0, 0.16),
    0 6px 32px -4px rgba(0, 0, 0, 0.12),
    inset 0 0 16px rgba(255, 255, 255, 0.5);
}

:deep(h2) {
  color: #2c3e50 !important;
  font-weight: 600;
  margin-bottom: 24px;
  padding-bottom: 12px;
  position: relative;
}

:deep(h2::after) {
  content: '';
  position: absolute;
  bottom: 0;
  left: 0;
  width: 48px;
  height: 3px;
  background: linear-gradient(90deg, #4facfe 0%, #00f2fe 100%);
  border-radius: 2px;
}

:deep(.el-form-item__label) {
  color: #4a5568 !important;
  font-weight: 500 !important;
  letter-spacing: 0.5px;
}

:deep(.el-input__prefix) {
  left: 12px !important;
  color: #a0aec0 !important;
}

:deep(.el-input__inner:focus) {
  border-color: #4facfe !important;
  box-shadow: 0 0 0 2px rgba(79, 172, 254, 0.2) !important;
}

:deep(.el-button) {
  background: linear-gradient(135deg, #4facfe 0%, #00f2fe 100%) !important;
  border: none !important;
  border-radius: 8px !important;
  color: white !important;
  font-weight: 600;
  padding: 12px 32px !important;
  transition:
    transform 0.2s ease,
    box-shadow 0.2s ease;
}

:deep(.el-button:hover) {
  transform: translateY(-1px);
  box-shadow: 0 6px 16px -4px rgba(79, 172, 254, 0.4) !important;
}

:deep(.el-collapse) {
  border: none !important;
  margin-top: 20px;
}

:deep(.el-collapse-item__header) {
  color: #4a5568 !important;
  font-weight: 500 !important;
  padding-left: 28px !important;
  background: transparent !important;
}

:deep(.el-collapse-item__arrow) {
  left: 0 !important;
  color: #718096 !important;
}

.guide-list {
  line-height: 1.8;
  padding-left: 20px;
}

.guide-list li {
  margin: 8px 0;
}

.guide-list code {
  background: rgba(79, 172, 254, 0.1);
  color: #4facfe;
  padding: 2px 6px;
  border-radius: 4px;
  font-family: Monaco, Consolas, monospace;
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

@media (max-width: 640px) {
  .function-card {
    margin: 20px;
    border-radius: 12px;
  }

  :deep(h2) {
    font-size: 1.5rem;
  }
}
</style>
