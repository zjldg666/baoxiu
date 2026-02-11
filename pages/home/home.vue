<template>
	<view class="container">
		<navbar :username="username" @logout="logout" />

		<!-- 1. 中间：功能按钮九宫格 (flex: 1 占满中间空间) -->
		<scroll-view scroll-y class="content-area">
			<view class="grid-container">
				<!-- 第一行 -->
				<view class="grid-row">
					<view class="grid-card" @click="scanAndNavigate">
						<image class="card-icon" src="../../static/icon/SaomaBx.png"></image>
						<text class="card-title">扫码报修</text>
					</view>
					<view class="grid-card" @click="TomobileRepair">
						<image class="card-icon" src="../../static/icon/ShoujiBx.png"></image>
						<text class="card-title">手选报修</text>
					</view>
				</view>
				
				<!-- 第二行 -->
				<view class="grid-row">
					<view class="grid-card" @click="TorepairList">
						<image class="card-icon" src="../../static/icon/bxQingdan.png"></image>
						<text class="card-title">报修进度</text>
					</view>
					<view class="grid-card" @click="navigateTo('assetList')">
						<image class="card-icon" src="../../static/icon/ZichanQingdan.png"></image>
						<text class="card-title">{{ moduleType === 1 ? '设备清单' : '设施清单' }}</text>
					</view>
				</view>
				
				<!-- 第三行 -->
				<view class="grid-row">
					<view class="grid-card" @click="navigateTo('simpleReport')">
						<image class="card-icon" src="../../static/icon/EasyBaobiao.png"></image>
						<text class="card-title">简单报表</text>
					</view>
					<!-- 占位元素，保持布局整齐 -->
					<view class="grid-card invisible"></view> 
				</view>
			</view>
		</scroll-view>

		<!-- 2. 底部：模块切换 Tab (固定在底部) -->
		<view class="bottom-tab-container">
			<view class="segment-control">
				<view 
					class="segment-item" 
					:class="{ active: moduleType === 1 }" 
					@click="switchTab(1)"
				>
					<text class="segment-text">设备报修</text>
				</view>
				<view 
					class="segment-item" 
					:class="{ active: moduleType === 2 }" 
					@click="switchTab(2)"
				>
					<text class="segment-text">设施报修</text>
				</view>
			</view>
		</view>

	</view>
</template>

<script>
	import Navbar from '@/components/Navbar.vue'

	export default {
		components: {
			Navbar
		},
		data() {
			return {
				username: '',
				moduleType: 1, // 1代表设备报修，2代表设施报修
			}

		},
		onLoad() {
			// 当页面加载时尝试从本地存储获取用户名
			const userInfo = uni.getStorageSync('userInfo');
			if (userInfo && userInfo.username) {
				this.username = userInfo.username;
			}
		},
		methods: {

			switchTab(type) {
				this.moduleType = type;
			},
			// 扫码报修功能
			scanAndNavigate() {
				uni.scanCode({
					success: (res) => {
						const AssetsCode = res.result;
						console.log(AssetsCode);
						this.navigateToRepair(AssetsCode);
					},
					fail: () => {
						uni.showToast({
							title: '扫码失败，请重试',
							icon: 'none'
						});
					}
				});
			},

			navigateToRepair(AssetsCode) {
				const userInfo = uni.getStorageSync('userInfo');
				console.log(userInfo.connid);
				if (!userInfo.token || !userInfo.connid) {
					uni.showToast({
						title: '请先登录',
						icon: 'none'
					});
					return;
				}
				const requestData = {
					connid: userInfo.connid,
					token: userInfo.token,
					AssetsCode: AssetsCode,
					isEdit: true,
					moduleType: this.moduleType
				};

				// 2. 添加日志打印
				console.log('ShowNewInfo 接口请求参数:', requestData);

				uni.request({
					url: 'http://13.94.38.44:8000/AssetsRepair/ShowNewInfo',
					method: 'POST',
					header: {
						'content-type': 'application/json'
					},
					data: JSON.stringify({
						connid: userInfo.connid,
						token: userInfo.token,
						AssetsCode: AssetsCode,
						isEdit: true,
						moduleType: this.moduleType
					}),
					success: (response) => {
						const result = typeof response.data === 'string' ? JSON.parse(response.data) : response
							.data;
						console.log(result);
						console.log(result.AssetsCode);
						if (!result.IsError) {
							// 将资产信息作为参数传递给repair页面
							uni.navigateTo({
								url: `/pages/repair/repair?data=${JSON.stringify(result)}`
							});
						} else {
							uni.showToast({
								title: '获取资产信息失败',
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

			TomobileRepair() {
				uni.navigateTo({
					url: `/pages/mobileRepair/mobileRepair?moduleType=${this.moduleType}`
				});
			},

			TorepairList() {
				uni.navigateTo({
					url: `/pages/repairList/repairList?moduleType=${this.moduleType}`
				});
			},

			navigateTo(page) {
				uni.navigateTo({
					url: `/pages/${page}/${page}?moduleType=${this.moduleType}`
				});
			},
			logout() {
				uni.removeStorageSync('userInfo'); // 修改为移除整个userInfo缓存
				uni.redirectTo({
					url: '/pages/login/login'
				});
			}
		}
	}
</script>

<style scoped>
	/* --- 全局容器 --- */
	.container {
		display: flex;
		flex-direction: column;
		height: 100vh;
		background-color: #f7f8fa; /* 浅灰底色 */
		overflow: hidden;
	}

	/* --- 1. 中间九宫格区域 --- */
	.content-area {
		flex: 1; /* 占据剩余空间 */
		width: 100%;
		box-sizing: border-box;
	}

	.grid-container {
		padding: 40rpx 30rpx;
		display: flex;
		flex-direction: column;
		gap: 30rpx; /* 行间距 */
	}

	.grid-row {
		display: flex;
		justify-content: space-between;
		gap: 30rpx; /* 列间距 */
	}

	/* 卡片样式 */
	.grid-card {
		flex: 1;
		background-color: #fff;
		border-radius: 24rpx;
		/* 移除之前的 padding: 40rpx 20rpx; 改为更加紧凑 */
		padding: 30rpx 0; 
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		box-shadow: 0 8rpx 24rpx rgba(0,0,0,0.03);
		transition: transform 0.1s;
		/* 稍微减小最小高度，因为内容变少了 */
		min-height: 200rpx; 
	}

	.grid-card:active {
		transform: scale(0.96);
		background-color: #fafafa;
	}

	.invisible {
		visibility: hidden;
		pointer-events: none;
	}

	.card-icon {
		width: 80rpx;
		height: 80rpx;
		/* 增加一点底边距，让图标和文字分开 */
		margin-bottom: 24rpx;
	}

	.card-title {
		font-size: 32rpx; /* 字体稍微大一点，因为没有描述了 */
		font-weight: 600;
		color: #333;
		margin-bottom: 0; /* 不需要底边距了 */
	}

	/* --- 2. 底部 Tab 切换区 (固定) --- */
	.bottom-tab-container {
		flex-shrink: 0; /* 禁止被压缩 */
		background-color: #fff;
		padding: 20rpx 40rpx;
		/* 适配全面屏底部安全区 */
		padding-bottom: calc(30rpx + constant(safe-area-inset-bottom));
		padding-bottom: calc(30rpx + env(safe-area-inset-bottom));
		
		border-top-left-radius: 30rpx;
		border-top-right-radius: 30rpx;
		box-shadow: 0 -4rpx 20rpx rgba(0,0,0,0.05); /* 向上投影 */
		z-index: 10;
	}

	/* 分段控制器容器 */
	.segment-control {
		display: flex;
		background-color: #f0f2f5; /* 灰色背景条 */
		border-radius: 50rpx; /* 全胶囊圆角 */
		padding: 8rpx;
		position: relative;
		height: 88rpx;
		box-sizing: border-box;
	}

	/* 单个选项 */
	.segment-item {
		flex: 1;
		display: flex;
		justify-content: center;
		align-items: center;
		border-radius: 40rpx; /* 内层圆角 */
		transition: all 0.3s ease;
	}

	.segment-text {
		font-size: 28rpx;
		color: #606266;
		font-weight: 500;
	}

	/* 选中状态 */
	.segment-item.active {
		background-color: #fff;
		box-shadow: 0 4rpx 10rpx rgba(0,0,0,0.1);
	}

	.segment-item.active .segment-text {
		color: #007aff; /* 蓝色高亮 */
		font-weight: bold;
		font-size: 30rpx;
	}

</style>