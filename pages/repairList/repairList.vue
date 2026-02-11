<template>
  <view class="container">
    <!-- 1. 顶部导航 (固定) -->
    <navbar :username="username" @logout="logout" />

    <!-- 2. 内容区域 (Flex 布局) -->
    <view class="content">
      
      <!-- A. 表头 (放在 scroll-view 外面，所以固定不动) -->
      <view class="table-header-wrapper">
        <view class="header-cell col-status">进度</view>
        <view class="header-cell col-doc">报修单号</view>
        <view class="header-cell col-name">资产名称</view>
        <view class="header-cell col-desc">故障描述</view>
      </view>

      <!-- B. 列表内容 (scroll-view 占据剩余空间，内部滚动) -->
      <scroll-view scroll-y class="table-body">
        <view 
          class="table-row" 
          v-for="(item, index) in assetsList" 
          :key="index"
          @click="navigateToDetail(item)"
        >
          <view class="cell col-status">{{ item.进度 || '-' }}</view>
          <view class="cell col-doc">{{ item.报修单号 || '-' }}</view>
          <view class="cell col-name">{{ item.资产名称 || '-' }}</view>
          <view class="cell col-desc">{{ item.故障描述 || '-' }}</view>
        </view>

        <!-- 状态提示 -->
        <view v-if="loading" class="state-container">
          <text class="state-text">加载中...</text>
        </view>

        <view v-if="!assetsList.length && !loading && !error" class="state-container">
          <text class="state-text">暂无数据</text>
        </view>

        <view v-if="error" class="state-container">
          <text class="error-text">加载失败</text>
          <button class="refresh-btn" @click="fetchAssets(true)">刷新</button>
        </view>
        
        <!-- 底部垫高 (防止最后一条数据贴底) -->
        <view style="height: 30rpx;"></view>
      </scroll-view>
    </view>
  </view>
</template>

<script>
	
import Navbar from '@/components/Navbar.vue'

export default {
	components:{
		Navbar
	},
  data() {
    return {
		username:'',
      assetsList: [],
      pagesize: 10,
      currentPage: 1,
      hasMoreData: true,
      loading: false,
      error: false,
	  moduleType: ''
    };
  },
  onLoad(options) {
	  if (options && options.moduleType) {
	      this.moduleType = options.moduleType;
	    }
	  // 当页面加载时尝试从本地存储获取用户名
	  const userInfo = uni.getStorageSync('userInfo');
	  if (userInfo && userInfo.username) {
	    this.username = userInfo.username;
	  };
    this.fetchAssets();
  },
  methods: {
    fetchAssets(reset = false) {
      if (reset) {
        this.assetsList = [];
        this.currentPage = 1;
        this.hasMoreData = true;
        this.error = false; // 清除错误状态
      }

      const userInfo = uni.getStorageSync('userInfo');
      console.log(userInfo);

      if (!userInfo.token || !userInfo.connid) {
        uni.showToast({
          title: '请先登录',
          icon: 'none'
        });
        return;
      }

      this.loading = true;

      uni.request({
        url: 'http://13.94.38.44:8000/AssetsRepair/GetMainList',
        method: 'POST',
        header: {
          'content-type': 'application/json'
        },
        data: JSON.stringify({
          connid: userInfo.connid,
          token: userInfo.token,
		  moduleType: this.moduleType
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
            this.loading = false;
            this.error = true; // 设置错误状态
            return;
          }

          console.log(result);

          if (!result.IsError && result.list) {
            if (Array.isArray(result.list) && result.list.length > 0) {
              this.assetsList = reset ? result.list : [...this.assetsList, ...result.list];
              this.currentPage++;
            } else {
              this.hasMoreData = false;
            }
          } else {
            uni.showToast({
              title: '获取资产信息失败',
              icon: 'none'
            });
            this.error = true; // 设置错误状态
          }
        },
        fail: () => {
          uni.showToast({
            title: '请求失败，请检查网络连接',
            icon: 'none'
          });
          this.error = true; // 设置错误状态
        },
        complete: () => {
          this.loading = false;
        }
      });
    },
    // 跳转报修明细和维修明细页面
    navigateToDetail(item) {
      if (item.进度) {
        uni.navigateTo({
          url: `/pages/baoxiuMIngxi/baoxiuMIngxi?docnum=${item.报修单号}`
        });
      } 
	  // else {
   //      uni.navigateTo({
   //        url: `/pages/weixiuMingxi/weixiuMingxi?docnum=${item.报修单号}`
   //      });
   //    }
    },
	logout (){
	  uni.removeStorageSync('userInfo'); // 修改为移除整个userInfo缓存
	  uni.redirectTo({
	    url: '/pages/login/login'
	  });
	}
  }
};
</script>

<style scoped>
  /* --- 1. 全局容器 (关键：定高 + 禁止溢出) --- */
  .container {
    display: flex;
    flex-direction: column;
    height: 100vh; /* 占满整个屏幕高度 */
    background-color: #f5f7fa;
    overflow: hidden; /* 防止整个页面出现滚动条 */
  }

  /* --- 2. 内容区域 (关键：Flex 纵向布局) --- */
  .content {
    flex: 1; /* 占据 Navbar 以下的所有空间 */
    display: flex;
    flex-direction: column; /* 让表头和列表上下排列 */
    width: 100%;
    margin: 0 auto;
    background-color: #fff;
    border-top: 1rpx solid #e4e7ed;
    overflow: hidden; /* 关键：限制内容不溢出 content */
  }

  /* --- 3. 表头样式 (固定) --- */
  .table-header-wrapper {
    /* 不设置 position: fixed，因为它在 flex 容器里自然就是顶部的 */
    flex-shrink: 0; /* 关键：禁止被压缩 */
    display: flex;
    width: 100%;
    background-color: #f5f7fa;
    border-bottom: 1rpx solid #e4e7ed;
    box-sizing: border-box;
  }

  .header-cell {
    padding: 24rpx 10rpx;
    font-size: 28rpx;
    font-weight: 600;
    color: #606266;
    text-align: center;
    border-right: 1rpx solid #ebeef5;
    box-sizing: border-box;
    display: flex;
    align-items: center;
    justify-content: center;
  }
  
  .header-cell:last-child {
    border-right: none;
  }

  /* --- 4. 表格主体 (关键：滚动区域) --- */
  .table-body {
    flex: 1;   /* 关键：自动占据剩余高度 */
    height: 0; /* 关键：配合 flex:1 使用，强制启用内部滚动 */
    width: 100%;
  }

  .table-row {
    display: flex;
    width: 100%;
    border-bottom: 1rpx solid #ebeef5;
  }

  /* 斑马纹 */
  .table-row:nth-child(even) {
    background-color: #fafafa;
  }

  .cell {
    padding: 20rpx 10rpx;
    font-size: 26rpx;
    color: #333;
    text-align: center;
    display: flex;
    align-items: center;
    justify-content: center;
    word-break: break-all;
    border-right: 1rpx solid #ebeef5;
    box-sizing: border-box;
  }
  
  .cell:last-child {
    border-right: none;
  }

  /* --- 列宽比例设置 (表头和内容必须一致) --- */
  .col-status { flex: 1.0; } /* 进度 */
  .col-doc    { flex: 2.1; } /* 单号 */
  .col-name   { flex: 1.3; } /* 名称 */
  .col-desc   { flex: 1.9; } /* 描述 */

  /* --- 状态提示 --- */
  .state-container {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    padding: 60rpx 0;
  }

  .state-text {
    font-size: 28rpx;
    color: #909399;
  }
  
  .error-text {
    font-size: 28rpx;
    color: #f56c6c;
    margin-bottom: 20rpx;
  }

  .refresh-btn {
    padding: 0 40rpx;
    height: 60rpx;
    line-height: 60rpx;
    background-color: #409eff;
    color: white;
    font-size: 26rpx;
    border-radius: 30rpx;
  }
</style>



