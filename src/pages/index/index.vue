<template>
  <view class="container">
    <!-- 状态栏 -->
    <view class="status-bar">
      <view class="status-item">
        <text class="status-label">连接状态:</text>
        <text :class="['status-value', isConnected ? 'connected' : 'disconnected']">
          {{ isConnected ? '已连接' : '未连接' }}
        </text>
      </view>
      <view class="status-item">
        <text class="status-label">设备名称:</text>
        <text class="status-value">{{ deviceName || '未连接设备' }}</text>
      </view>
      <view class="status-item" v-if="isConnected">
        <text class="status-label">采样频率:</text>
        <text class="status-value">{{ deviceMaxFrequency }}Hz</text>
      </view>
    </view>

    <!-- 心率数据显示 -->
    <view class="heart-rate-display">
      <view class="heart-rate-value">{{ currentHeartRate || '--' }}</view>
      <text class="heart-rate-unit">BPM</text>
      <view class="heart-rate-label">实时心率</view>
    </view>

    <!-- 心电图波形显示 -->
    <view class="ecg-container">
      <view class="ecg-title">心电图波形</view>
      <canvas
        canvas-id="ecgCanvas"
        class="ecg-canvas"
      ></canvas>
    </view>

    <!-- 统计信息 -->
    <view class="stats-container">
      <view class="stat-item">
        <text class="stat-label">平均心率</text>
        <text class="stat-value">{{ averageHeartRate || '--' }}</text>
      </view>
      <view class="stat-item">
        <text class="stat-label">最高心率</text>
        <text class="stat-value">{{ maxHeartRate || '--' }}</text>
      </view>
      <view class="stat-item">
        <text class="stat-label">最低心率</text>
        <text class="stat-value">{{ minHeartRate || '--' }}</text>
      </view>
      <view class="stat-item">
        <text class="stat-label">监测时长</text>
        <text class="stat-value">{{ monitoringTime }}</text>
      </view>
      <view class="stat-item" v-if="isConnected">
        <text class="stat-label">实时频率</text>
        <text class="stat-value">{{ currentFrequency.toFixed(1) }}Hz</text>
      </view>
      <view class="stat-item" v-if="isConnected">
        <text class="stat-label">数据包数</text>
        <text class="stat-value">{{ dataPacketCount }}</text>
      </view>
    </view>

    <!-- 操作按钮 -->
    <view class="action-buttons">
      <button
        v-if="!isConnected"
        class="action-btn primary"
        @click="scanDevices"
        :disabled="isScanning"
      >
        {{ isScanning ? '搜索中...' : '搜索设备' }}
      </button>
      <button
        v-if="isConnected"
        class="action-btn danger"
        @click="disconnectDevice"
      >
        断开连接
      </button>
      <button
        v-if="isConnected"
        class="action-btn"
        @click="clearData"
      >
        清除数据
      </button>
      <button
        class="action-btn"
        @click="viewHistory"
      >
        查看历史
      </button>
    </view>

    <!-- 设备列表弹窗 -->
    <view v-if="showDeviceList" class="device-modal">
      <cover-view class="device-modal-mask" @click="closeDeviceList"></cover-view>
      <cover-view class="device-list">
        <cover-view class="device-list-header">
          <cover-view class="device-list-title">选择设备</cover-view>
          <cover-view class="device-list-close" @click="closeDeviceList">×</cover-view>
        </cover-view>
        <cover-view class="device-scroll" scroll-y>
          <cover-view
            v-for="(device, index) in deviceList"
            :key="index"
            class="device-item"
            @click="connectDevice(device)"
          >
            <cover-view class="device-info">
              <cover-view class="device-name">{{ device.name || '未知设备' }}</cover-view>
              <cover-view class="device-id">{{ device.deviceId }}</cover-view>
            </cover-view>
            <cover-view class="device-connect-btn">连接</cover-view>
          </cover-view>
          <cover-view v-if="deviceList.length === 0" class="device-empty">
            <cover-view>未发现设备，请确保心率带已开启</cover-view>
          </cover-view>
        </cover-view>
      </cover-view>
    </view>
  </view>
</template>

<script>
export default {
  data() {
    return {
      // 蓝牙相关
      isConnected: false,
      isScanning: false,
      deviceName: '',
      deviceId: '',
      serviceId: '',
      characteristicId: '',
      deviceList: [],
      showDeviceList: false,
      
      // 频率探测相关
      deviceMaxFrequency: 1, // 设备支持的最大频率(Hz)
      currentFrequency: 0, // 当前实际频率
      dataPacketCount: 0, // 数据包计数
      frequencyTestResults: [], // 频率测试结果
      lastDataTime: null, // 上次数据时间
      dataIntervals: [], // 数据间隔记录
      
      // 心率数据
      currentHeartRate: null,
      heartRateHistory: [],
      ecgData: [], // ECG波形数据
      
      // 统计数据
      averageHeartRate: null,
      maxHeartRate: null,
      minHeartRate: null,
      monitoringTime: '00:00:00',
      startTime: null,
      monitoringTimer: null,
      
      // Canvas相关
      canvasWidth: 750,
      canvasHeight: 300,
      ctx: null,
      
      // 心率服务UUID（标准蓝牙心率服务）
      HEART_RATE_SERVICE_UUID: '0000180d-0000-1000-8000-00805f9b34fb',
      HEART_RATE_CHARACTERISTIC_UUID: '00002a37-0000-1000-8000-00805f9b34fb',
    }
  },
  onLoad() {
    this.initCanvas()
    this.initBluetooth()
    this.loadHistoryData()
  },
  onUnload() {
    this.cleanup()
  },
  methods: {
    // 初始化Canvas
    initCanvas() {
      this.canvasWidth = 750
      this.canvasHeight = 300
      this.ctx = uni.createCanvasContext('ecgCanvas', this)
      this.drawECG()
    },
    
    // 初始化蓝牙
    initBluetooth() {
      uni.openBluetoothAdapter({
        success: (res) => {
          console.log('蓝牙初始化成功', res)
        },
        fail: (err) => {
          console.error('蓝牙初始化失败', err)
          uni.showToast({
            title: '蓝牙初始化失败',
            icon: 'none'
          })
        }
      })
    },
    
    // 搜索设备
    scanDevices() {
      if (this.isScanning) return
      
      this.isScanning = true
      this.deviceList = []
      this.showDeviceList = true
      
      uni.startBluetoothDevicesDiscovery({
        services: [this.HEART_RATE_SERVICE_UUID],
        allowDuplicatesKey: false,
        success: (res) => {
          console.log('开始搜索设备', res)
        },
        fail: (err) => {
          console.error('搜索设备失败', err)
          this.isScanning = false
          uni.showToast({
            title: '搜索设备失败',
            icon: 'none'
          })
        }
      })
      
      // 监听设备发现
      uni.onBluetoothDeviceFound((res) => {
        res.devices.forEach(device => {
          if (!this.deviceList.find(d => d.deviceId === device.deviceId)) {
            this.deviceList.push(device)
          }
        })
      })
      
      // 10秒后停止搜索
      setTimeout(() => {
        uni.stopBluetoothDevicesDiscovery({
          success: () => {
            this.isScanning = false
            console.log('停止搜索设备')
          }
        })
      }, 10000)
    },
    
    // 连接设备
    async connectDevice(device) {
      uni.showLoading({ title: '连接中...' })
      
      const deviceId = device.deviceId
      this.deviceName = device.name || '心率带'
      
      try {
        // 创建连接
        await this.createConnection(deviceId)
        this.deviceId = deviceId
        this.isConnected = true
        
        // 发现服务
        await this.discoverServices(deviceId)
        
        // 开始监测
        this.startMonitoring()
        
      } catch (error) {
        console.error('连接失败', error)
        uni.hideLoading()
        uni.showToast({
          title: '连接失败',
          icon: 'none'
        })
      }
      
      // 监听连接状态
      uni.onBLEConnectionStateChange((res) => {
        if (!res.connected) {
          this.isConnected = false
          this.stopMonitoring()
          uni.showToast({
            title: '设备已断开',
            icon: 'none'
          })
        }
      })
      
      this.closeDeviceList()
    },
    
    // 创建连接
    createConnection(deviceId) {
      return new Promise((resolve, reject) => {
        uni.createBLEConnection({
          deviceId: deviceId,
          success: (res) => {
            console.log('连接成功', res)
            resolve(res)
          },
          fail: (err) => {
            reject(err)
          }
        })
      })
    },
    
    // 发现服务
    async discoverServices(deviceId) {
      try {
        const services = await this.getBLEDeviceServices(deviceId)
        console.log('发现服务', services)
        
        const service = services.find(s => 
          s.uuid.toLowerCase() === this.HEART_RATE_SERVICE_UUID.toLowerCase()
        )
        
        if (service) {
          this.serviceId = service.uuid
          await this.discoverCharacteristics(deviceId, service.uuid)
        } else {
          // 如果没有找到标准服务，尝试使用第一个服务
          if (services.length > 0) {
            this.serviceId = services[0].uuid
            await this.discoverCharacteristics(deviceId, services[0].uuid)
          } else {
            throw new Error('未找到可用服务')
          }
        }
      } catch (error) {
        console.error('发现服务失败', error)
        throw error
      }
    },
    
    // 获取设备服务
    getBLEDeviceServices(deviceId) {
      return new Promise((resolve, reject) => {
        uni.getBLEDeviceServices({
          deviceId: deviceId,
          success: (res) => resolve(res.services),
          fail: reject
        })
      })
    },
    
    // 发现特征值
    async discoverCharacteristics(deviceId, serviceId) {
      try {
        const characteristics = await this.getBLEDeviceCharacteristics(deviceId, serviceId)
        console.log('发现特征值', characteristics)
        
        const characteristic = characteristics.find(c => 
          c.properties.notify || c.properties.read || c.properties.indicate
        )
        
        if (characteristic) {
          this.characteristicId = characteristic.uuid
          console.log('选择的特征值属性:', characteristic.properties)
          
          // 探测设备频率能力
          await this.probeDeviceFrequency(deviceId, serviceId, characteristic.uuid, characteristic.properties)
          
        } else {
          throw new Error('未找到可用特征值')
        }
      } catch (error) {
        console.error('发现特征值失败', error)
        throw error
      }
    },
    
    // 获取设备特征值
    getBLEDeviceCharacteristics(deviceId, serviceId) {
      return new Promise((resolve, reject) => {
        uni.getBLEDeviceCharacteristics({
          deviceId: deviceId,
          serviceId: serviceId,
          success: (res) => resolve(res.characteristics),
          fail: reject
        })
      })
    },
    
    // 探测设备频率能力
    async probeDeviceFrequency(deviceId, serviceId, characteristicId, properties) {
      console.log('开始探测设备频率能力...')
      
      if (properties.notify || properties.indicate) {
        // 使用通知模式 - 设备主动推送数据
        await this.enableNotification(deviceId, serviceId, characteristicId)
        
        // 监听数据并分析频率
        this.setupFrequencyAnalysis()
        
      } else if (properties.read) {
        // 使用读取模式 - 需要主动轮询
        const maxFrequency = await this.testReadFrequency(deviceId, serviceId, characteristicId)
        this.deviceMaxFrequency = maxFrequency
        console.log(`设备支持的最大读取频率: ${maxFrequency}Hz`)
        
        // 使用探测到的频率开始读取
        this.startOptimizedReading(deviceId, serviceId, characteristicId, maxFrequency)
      }
      
      uni.hideLoading()
      uni.showToast({
        title: `设备频率: ${this.deviceMaxFrequency}Hz`,
        icon: 'success'
      })
    },
    
    // 启用通知
    enableNotification(deviceId, serviceId, characteristicId) {
      return new Promise((resolve, reject) => {
        uni.notifyBLECharacteristicValueChange({
          deviceId: deviceId,
          serviceId: serviceId,
          characteristicId: characteristicId,
          state: true,
          success: () => {
            console.log('启用通知成功')
            
            // 监听数据
            uni.onBLECharacteristicValueChange((res) => {
              this.handleHeartRateData(res.value)
            })
            
            // 对于通知模式，设备决定频率，我们设置为较高的期望值
            this.deviceMaxFrequency = 10 // 假设设备支持10Hz
            resolve()
          },
          fail: (err) => {
            console.error('启用通知失败', err)
            reject(err)
          }
        })
      })
    },
    
    // 测试读取频率
    async testReadFrequency(deviceId, serviceId, characteristicId) {
      console.log('开始频率测试...')
      const testResults = []
      const testDuration = 3000 // 测试3秒
      const startTime = Date.now()
      let testCount = 0
      
      return new Promise((resolve) => {
        const testInterval = setInterval(async () => {
          if (Date.now() - startTime >= testDuration) {
            clearInterval(testInterval)
            
            // 计算平均频率
            const avgInterval = testResults.reduce((a, b) => a + b, 0) / testResults.length
            const maxFrequency = Math.floor(1000 / avgInterval)
            
            console.log(`频率测试完成: ${testCount}次读取，平均间隔: ${avgInterval.toFixed(1)}ms, 最大频率: ${maxFrequency}Hz`)
            resolve(Math.min(maxFrequency, 50)) // 限制最大50Hz
          }
          
          const readStart = Date.now()
          try {
            await this.singleRead(deviceId, serviceId, characteristicId)
            const responseTime = Date.now() - readStart
            testResults.push(responseTime)
            testCount++
          } catch (error) {
            console.error('测试读取失败', error)
          }
        }, 20) // 初始测试间隔20ms
      })
    },
    
    // 单次读取
    singleRead(deviceId, serviceId, characteristicId) {
      return new Promise((resolve, reject) => {
        uni.readBLECharacteristicValue({
          deviceId,
          serviceId,
          characteristicId,
          success: resolve,
          fail: reject
        })
      })
    },
    
    // 设置频率分析
    setupFrequencyAnalysis() {
      this.dataIntervals = []
      this.lastDataTime = null
      this.dataPacketCount = 0
    },
    
    // 开始优化读取
    startOptimizedReading(deviceId, serviceId, characteristicId, maxFrequency) {
      console.log(`开始优化读取，频率: ${maxFrequency}Hz`)
      
      const readInterval = Math.max(20, Math.floor(1000 / maxFrequency)) // 最小间隔20ms
      let isReading = false
      
      const readData = async () => {
        if (!this.isConnected || isReading) return
        
        isReading = true
        try {
          await this.singleRead(deviceId, serviceId, characteristicId)
        } catch (error) {
          console.error('读取数据失败', error)
        }
        isReading = false
      }
      
      // 使用setInterval保持稳定频率
      this.readingTimer = setInterval(readData, readInterval)
      
      // 监听数据
      uni.onBLECharacteristicValueChange((res) => {
        this.handleHeartRateData(res.value)
      })
    },
    
    // 处理心率数据（增强频率统计）
    handleHeartRateData(buffer) {
      try {
        // 频率统计
        const now = Date.now()
        this.dataPacketCount++
        
        if (this.lastDataTime) {
          const interval = now - this.lastDataTime
          this.dataIntervals.push(interval)
          
          // 保持最近50个间隔记录
          if (this.dataIntervals.length > 50) {
            this.dataIntervals.shift()
          }
          
          // 计算实时频率
          const avgInterval = this.dataIntervals.reduce((a, b) => a + b) / this.dataIntervals.length
          this.currentFrequency = 1000 / avgInterval
        }
        this.lastDataTime = now
        
        // 解析心率数据
        const data = new Uint8Array(buffer)
        let heartRate = 0
        
        if (data.length >= 2) {
          const flags = data[0]
          if (flags & 0x01) {
            // 16位心率值
            heartRate = data[1] | (data[2] << 8)
          } else {
            // 8位心率值
            heartRate = data[1]
          }
        }
        
        if (heartRate > 0 && heartRate < 300) {
          this.currentHeartRate = heartRate
          
          // 添加到历史记录
          this.heartRateHistory.push({
            value: heartRate,
            timestamp: now
          })
          
          // 生成ECG波形数据
          this.generateECGData(heartRate)
          
          // 更新统计数据
          this.updateStatistics()
          
          // 保存数据
          this.saveData()
        }
      } catch (error) {
        console.error('解析心率数据失败', error)
      }
    },
    
    // 生成ECG波形数据（模拟心电图）
    generateECGData(heartRate) {
      // 根据设备频率调整采样率
      const samplesPerSecond = Math.max(50, Math.min(200, this.deviceMaxFrequency * 10))
      const bpm = heartRate
      const interval = 60 / bpm
      const samplesPerBeat = Math.max(10, interval * samplesPerSecond)
      
      const beatWaveform = []
      for (let i = 0; i < samplesPerBeat; i++) {
        const phase = (i / samplesPerBeat) * 2 * Math.PI
        let value = 0
        
        if (phase > 1.5 && phase < 2.0) {
          value = Math.sin((phase - 1.5) * 10) * 0.8
        } else if (phase > 0.2 && phase < 0.6) {
          value = Math.sin((phase - 0.2) * 5) * 0.3
        } else if (phase > 2.2 && phase < 2.8) {
          value = Math.sin((phase - 2.2) * 3) * 0.4
        }
        
        beatWaveform.push(value)
      }
      
      this.ecgData.push(...beatWaveform)
      
      const maxSamples = samplesPerSecond * 30
      if (this.ecgData.length > maxSamples) {
        this.ecgData = this.ecgData.slice(-maxSamples)
      }
      
      this.drawECG()
    },
    
    // 绘制心电图
    drawECG() {
      if (!this.ctx) return
      
      const ctx = this.ctx
      const width = this.canvasWidth
      const height = this.canvasHeight
      
      ctx.clearRect(0, 0, width, height)
      
      // 绘制网格
      ctx.setStrokeStyle('#333333')
      ctx.setLineWidth(1)
      
      for (let y = 0; y <= height; y += 50) {
        ctx.beginPath()
        ctx.moveTo(0, y)
        ctx.lineTo(width, y)
        ctx.stroke()
      }
      
      for (let x = 0; x <= width; x += 50) {
        ctx.beginPath()
        ctx.moveTo(x, 0)
        ctx.lineTo(x, height)
        ctx.stroke()
      }
      
      if (this.ecgData.length === 0) {
        ctx.draw()
        return
      }
      
      // 绘制ECG波形
      ctx.beginPath()
      ctx.setStrokeStyle('#00ff00')
      ctx.setLineWidth(2)
      
      const centerY = height / 2
      const amplitude = height * 0.3
      const dataLength = this.ecgData.length
      const samplesToShow = Math.min(dataLength, 2000)
      const startIndex = Math.max(0, dataLength - samplesToShow)
      
      const drawStep = Math.max(1, Math.floor(30 * (1 / this.deviceMaxFrequency)))
      const displayData = []
      
      for (let i = startIndex; i < dataLength; i += drawStep) {
        displayData.push(this.ecgData[i])
      }
      
      if (displayData.length === 0) {
        ctx.draw()
        return
      }
      
      const stepX = width / displayData.length
      let isFirstPoint = true
      
      for (let j = 0; j < displayData.length; j++) {
        const value = displayData[j]
        const x = j * stepX
        const y = centerY - (value * amplitude)
        
        if (isFirstPoint) {
          ctx.moveTo(x, y)
          isFirstPoint = false
        } else {
          ctx.lineTo(x, y)
        }
      }
      
      ctx.stroke()
      ctx.draw()
    },
    
    // 更新统计数据
    updateStatistics() {
      if (this.heartRateHistory.length === 0) return
      
      const values = this.heartRateHistory.map(h => h.value)
      this.averageHeartRate = Math.round(values.reduce((a, b) => a + b, 0) / values.length)
      this.maxHeartRate = Math.max(...values)
      this.minHeartRate = Math.min(...values)
    },
    
    // 开始监测
    startMonitoring() {
      this.startTime = Date.now()
      this.heartRateHistory = []
      this.ecgData = []
      this.setupFrequencyAnalysis()
      
      this.monitoringTimer = setInterval(() => {
        if (this.startTime) {
          const elapsed = Math.floor((Date.now() - this.startTime) / 1000)
          const hours = Math.floor(elapsed / 3600)
          const minutes = Math.floor((elapsed % 3600) / 60)
          const seconds = elapsed % 60
          
          this.monitoringTime = `${String(hours).padStart(2, '0')}:${String(minutes).padStart(2, '0')}:${String(seconds).padStart(2, '0')}`
        }
      }, 1000)
    },
    
    // 停止监测
    stopMonitoring() {
      if (this.monitoringTimer) {
        clearInterval(this.monitoringTimer)
        this.monitoringTimer = null
      }
      if (this.readingTimer) {
        clearInterval(this.readingTimer)
        this.readingTimer = null
      }
    },
    
    // 断开连接
    disconnectDevice() {
      if (this.deviceId) {
        uni.closeBLEConnection({
          deviceId: this.deviceId,
          success: () => {
            this.isConnected = false
            this.stopMonitoring()
            console.log('断开连接成功')
          }
        })
      }
    },
    
    // 关闭设备列表
    closeDeviceList() {
      this.showDeviceList = false
      this.isScanning = false
      uni.stopBluetoothDevicesDiscovery()
    },
    
    // 清除数据
    clearData() {
      uni.showModal({
        title: '确认清除',
        content: '确定要清除所有数据吗？',
        success: (res) => {
          if (res.confirm) {
            this.heartRateHistory = []
            this.ecgData = []
            this.currentHeartRate = null
            this.averageHeartRate = null
            this.maxHeartRate = null
            this.minHeartRate = null
            this.startTime = Date.now()
            this.monitoringTime = '00:00:00'
            this.dataPacketCount = 0
            this.currentFrequency = 0
            this.ecgData = []
            this.drawECG()
            uni.removeStorageSync('heartRateHistory')
            uni.showToast({
              title: '数据已清除',
              icon: 'success'
            })
          }
        }
      })
    },
    
    // 查看历史
    viewHistory() {
      uni.navigateTo({
        url: '/pages/history/history'
      })
    },
    
    // 保存数据
    saveData() {
      try {
        const now = Date.now()
        const oneDayAgo = now - 24 * 60 * 60 * 1000
        const recentData = this.heartRateHistory.filter(h => h.timestamp > oneDayAgo)
        uni.setStorageSync('heartRateHistory', recentData)
      } catch (error) {
        console.error('保存数据失败', error)
      }
    },
    
    // 加载历史数据
    loadHistoryData() {
      try {
        const data = uni.getStorageSync('heartRateHistory')
        if (data && Array.isArray(data)) {
          this.heartRateHistory = data
          this.updateStatistics()
        }
      } catch (error) {
        console.error('加载历史数据失败', error)
      }
    },
    
    // 清理资源
    cleanup() {
      this.stopMonitoring()
      if (this.deviceId) {
        uni.closeBLEConnection({
          deviceId: this.deviceId
        })
      }
      uni.closeBluetoothAdapter()
    }
  }
}
</script>

<style scoped>
.container {
  min-height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  padding: 20rpx;
}

.status-bar {
  background: rgba(255, 255, 255, 0.9);
  border-radius: 20rpx;
  padding: 30rpx;
  margin-bottom: 30rpx;
  display: flex;
  justify-content: space-between;
  flex-wrap: wrap;
}

.status-item {
  display: flex;
  flex-direction: column;
  margin-bottom: 10rpx;
  min-width: 30%;
}

.status-label {
  font-size: 24rpx;
  color: #666;
  margin-bottom: 10rpx;
}

.status-value {
  font-size: 28rpx;
  font-weight: bold;
}

.status-value.connected {
  color: #00ff00;
}

.status-value.disconnected {
  color: #ff0000;
}

.heart-rate-display {
  background: rgba(255, 255, 255, 0.95);
  border-radius: 30rpx;
  padding: 60rpx 40rpx;
  margin-bottom: 30rpx;
  text-align: center;
  box-shadow: 0 10rpx 30rpx rgba(0, 0, 0, 0.2);
}

.heart-rate-value {
  font-size: 120rpx;
  font-weight: bold;
  color: #ff4757;
  line-height: 1.2;
}

.heart-rate-unit {
  font-size: 36rpx;
  color: #666;
  margin-left: 20rpx;
}

.heart-rate-label {
  font-size: 28rpx;
  color: #999;
  margin-top: 20rpx;
}

.ecg-container {
  background: rgba(255, 255, 255, 0.95);
  border-radius: 20rpx;
  padding: 30rpx;
  margin-bottom: 30rpx;
  box-shadow: 0 10rpx 30rpx rgba(0, 0, 0, 0.2);
  position: relative;
  z-index: 1;
}

.ecg-title {
  font-size: 32rpx;
  font-weight: bold;
  color: #333;
  margin-bottom: 20rpx;
  text-align: center;
}

.ecg-canvas {
  width: 100%;
  height: 400rpx;
  background: #000;
  border-radius: 10rpx;
  position: relative;
  z-index: 1;
}

.stats-container {
  background: rgba(255, 255, 255, 0.95);
  border-radius: 20rpx;
  padding: 30rpx;
  margin-bottom: 30rpx;
  display: flex;
  flex-wrap: wrap;
  justify-content: space-between;
  box-shadow: 0 10rpx 30rpx rgba(0, 0, 0, 0.2);
}

.stat-item {
  width: 48%;
  text-align: center;
  padding: 20rpx 0;
  margin-bottom: 20rpx;
}

.stat-label {
  display: block;
  font-size: 24rpx;
  color: #666;
  margin-bottom: 10rpx;
}

.stat-value {
  display: block;
  font-size: 36rpx;
  font-weight: bold;
  color: #667eea;
}

.action-buttons {
  display: flex;
  flex-wrap: wrap;
  gap: 20rpx;
}

.action-btn {
  flex: 1;
  min-width: 200rpx;
  height: 80rpx;
  line-height: 80rpx;
  border-radius: 40rpx;
  font-size: 28rpx;
  border: none;
  background: rgba(255, 255, 255, 0.9);
  color: #333;
}

.action-btn.primary {
  background: #4CAF50;
  color: #fff;
}

.action-btn.danger {
  background: #f44336;
  color: #fff;
}

.action-btn[disabled] {
  opacity: 0.6;
}

/* 设备列表弹窗样式 */
.device-modal {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 9999;
}

.device-modal-mask {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
}

.device-list {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 80%;
  max-height: 70%;
  background: #fff;
  border-radius: 20rpx;
  overflow: hidden;
  z-index: 10000;
}

.device-list-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 30rpx;
  border-bottom: 1rpx solid #eee;
}

.device-list-title {
  font-size: 32rpx;
  font-weight: bold;
  color: #333;
}

.device-list-close {
  font-size: 50rpx;
  color: #999;
  line-height: 1;
}

.device-scroll {
  max-height: 600rpx;
}

.device-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 30rpx;
  border-bottom: 1rpx solid #eee;
  min-height: 120rpx;
  box-sizing: border-box;
}

.device-info {
  flex: 1;
  display: flex;
  flex-direction: column;
}

.device-name {
  font-size: 28rpx;
  color: #333;
  margin-bottom: 10rpx;
}

.device-id {
  font-size: 24rpx;
  color: #999;
}

.device-connect-btn {
  font-size: 26rpx;
  color: #fff;
  background: #5c6bc0;
  padding: 15rpx 30rpx;
  border-radius: 20rpx;
  margin-left: 20rpx;
  flex-shrink: 0;
}

.device-empty {
  padding: 60rpx 30rpx;
  text-align: center;
  color: #999;
  font-size: 28rpx;
}
</style>