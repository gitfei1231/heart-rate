<template>
  <view class="container">
    <!-- 统计信息 -->
    <view class="stats-header">
      <view class="stat-row">
        <view class="stat-card">
          <text class="stat-label">总记录数</text>
          <text class="stat-value">{{ historyData.length }}</text>
        </view>
        <view class="stat-card">
          <text class="stat-label">平均心率</text>
          <text class="stat-value">{{ averageHeartRate || '--' }}</text>
        </view>
      </view>
      <view class="stat-row">
        <view class="stat-card">
          <text class="stat-label">最高心率</text>
          <text class="stat-value">{{ maxHeartRate || '--' }}</text>
        </view>
        <view class="stat-card">
          <text class="stat-label">最低心率</text>
          <text class="stat-value">{{ minHeartRate || '--' }}</text>
        </view>
      </view>
    </view>

    <!-- 操作栏 -->
    <view class="action-bar">
      <button class="export-btn" @click="exportToExcel">
        <text class="export-icon">📊</text>
        <text>导出Excel</text>
      </button>
      <button class="delete-btn" @click="clearAllData" v-if="historyData.length > 0">
        <text class="delete-icon">🗑️</text>
        <text>清空数据</text>
      </button>
    </view>

    <!-- 数据列表 -->
    <scroll-view class="data-list" scroll-y>
      <view
        v-for="(item, index) in historyData"
        :key="index"
        class="data-item"
      >
        <view class="data-time">
          <text class="time-text">{{ formatTime(item.timestamp) }}</text>
        </view>
        <view class="data-value">
          <text class="value-text">{{ item.value }}</text>
          <text class="value-unit">BPM</text>
        </view>
        <view class="data-status" :class="getStatusClass(item.value)">
          <text>{{ getStatusText(item.value) }}</text>
        </view>
      </view>
      <view v-if="historyData.length === 0" class="empty-state">
        <text class="empty-icon">📋</text>
        <text class="empty-text">暂无历史数据</text>
      </view>
    </scroll-view>
  </view>
</template>

<script>
import * as XLSX from 'xlsx'

export default {
  data() {
    return {
      historyData: [],
      averageHeartRate: null,
      maxHeartRate: null,
      minHeartRate: null,
    }
  },
  onLoad() {
    this.loadHistoryData()
  },
  onShow() {
    // 每次显示页面时重新加载数据
    this.loadHistoryData()
  },
  methods: {
    // 加载历史数据
    loadHistoryData() {
      try {
        const data = uni.getStorageSync('heartRateHistory')
        if (data && Array.isArray(data)) {
          // 按时间倒序排列
          this.historyData = data.sort((a, b) => b.timestamp - a.timestamp)
          this.calculateStatistics()
        }
      } catch (error) {
        console.error('加载历史数据失败', error)
      }
    },
    
    // 计算统计数据
    calculateStatistics() {
      if (this.historyData.length === 0) {
        this.averageHeartRate = null
        this.maxHeartRate = null
        this.minHeartRate = null
        return
      }
      
      const values = this.historyData.map(item => item.value)
      this.averageHeartRate = Math.round(
        values.reduce((a, b) => a + b, 0) / values.length
      )
      this.maxHeartRate = Math.max(...values)
      this.minHeartRate = Math.min(...values)
    },
    
    // 格式化时间
    formatTime(timestamp) {
      const date = new Date(timestamp)
      const year = date.getFullYear()
      const month = String(date.getMonth() + 1).padStart(2, '0')
      const day = String(date.getDate()).padStart(2, '0')
      const hours = String(date.getHours()).padStart(2, '0')
      const minutes = String(date.getMinutes()).padStart(2, '0')
      const seconds = String(date.getSeconds()).padStart(2, '0')
      
      return `${year}-${month}-${day} ${hours}:${minutes}:${seconds}`
    },
    
    // 获取状态类
    getStatusClass(value) {
      if (value < 60) return 'status-low'
      if (value > 100) return 'status-high'
      return 'status-normal'
    },
    
    // 获取状态文本
    getStatusText(value) {
      if (value >= 0 && value <= 103) return '恢复区'
      if (value >= 104 && value <= 120) return '燃脂区'
      if (value >= 121 && value <= 140) return '有氧区'
      if (value >= 141 && value <= 152) return '阈值区'
      if (value >= 153 && value <= 170) return '无氧区'
      return '心率过高'
    },
    
    // 导出Excel
    exportToExcel() {
      if (this.historyData.length === 0) {
        uni.showToast({
          title: '暂无数据可导出',
          icon: 'none'
        })
        return
      }
      
      uni.showLoading({ title: '正在导出...' })
      
      try {
        // 准备数据
        const excelData = [
          ['时间', '心率值(BPM)', '状态', '日期', '小时']
        ]
        
        this.historyData.forEach(item => {
          const date = new Date(item.timestamp)
          const dateStr = `${date.getFullYear()}-${String(date.getMonth() + 1).padStart(2, '0')}-${String(date.getDate()).padStart(2, '0')}`
          const timeStr = `${String(date.getHours()).padStart(2, '0')}:${String(date.getMinutes()).padStart(2, '0')}:${String(date.getSeconds()).padStart(2, '0')}`
          const status = this.getStatusText(item.value)
          
          excelData.push([
            timeStr,
            item.value,
            status,
            dateStr,
            date.getHours()
          ])
        })
        
        // 创建工作簿
        const ws = XLSX.utils.aoa_to_sheet(excelData)
        const wb = XLSX.utils.book_new()
        XLSX.utils.book_append_sheet(wb, ws, '心率数据')
        
        // 生成文件名
        const now = new Date()
        const fileName = `心率数据_${now.getFullYear()}${String(now.getMonth() + 1).padStart(2, '0')}${String(now.getDate()).padStart(2, '0')}_${String(now.getHours()).padStart(2, '0')}${String(now.getMinutes()).padStart(2, '0')}.xlsx`
        
        // 导出文件
        // #ifdef H5
        XLSX.writeFile(wb, fileName)
        uni.hideLoading()
        uni.showToast({
          title: '导出成功',
          icon: 'success'
        })
        // #endif
        
        // #ifdef MP-WEIXIN
        // 微信小程序：将Excel转换为CSV格式，更兼容
        const csv = XLSX.utils.sheet_to_csv(ws)
        const csvData = '\ufeff' + csv // 添加BOM以支持中文
        
        // 保存为CSV文件（微信小程序对CSV支持更好）
        const fs = uni.getFileSystemManager()
        const filePath = `${wx.env.USER_DATA_PATH}/${fileName.replace('.xlsx', '.csv')}`
        
        fs.writeFile({
          filePath: filePath,
          data: csvData,
          encoding: 'utf8',
          success: () => {
            uni.hideLoading()
            // 自动打开分享菜单
            setTimeout(() => {
              // 使用uni.share或直接调用微信分享
              // #ifdef MP-WEIXIN
              // 微信小程序使用button open-type="share"或调用分享API
              uni.showActionSheet({
                itemList: ['分享给好友'],
                success: (res) => {
                  if (res.tapIndex === 0) {
                    // 分享给好友
                    wx.shareFileMessage({
                      filePath: filePath,
                      fileName: fileName.replace('.xlsx', '.csv'),
                      success: () => {
                        console.log('分享成功')
                      },
                      fail: (err) => {
                        console.log('分享失败', err)
                        // 提示用户手动分享
                        uni.showModal({
                          title: '提示',
                          content: '请点击右上角菜单，选择"转发"分享给好友',
                          showCancel: false,
                          confirmText: '知道了'
                        })
                      }
                    })
                  }
                }
              })
              // #endif
            }, 500)
          },
          fail: (err) => {
            console.error('写入文件失败', err)
            // 如果失败，尝试使用数组缓冲区方式
            try {
              const wbout = XLSX.write(wb, { type: 'array', bookType: 'xlsx' })
              const base64 = uni.arrayBufferToBase64(wbout)
              const xlsxPath = `${wx.env.USER_DATA_PATH}/${fileName}`
              
              fs.writeFile({
                filePath: xlsxPath,
                data: base64,
                encoding: 'base64',
                success: () => {
                  uni.hideLoading()
                  // 尝试分享Excel文件
                  setTimeout(() => {
                    wx.shareFileMessage({
                      filePath: xlsxPath,
                      fileName: fileName,
                      success: () => {
                        console.log('分享成功')
                      },
                      fail: () => {
                        uni.showModal({
                          title: '提示',
                          content: '请点击右上角菜单，选择"转发"分享给好友',
                          showCancel: false,
                          confirmText: '知道了'
                        })
                      }
                    })
                  }, 500)
                },
                fail: () => {
                  uni.hideLoading()
                  uni.showToast({
                    title: '导出失败',
                    icon: 'none'
                  })
                }
              })
            } catch (e) {
              uni.hideLoading()
              uni.showToast({
                title: '导出失败',
                icon: 'none'
              })
            }
          }
        })
        // #endif
        
      } catch (error) {
        console.error('导出Excel失败', error)
        uni.hideLoading()
        uni.showToast({
          title: '导出失败',
          icon: 'none'
        })
      }
    },
    
    // 清空所有数据
    clearAllData() {
      uni.showModal({
        title: '确认清空',
        content: '确定要清空所有历史数据吗？此操作不可恢复',
        success: (res) => {
          if (res.confirm) {
            try {
              uni.removeStorageSync('heartRateHistory')
              this.historyData = []
              this.averageHeartRate = null
              this.maxHeartRate = null
              this.minHeartRate = null
              uni.showToast({
                title: '已清空',
                icon: 'success'
              })
            } catch (error) {
              console.error('清空数据失败', error)
              uni.showToast({
                title: '清空失败',
                icon: 'none'
              })
            }
          }
        }
      })
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

.stats-header {
  display: flex;
  flex-direction: column;
  gap: 20rpx;
  margin-bottom: 30rpx;
}

.stat-row {
  display: flex;
  gap: 20rpx;
}

.stat-card {
  flex: 1;
  width: calc(50% - 10rpx);
  background: rgba(255, 255, 255, 0.95);
  border-radius: 20rpx;
  padding: 30rpx;
  text-align: center;
  box-shadow: 0 10rpx 30rpx rgba(0, 0, 0, 0.2);
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

.action-bar {
  display: flex;
  gap: 20rpx;
  margin-bottom: 30rpx;
}

.export-btn,
.delete-btn {
  flex: 1;
  height: 80rpx;
  line-height: 80rpx;
  border-radius: 40rpx;
  font-size: 28rpx;
  border: none;
  background: rgba(255, 255, 255, 0.95);
  color: #333;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10rpx;
}

.export-btn {
  background: #4CAF50;
  color: #fff;
}

.delete-btn {
  background: #f44336;
  color: #fff;
}

.export-icon,
.delete-icon {
  font-size: 32rpx;
}

.data-list {
  height: calc(100vh - 400rpx);
  background: rgba(255, 255, 255, 0.95);
  border-radius: 20rpx;
  padding: 20rpx;
  box-shadow: 0 10rpx 30rpx rgba(0, 0, 0, 0.2);
}

.data-item {
  display: flex;
  align-items: center;
  padding: 30rpx 20rpx;
  border-bottom: 1rpx solid #eee;
}

.data-item:last-child {
  border-bottom: none;
}

.data-time {
  flex: 1;
}

.time-text {
  font-size: 26rpx;
  color: #666;
}

.data-value {
  flex: 1;
  text-align: center;
  display: flex;
  align-items: baseline;
  justify-content: center;
  gap: 10rpx;
}

.value-text {
  font-size: 40rpx;
  font-weight: bold;
  color: #ff4757;
}

.value-unit {
  font-size: 24rpx;
  color: #999;
}

.data-status {
  flex: 0 0 120rpx;
  text-align: center;
  padding: 10rpx 20rpx;
  border-radius: 20rpx;
  font-size: 24rpx;
}

.status-normal {
  background: #00ff00;
  color: #fff;
}

.status-high {
  background: #ff4757;
  color: #fff;
}

.status-low {
  background: #ffa502;
  color: #fff;
}

.empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 100rpx 0;
}

.empty-icon {
  font-size: 120rpx;
  margin-bottom: 30rpx;
}

.empty-text {
  font-size: 28rpx;
  color: #999;
}
</style>

