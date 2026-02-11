
<template>
  <view class="container">
    <navbar :username="username" @logout="logout" />
    
    <!-- 顶部固定 Tabs -->
    <view class="tabs-wrapper">
      <view class="tabs">
        <view 
          :class="['tab', { active: currentTab === 'baoxiu' }]" 
          @click="switchTab('baoxiu')"
        >
          <text>报修明细</text>
          <view class="active-line" v-if="currentTab === 'baoxiu'"></view>
        </view>
        <view 
          :class="['tab', { active: currentTab === 'weixiu' }]" 
          @click="switchTab('weixiu')"
        >
          <text>维修明细</text>
          <view class="active-line" v-if="currentTab === 'weixiu'"></view>
        </view>
      </view>
    </view>

    <!-- 内容区域 - 使用 scroll-view 优化滚动体验 -->
    <scroll-view scroll-y class="scroll-content">
      
      <!-- 报修明细 -->
      <view v-if="currentTab === 'baoxiu'" class="content-wrapper fade-in">
        <!-- 基础信息卡片 -->
        <view class="card">
          <view class="card-header">基础信息</view>
          
          <view class="detail-row">
            <text class="label">报修单号</text>
            <text class="value selectable">{{ detail.docnum || '-' }}</text>
          </view>
          <view class="detail-row">
            <text class="label">资产编号</text>
            <text class="value selectable">{{ detail.AssetsCode || '-' }}</text>
          </view>
          <view class="detail-row">
            <text class="label">资产名称</text>
            <text class="value highlight">{{ detail.AssetsName || '-' }}</text>
          </view>
          <view class="detail-row">
            <text class="label">使用部门</text>
            <text class="value">{{ detail.UserDep || '-' }}</text>
          </view>
          <view class="detail-row">
            <text class="label">资产类型</text>
            <text class="value">{{ detail.AssetsType || '-' }}</text>
          </view>
		  <view class="detail-row no-border">
		    <text class="label">故障日期</text>
		    <text class="value">{{ detail.FaultDate || '-' }}</text>
		  </view>
          <view class="detail-row no-border">
            <text class="label">希望完工日期</text>
            <text class="value">{{ detail.FinishDate || '-' }}</text>
          </view>
        </view>

        <!-- 故障描述与图片卡片 -->
        <view class="card">
          <view class="card-header">故障详情</view>
          
          <view class="detail-block">
            <text class="label-block">故障描述</text>
            <view class="value-box">
              <text>{{ detail.Discription || '暂无描述' }}</text>
            </view>
          </view>

          <view class="detail-block">
            <text class="label-block">相关照片</text>
            <view v-if="detail.FilesList && detail.FilesList.length > 0" class="image-grid">
              <view v-for="(file, fileIndex) in detail.FilesList" :key="fileIndex" class="image-item">
                <image 
                  :src="file" 
                  mode="aspectFill" 
                  class="img" 
                  @click="openImageViewer(file)" 
                />
              </view>
            </view>
            <view v-else class="no-data">暂无相关图片</view>
          </view>
        </view>
      </view>

      <!-- 维修明细 -->
      <view v-if="currentTab === 'weixiu'" class="content-wrapper fade-in">
        <view class="card">
          <view class="card-header">维修单信息</view>
          
          <view class="detail-row">
            <text class="label">维修单号</text>
            <text class="value selectable">{{ detail.wxdocnum || '-' }}</text>
          </view>
          <view class="detail-row">
            <text class="label">预计金额</text>
            <text class="value price">{{ detail.Amount || '0' }}</text>
          </view>
          <view class="detail-row no-border">
            <text class="label">币别</text>
            <text class="value">{{ detail.Currency || 'RMB' }}</text>
          </view>
        </view>

        <view class="card">
          <view class="card-header">维修方案</view>
          
          <view class="detail-block">
            <text class="label-block">原因分析</text>
            <view class="value-box">
              <text>{{ detail.Reason || '暂无分析' }}</text>
            </view>
          </view>
          
          <view class="detail-block">
            <text class="label-block">维修方案</text>
            <view class="value-box">
              <text>{{ detail.SchemeTxt || '暂无方案' }}</text>
            </view>
          </view>
        </view>
      </view>
      
      <!-- 底部留白 -->
      <view style="height: 40rpx;"></view>
    </scroll-view>

    <!-- 图片预览组件 -->
    <image-viewer
      :visible="isImageViewerVisible"
      :current-image="selectedImageUrl"
      :on-close="closeImageViewer"
    ></image-viewer>
  </view>
</template>

<script>
import Navbar from '@/components/Navbar.vue';
import ImageViewer from '@/components/ImageViewer.vue';

export default {
  components: {
    Navbar,
	ImageViewer
  },
  data() {
    return {
      username: '',
      detail: {},
      currentTab: 'baoxiu', // 默认选中的标签
	  isImageViewerVisible: false,
	  selectedImageUrl: '',
    };
  },
  onLoad(options) {
    const userInfo = uni.getStorageSync('userInfo');
    if (userInfo && userInfo.username) {
      this.username = userInfo.username;
    }
    this.fetchDetail(options.docnum); // 加载报修明细数据
    this.fetchWeixiuDetail(options.docnum); // 预加载维修明细数据
  },
  methods: {
	  closeImageViewer() {
	     this.isImageViewerVisible = false;
	   },
	   openImageViewer(url) {
	     this.selectedImageUrl = url;
	     this.isImageViewerVisible = true;
	   },
	
    logout() {
      uni.removeStorageSync('userInfo'); // 修改为移除整个userInfo缓存
      uni.redirectTo({
        url: '/pages/login/login'
      });
    },
    fetchDetail(docnum) {
      const userInfo = uni.getStorageSync('userInfo');

      if (!userInfo.token || !userInfo.connid) {
        uni.showToast({
          title: '请先登录',
          icon: 'none'
        });
        return;
      }

      uni.request({
        url: 'http://13.94.38.44:8000/AssetsRepair/GetMainListDetail',
        method: 'POST',
        header: {
          'content-type': 'application/json'
        },
        data: JSON.stringify({
          connid: userInfo.connid,
          token: userInfo.token,
          docnum: docnum
        }),
        success: (response) => {
          let result;
          try {
            result = typeof response.data === 'string' ? JSON.parse(response.data) : response.data;
			console.log(result)
          } catch (error) {
            console.error('解析响应数据失败:', error);
            uni.showToast({
              title: '解析数据失败',
              icon: 'none'
            });
            return;
          }

          if (!result.isError) {
            this.detail = { ...this.detail, ...result }; // 合并结果到detail对象
          } else {
            uni.showToast({
              title: '获取详细信息失败',
              icon: 'none'
            });
          }
        },
        fail: () => {
          uni.showToast({
            title: '请求失败，请检查网络连接',
            icon: 'none'
          });
        }
      });
    },
    fetchWeixiuDetail(docnum) {
      const userInfo = uni.getStorageSync('userInfo');

      if (!userInfo.token || !userInfo.connid) {
        uni.showToast({
          title: '请先登录',
          icon: 'none'
        });
        return;
      }

      uni.request({
        url: 'http://13.94.38.44:8000/AssetsRepair/GetMainListDetail', // 假设API相同，可以根据实际情况调整
        method: 'POST',
        header: {
          'content-type': 'application/json'
        },
        data: JSON.stringify({
          connid: userInfo.connid,
          token: userInfo.token,
          docnum: docnum
        }),
        success: (response) => {
          let result;
          try {
            result = typeof response.data === 'string' ? JSON.parse(response.data) : response.data;
          } catch (error) {
            console.error('解析响应数据失败:', error);
            uni.showToast({
              title: '解析数据失败',
              icon: 'none'
            });
            return;
          }

          if (!result.isError) {
            this.detail = { ...this.detail, ...result }; // 合并维修明细结果到detail对象
          } else {
            uni.showToast({
              title: '获取详细信息失败',
              icon: 'none'
            });
          }
        },
        fail: () => {
          uni.showToast({
            title: '请求失败，请检查网络连接',
            icon: 'none'
          });
        }
      });
    },
    switchTab(tab) {
      this.currentTab = tab; // 切换当前标签
    }
  }
};
</script>

<style scoped>
/* --- 全局布局 --- */
.container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  background-color: #f5f7fa; /* 浅灰底色 */
}

/* --- Tabs 样式 --- */
.tabs-wrapper {
  background-color: #fff;
  z-index: 10;
  box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.03);
}

.tabs {
  display: flex;
  justify-content: space-around;
  height: 88rpx;
}

.tab {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  font-size: 30rpx;
  color: #606266;
  position: relative;
  transition: all 0.3s;
}

.tab.active {
  color: #007aff;
  font-weight: bold;
  font-size: 32rpx;
}

.active-line {
  position: absolute;
  bottom: 8rpx;
  width: 40rpx;
  height: 6rpx;
  background-color: #007aff;
  border-radius: 3rpx;
}

/* --- 滚动区域 --- */
.scroll-content {
  flex: 1;
  height: 0;
  width: 100%;
}

.content-wrapper {
  padding: 24rpx;
}

/* --- 卡片通用样式 --- */
.card {
  background-color: #fff;
  border-radius: 16rpx;
  padding: 0 30rpx;
  margin-bottom: 24rpx;
  box-shadow: 0 2rpx 8rpx rgba(0,0,0,0.02);
}

.card-header {
  font-size: 30rpx;
  font-weight: bold;
  color: #333;
  padding: 24rpx 0;
  border-bottom: 1rpx solid #f0f0f0;
  margin-bottom: 10rpx;
}

/* --- 左右结构的详情行 --- */
.detail-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 24rpx 0;
  border-bottom: 1rpx solid #f9f9f9;
}

.no-border {
  border-bottom: none;
}

.label {
  font-size: 28rpx;
  color: #909399; /* 浅灰标签 */
  width: 180rpx;
}

.value {
  flex: 1;
  text-align: right;
  font-size: 28rpx;
  color: #333;
  line-height: 1.4;
  word-break: break-all;
}

.selectable {
  user-select: text;
}

.highlight {
  color: #007aff;
  font-weight: 500;
}

.price {
  color: #ff4d4f;
  font-weight: bold;
  font-family: Arial, sans-serif;
}

/* --- 上下结构的块级详情 (用于长文本) --- */
.detail-block {
  padding: 24rpx 0;
  border-bottom: 1rpx solid #f9f9f9;
}

.detail-block:last-child {
  border-bottom: none;
}

.label-block {
  display: block;
  font-size: 28rpx;
  color: #909399;
  margin-bottom: 16rpx;
}

.value-box {
  background-color: #f9fafe; /* 淡背景色块 */
  padding: 20rpx;
  border-radius: 8rpx;
  font-size: 28rpx;
  color: #333;
  line-height: 1.6;
  text-align: justify;
}

/* --- 图片区域 --- */
.image-grid {
  display: flex;
  flex-wrap: wrap;
  gap: 20rpx;
}

.image-item {
  width: 150rpx;
  height: 150rpx;
  border-radius: 8rpx;
  overflow: hidden;
  background-color: #f0f0f0;
}

.img {
  width: 100%;
  height: 100%;
}

.no-data {
  font-size: 26rpx;
  color: #c0c4cc;
  padding: 10rpx 0;
}

/* --- 简单动画 --- */
.fade-in {
  animation: fadeIn 0.3s ease-out;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(10rpx); }
  to { opacity: 1; transform: translateY(0); }
}
</style>