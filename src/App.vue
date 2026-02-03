<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

interface GoldPrice {
  current: number
  open: number
  high: number
  low: number
  change: number
  changePercent: number
  updateTime: string
}

const goldPrice = ref<GoldPrice>({
  current: 0,
  open: 0,
  high: 0,
  low: 0,
  change: 0,
  changePercent: 0,
  updateTime: '--:--:--'
})

const loading = ref(true)
const error = ref('')
let timer: number | null = null

// 获取金价数据
async function fetchGoldPrice() {
  try {
    // 使用免费的金价API（实时人民币金价）
    const response = await fetch('https://api.pearktrue.cn/api/goldprice/')
    
    if (!response.ok) {
      throw new Error('网络请求失败')
    }
    
    const data = await response.json()
    console.log('API返回数据:', data)
    
    // 解析返回数据
    if (data && data.code === 200 && data.data) {
      // 找到黄金9999的数据
      const goldData = data.data.find((item: any) => 
        item.name === '黄金9999' || item.name === 'Au99.99' || item.name?.includes('9999')
      ) || data.data[0]
      
      console.log('解析的金价数据:', goldData)
      
      if (goldData) {
        // 根据实际API字段解析
        const current = parseFloat(goldData.buyprice) || parseFloat(goldData.recycleprice) || 0
        const open = parseFloat(goldData.openprice) || parseFloat(goldData.open) || parseFloat(goldData.todayopen) || parseFloat(goldData.minprice) || current
        const high = parseFloat(goldData.maxprice) || current
        const low = parseFloat(goldData.minprice) || current
        
        // 计算相对于今日开盘价的涨跌
        const change = parseFloat((current - open).toFixed(2))
        const changePercent = open > 0 ? parseFloat(((change / open) * 100).toFixed(2)) : 0
        
        console.log('解析结果:', { current, open, high, low, change, changePercent })
        
        goldPrice.value = {
          current,
          open,
          high,
          low,
          change,
          changePercent,
          updateTime: new Date().toLocaleTimeString('zh-CN')
        }
        loading.value = false
        error.value = ''
        return
      }
    }
    
    throw new Error('数据格式异常')
  } catch (e) {
    console.error('接口1失败:', e)
    try {
      await fetchFromJintou()
    } catch (e2) {
      console.error('所有接口失败:', e2)
      error.value = '获取数据失败'
      loading.value = false
    }
  }
}

// 备用：金投网数据
async function fetchFromJintou() {
  // 金投网实时报价页面数据接口
  const response = await fetch('https://flash.jintou.com/gold/sg/data_new.js?t=' + Date.now())
  
  if (!response.ok) {
    throw new Error('请求失败')
  }
  
  const text = await response.text()
  console.log('金投网返回:', text.substring(0, 200))
  
  // 尝试解析 JSONP 格式
  const jsonMatch = text.match(/\{[\s\S]*\}/)
  if (jsonMatch) {
    const data = JSON.parse(jsonMatch[0])
    // 查找 AU9999 数据
    if (data && data.data) {
      const au9999 = Object.values(data.data).find((item: any) => 
        item.name?.includes('9999') || item.symbol === 'AU9999'
      ) as any
      
      if (au9999) {
        const current = parseFloat(au9999.price) || 0
        const open = parseFloat(au9999.open) || parseFloat(au9999.yclose) || current
        const change = parseFloat((current - open).toFixed(2))
        const changePercent = open > 0 ? parseFloat(((change / open) * 100).toFixed(2)) : 0
        
        goldPrice.value = {
          current,
          open,
          high: parseFloat(au9999.high) || 0,
          low: parseFloat(au9999.low) || 0,
          change,
          changePercent,
          updateTime: new Date().toLocaleTimeString('zh-CN')
        }
        loading.value = false
        error.value = ''
        return
      }
    }
  }
  
  throw new Error('解析失败')
}

// 开始拖动窗口
async function startDrag() {
  try {
    const { getCurrentWindow } = await import('@tauri-apps/api/window')
    const appWindow = getCurrentWindow()
    await appWindow.startDragging()
  } catch (e) {
    console.log('拖动失败:', e)
  }
}

// 关闭窗口
async function closeWindow() {
  try {
    const { getCurrentWindow } = await import('@tauri-apps/api/window')
    const appWindow = getCurrentWindow()
    await appWindow.close()
  } catch (e) {
    console.log('关闭窗口失败:', e)
  }
}

onMounted(() => {
  fetchGoldPrice()
  // 每30秒更新一次
  timer = window.setInterval(fetchGoldPrice, 30000)
})

onUnmounted(() => {
  if (timer) {
    clearInterval(timer)
  }
})
</script>

<template>
  <div class="widget">
    <!-- 标题栏 - 可拖动 -->
    <div class="header" @mousedown="startDrag">
      <span class="title">🏆 实时金价</span>
      <button class="close-btn" @click.stop="closeWindow">×</button>
    </div>
    
    <!-- 价格显示 -->
    <div class="price-container">
      <div v-if="loading" class="loading">加载中...</div>
      <div v-else-if="error" class="error">{{ error }}</div>
      <template v-else>
        <div class="current-price" :class="{ up: goldPrice.change >= 0, down: goldPrice.change < 0 }">
          ¥{{ goldPrice.current.toFixed(2) }}<span class="unit">/g</span>
        </div>
        
        <div class="change-info" :class="{ up: goldPrice.change >= 0, down: goldPrice.change < 0 }">
          {{ goldPrice.change >= 0 ? '+' : '' }}{{ goldPrice.change.toFixed(2) }}
          ({{ goldPrice.change >= 0 ? '+' : '' }}{{ goldPrice.changePercent.toFixed(2) }}%)
        </div>
        
        <div class="high-low">
          <div class="high">
            <span class="label">▲ 最高</span>
            <span class="value">{{ goldPrice.high.toFixed(2) }}</span>
          </div>
          <div class="low">
            <span class="label">▼ 最低</span>
            <span class="value">{{ goldPrice.low.toFixed(2) }}</span>
          </div>
        </div>
        
        <div class="update-time">
          更新: {{ goldPrice.updateTime }}
        </div>
      </template>
    </div>
  </div>
</template>

<style scoped>
.widget {
  width: 220px;
  background: linear-gradient(135deg, rgba(30, 30, 40, 0.5), rgba(20, 20, 30, 0.5));
  border-radius: 16px;
  overflow: hidden;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.4);
  border: 1px solid rgba(255, 215, 0, 0.25);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px 14px;
  background: linear-gradient(90deg, rgba(255, 215, 0, 0.15), rgba(255, 165, 0, 0.1));
  border-bottom: 1px solid rgba(255, 215, 0, 0.1);
  cursor: move;
}

.title {
  font-size: 13px;
  font-weight: 600;
  color: #ffd700;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.3);
}

.close-btn {
  width: 20px;
  height: 20px;
  border: none;
  background: rgba(255, 255, 255, 0.1);
  color: #888;
  font-size: 16px;
  border-radius: 50%;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s;
}

.close-btn:hover {
  background: #e74c3c;
  color: white;
}

.price-container {
  padding: 16px;
  text-align: center;
}

.loading {
  color: #888;
  font-size: 14px;
  padding: 20px;
}

.error {
  color: #e74c3c;
  font-size: 14px;
  padding: 20px;
}

.current-price {
  font-size: 28px;
  font-weight: 700;
  color: #ffd700;
  margin-bottom: 4px;
  text-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
}

.current-price .unit {
  font-size: 14px;
  font-weight: 400;
  color: #999;
}

.change-info {
  font-size: 13px;
  margin-bottom: 14px;
}

.up {
  color: #e74c3c;
}

.down {
  color: #2ecc71;
}

.high-low {
  display: flex;
  justify-content: space-between;
  padding: 10px 0;
  border-top: 1px solid rgba(255, 255, 255, 0.1);
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.high, .low {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
}

.high .label {
  font-size: 11px;
  color: #e74c3c;
}

.low .label {
  font-size: 11px;
  color: #2ecc71;
}

.high .value, .low .value {
  font-size: 14px;
  font-weight: 600;
  color: #ddd;
}

.update-time {
  margin-top: 10px;
  font-size: 10px;
  color: #666;
}
</style>
