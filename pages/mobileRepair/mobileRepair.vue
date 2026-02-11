<template>
	<view class="container">
		<navbar :username="username" @logout="logout" />

		<view class="content">
			<!-- 1. 搜索框 (固定高度) -->
			<input type="text" v-model="searchQuery" placeholder="请输入搜索内容" @input="handleSearchInput"
				class="search-input" />

			<!-- 2. 表头 (移出 scroll-view，固定不动) -->
			<view class="table-header-wrapper">
				<view class="header-cell col-1">选择</view>
				<view class="header-cell col-2">资产编码</view>
				<view class="header-cell col-3">资产名称</view>
			</view>

			<!-- 3. 表格内容 (scroll-view 只包含数据行，占据剩余空间) -->
			<scroll-view scroll-y class="table-body" @scrolltolower="loadMore">
				<view class="table-row" v-for="(item, index) in filteredAssets" :key="index"
					@click="toggleSelection(item)" :class="{ 'selected': selectedAsset === item.资产编号 }">
					<view class="cell col-1">
						<view
							:class="{ 'radio-checked': selectedAsset === item.资产编号, 'radio-unchecked': selectedAsset !== item.资产编号 }">
						</view>
					</view>
					<view class="cell col-2">{{ item.资产编号 }}</view>
					<view class="cell col-3">{{ item.资产名称 }}</view>
				</view>

				<view v-if="loading" class="loading-indicator">加载中...</view>
				<!-- 垫高底部，防止最后一条数据被阴影遮挡（可选） -->
				<view style="height: 20rpx;"></view>
			</scroll-view>

			<!-- 4. 底部按钮 (固定在最下方) -->
			<view class="footer-section">
				<button class="repair-button" :disabled="!selectedAsset" @click="submitRepair">
					确定报修
				</button>
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
				searchQuery: '',
				assetsList: [],
				selectedAsset: '',
				currentPage: 1,
				pageSize: 10,
				loading: false,
				hasMoreData: true
			};
		},
		// 通过搜索，过滤信息
		computed: {
			filteredAssets() {
				return this.assetsList.filter(asset =>
					(asset.资产编号 && asset.资产编号.includes(this.searchQuery)) ||
					(asset.资产名称 && asset.资产名称.includes(this.searchQuery))
				).slice(0, this.currentPage * this.pageSize);
			},
			selectedAssetInfo() {
				return this.assetsList.find(asset => asset.资产编号 === this.selectedAsset);
			}
		},

		onLoad() {
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
				}

				if (!this.hasMoreData || this.loading) return;

				this.loading = true;
				const userInfo = uni.getStorageSync('userInfo');
				if (!userInfo.token || !userInfo.connid) {
					uni.showToast({
						title: '请先登录',
						icon: 'none'
					});
					this.loading = false;
					return;
				}

				uni.request({
					url: 'http://13.94.38.44:8000/AssetsRepair/GetAllCode',
					method: 'POST',
					header: {
						'content-type': 'application/json'
					},
					data: JSON.stringify({
						connid: userInfo.connid,
						token: userInfo.token,
						search: this.searchQuery,
						isEdit: true,
						page: this.currentPage,
						size: this.pageSize
					}),
					success: (response) => {
						const result = typeof response.data === 'string' ? JSON.parse(response.data) : response
							.data;
						console.log(result);
						if (!result.IsError) {
							if (result.list.length > 0) {
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
						}
					},
					fail: () => {
						uni.showToast({
							title: '请求失败，请检查网络连接',
							icon: 'none'
						});
					},
					complete: () => {
						this.loading = false;
					}
				});
			},
			handleSearchInput() {
				this.fetchAssets(true);
			},
			toggleSelection(item) {
				if (this.selectedAsset === item.资产编号) {
					this.selectedAsset = ''; // 取消选择
				} else {
					this.selectedAsset = item.资产编号; // 选择新项
				}
			},


			submitRepair() {
				if (!this.selectedAsset) {
					uni.showToast({
						title: '请选择一个资产',
						icon: 'none'
					});
					return;
				}

				const {
					资产编号,
					资产名称
				} = this.selectedAssetInfo;

				console.log(this.selectedAssetInfo)
				// 确认报修，拿到资产相应的数据，之后再跳转到repair页面
				uni.request({
					url: 'http://13.94.38.44:8000/AssetsRepair/GetAssetsInfoByCode', // 假设这是提交报修的API地址
					method: 'POST',
					header: {
						'content-type': 'application/json'
					},
					data: JSON.stringify({
						connid: uni.getStorageSync('userInfo').connid,
						AssetsCode: 资产编号,
						token: uni.getStorageSync('userInfo').token,
					}),
					success: (response) => {
						const result = typeof response.data === 'string' ? JSON.parse(response.data) : response
							.data;
						console.log(result)
						if (!result.IsError) {
							// 将资产信息作为参数传递给repair页面
							// 修改 mobileRepair 页面的跳转代码
							uni.navigateTo({
								url: `/pages/repair/repair?data=${encodeURIComponent(JSON.stringify(result))}`
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
			loadMore() {
				this.fetchAssets();
			},
			logout() {
				uni.removeStorageSync('userInfo'); // 修改为移除整个userInfo缓存
				uni.redirectTo({
					url: '/pages/login/login'
				});
			}
		}
	};
</script>

<style scoped>
	/* 定义统一的边框颜色变量（模拟），方便维护 */
	/* 页面主容器 */
	.container {
		display: flex;
		flex-direction: column;
		height: 100vh;
		background-color: #ffffff;
		overflow: hidden;
	}

	/* 内容区域 */
	.content {
		flex: 1;
		display: flex;
		flex-direction: column;
		width: 92%;
		/* 稍微宽一点 */
		max-width: 800rpx;
		margin: 0 auto;
		padding-top: 24rpx;
		overflow: hidden;
	}

	/* --- 1. 搜索框美化 --- */
	.search-input {
		width: 100%;
		height: 80rpx;
		padding: 0 30rpx;
		margin-bottom: 24rpx;
		/* 胶囊形状 */
		background-color: #f7f8fa;
		border: 1px solid transparent; /* 默认无边框 */
		border-radius: 40rpx;
		box-sizing: border-box;
		flex-shrink: 0;
		font-size: 28rpx;
		text-align: center; /* 像 iOS 一样居中 */
		transition: all 0.3s;
		color: #333;
	}
	
	/* 聚焦时显示淡淡的边框 */
	.search-input:focus {
		background-color: #fff;
		border-color: #007aff;
	}

	/* --- 2. 表格容器美化 --- */
	/* 表头容器 */
	.table-header-wrapper {
		display: flex;
		width: 100%;
		background-color: #f5f7fa; /* 极浅的蓝灰色背景 */
		flex-shrink: 0;
		border-top: 1rpx solid #ebeef5; /* 更柔和的线条 */
		border-left: 1rpx solid #ebeef5;
		border-top-left-radius: 12rpx; /* 左上圆角 */
		border-top-right-radius: 12rpx; /* 右上圆角 */
		box-sizing: border-box;
		overflow: hidden; /* 保证圆角不被子元素遮挡 */
	}

	/* 表格滚动区 */
	.table-body {
		flex: 1;
		height: 0;
		width: 100%;
		box-sizing: border-box;
		/* 底部加一点阴影暗示可以滚动 */
		background: #fff;
	}

	/* 表格行 */
	.table-row {
		display: flex;
		width: 100%;
		border-left: 1rpx solid #ebeef5;
		box-sizing: border-box;
		transition: background-color 0.2s;
	}
	
	/* 点击反馈 */
	.table-row:active {
		background-color: #f0f0f0;
	}

	/* --- 3. 单元格美化 --- */
	.header-cell {
		color: #606266; /* 次深灰 */
		font-size: 28rpx;
		font-weight: 600;
	}
	
	.cell {
		color: #303133; /* 深灰 */
		font-size: 26rpx;
	}

	.header-cell,
	.cell {
		text-align: center;
		border-right: 1rpx solid #ebeef5;
		border-bottom: 1rpx solid #ebeef5;
		padding: 24rpx 10rpx; /* 增加一点内边距 */
		display: flex;
		align-items: center;
		justify-content: center;
		word-break: break-all;
		box-sizing: border-box;
	}

	/* 列宽比例 */
	.col-1 { flex: 0.6; } /* 稍微宽一点放单选框 */
	.col-2 { flex: 2.2; }
	.col-3 { flex: 2.2; }

	/* --- 4. 选中状态美化 --- */
	.selected {
		background-color: #ecf5ff !important; /* 非常浅的蓝色，Element UI 风格 */
	}

	/* --- 5. 单选框重绘 (CSS绘制，不再用文字) --- */
	.radio-icon {
		width: 36rpx;
		height: 36rpx;
		border-radius: 50%;
		box-sizing: border-box;
		transition: all 0.2s;
		position: relative;
	}

	/* 未选中：灰色边框，空心 */
	.radio-unchecked {
		border: 2rpx solid #dcdfe6;
		background-color: #fff;
	}

	/* 选中：蓝色背景，蓝色边框，中间有白点 */
	.radio-checked {
		border: 2rpx solid #007aff;
		background-color: #007aff;
	}
	
	/* 选中的小白点 */
	.radio-checked::after {
		content: '';
		position: absolute;
		top: 50%;
		left: 50%;
		transform: translate(-50%, -50%);
		width: 14rpx;
		height: 14rpx;
		background-color: #fff;
		border-radius: 50%;
	}
	
	/* 覆盖掉之前的伪元素样式 */
	.radio-checked::before, .radio-unchecked::before {
		content: none; 
	}

	/* 加载提示 */
	.loading-indicator {
		text-align: center;
		padding: 30rpx;
		color: #909399;
		font-size: 24rpx;
	}

	/* --- 6. 底部按钮美化 --- */
	.footer-section {
		width: 100%;
		flex-shrink: 0;
		background-color: #fff; /* 纯白背景 */
		padding-top: 20rpx;
		padding-bottom: calc(60rpx + constant(safe-area-inset-bottom));
		padding-bottom: calc(60rpx + env(safe-area-inset-bottom));
		
		/* 顶部加一点点阴影，把按钮区和表格区分开 */
		box-shadow: 0 -4rpx 10rpx rgba(0,0,0,0.03); 
		z-index: 10;
	}

	.repair-button {
		width: 100%;
		height: 96rpx; /* 更高的按钮 */
		line-height: 96rpx;
		background: linear-gradient(135deg, #007aff, #0062cc); /* 增加微弱渐变 */
		color: white;
		border: none;
		border-radius: 48rpx; /* 大圆角 */
		font-size: 32rpx;
		font-weight: 600;
		letter-spacing: 2rpx;
		box-shadow: 0 8rpx 20rpx rgba(0, 122, 255, 0.3); /* 漂亮的蓝色投影 */
		transition: transform 0.1s;
	}

	.repair-button:active {
		transform: scale(0.98); /* 点击时的微缩效果 */
	}

	.repair-button:disabled {
		background: #e4e7ed;
		color: #a8abb2;
		box-shadow: none;
	}
</style>