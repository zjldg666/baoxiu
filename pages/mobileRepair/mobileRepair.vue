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
				<view class="table-row" v-for="(item, index) in assetsList" :key="index"
					@click="toggleSelection(item)" :class="{ 'selected': selectedAsset === item.资产编号 }">
					<view class="cell col-1">
						<!-- 将 class 改为 checkbox-icon 风格 -->
						<view class="checkbox-icon"
							:class="{ 'checkbox-checked': selectedAsset === item.资产编号, 'checkbox-unchecked': selectedAsset !== item.资产编号 }">
							<!-- 选中时显示的对勾图标（用CSS画或者字体图标） -->
							<text v-if="selectedAsset === item.资产编号" class="check-mark">✓</text>
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
				<button class="repair-button" :class="{ 'btn-disabled': !selectedAsset }" :disabled="!selectedAsset"
					@click="submitRepair">
					{{ !selectedAsset ? '请选择报修资产' : '确定报修' }}
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
				allAssets: [], // 存放接口返回的【全部】数据
				assetsList: [], // 存放【当前显示】的数据

				username: '',
				searchQuery: '',
				selectedAsset: '',
				currentPage: 1,
				pageSize: 20,
				loading: false,
				hasMoreData: true,
				moduleType: ''
			};
		},
		// 通过搜索，过滤信息
		computed: {
			selectedAssetInfo() {
				// 注意：这里要在 allAssets 里找，防止搜不到未渲染的数据
				return this.allAssets.find(asset => asset.资产编号 === this.selectedAsset);
			}
		},

		onLoad(options) {
			if (options.moduleType) {
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

				const requestData = {
					connid: userInfo.connid,
					token: userInfo.token,
					search: this.searchQuery,
					isEdit: true,
					moduleType: this.moduleType,

				};

				// 2. 添加日志打印
				console.log('GetAllCode 接口请求参数:', requestData);
				uni.request({
					url: 'http://13.94.38.44:8000/AssetsRepair/GetAllCode',
					method: 'POST',
					header: {
						'content-type': 'application/json'
					},
					data: JSON.stringify(requestData),
					success: (response) => {
						const result = typeof response.data === 'string' ? JSON.parse(response.data) : response
							.data;
						if (!result.IsError) {
							// 1. 把所有数据存入 allAssets
							this.allAssets = result.list || [];

							// 2. 初始化分页状态
							this.currentPage = 1;
							this.hasMoreData = true;
							this.assetsList = [];

							// 3. 执行第一次本地切片渲染
							this.renderLocalData();

						} else {
							uni.showToast({
								title: '获取数据失败',
								icon: 'none'
							});
						}
					},
					fail: () => {
						uni.showToast({
							title: '网络失败',
							icon: 'none'
						});
					},
					complete: () => {
						this.loading = false;
					}
				});
			},
handleSearchInput() {
				// 搜索时，重置页码，重新计算显示的列表
				this.currentPage = 1;
				this.hasMoreData = true;
				this.renderLocalData();
			},
			loadMore() {
							if (!this.hasMoreData) return;
							
							// 只是增加页码，然后重新切片显示
							this.currentPage++;
							this.renderLocalData();
						},
			toggleSelection(item) {
				if (this.selectedAsset === item.资产编号) {
					this.selectedAsset = ''; // 取消选择
				} else {
					this.selectedAsset = item.资产编号; // 选择新项
				}
			},renderLocalData() {
				// 1. 这一步做搜索过滤 (如果需要前端搜索)
				let filtered = this.allAssets;
				if (this.searchQuery) {
					filtered = this.allAssets.filter(asset => 
						(asset.资产编号 && asset.资产编号.includes(this.searchQuery)) ||
						(asset.资产名称 && asset.资产名称.includes(this.searchQuery))
					);
				}

				// 2. 计算当前应该显示多少条
				const total = filtered.length;
				const endIndex = this.currentPage * this.pageSize;
				
				// 3. 截取数据赋值给 assetsList
				this.assetsList = filtered.slice(0, endIndex);
				
				// 4. 判断是不是到底了
				if (this.assetsList.length >= total) {
					this.hasMoreData = false;
				} else {
					this.hasMoreData = true;
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
							result.moduleType = this.moduleType;
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
		background-color: #f7f8fa;
		border: 1px solid transparent;
		border-radius: 40rpx;
		box-sizing: border-box;
		flex-shrink: 0;
		font-size: 28rpx;
		text-align: center;
		transition: all 0.3s;
		color: #333;
	}

	.search-input:focus {
		background-color: #fff;
		border-color: #007aff;
	}

	/* --- 2. 表格容器美化 --- */
	.table-header-wrapper {
		display: flex;
		width: 100%;
		background-color: #f5f7fa;
		flex-shrink: 0;
		border-top: 1rpx solid #ebeef5;
		border-left: 1rpx solid #ebeef5;
		border-top-left-radius: 12rpx;
		border-top-right-radius: 12rpx;
		box-sizing: border-box;
		overflow: hidden;
	}

	.table-body {
		flex: 1;
		height: 0;
		width: 100%;
		box-sizing: border-box;
		background: #fff;
	}

	.table-row {
		display: flex;
		width: 100%;
		border-left: 1rpx solid #ebeef5;
		box-sizing: border-box;
		transition: background-color 0.2s;
	}

	.table-row:active {
		background-color: #f0f0f0;
	}

	/* 选中整行的背景色 */
	.selected {
		background-color: #ecf5ff !important;
	}

	/* --- 3. 单元格美化 --- */
	.header-cell {
		color: #606266;
		font-size: 28rpx;
		font-weight: 600;
	}

	.cell {
		color: #303133;
		font-size: 26rpx;
	}

	.header-cell,
	.cell {
		text-align: center;
		border-right: 1rpx solid #ebeef5;
		border-bottom: 1rpx solid #ebeef5;
		padding: 24rpx 10rpx;
		display: flex;
		align-items: center;
		justify-content: center;
		word-break: break-all;
		box-sizing: border-box;
	}

	.col-1 {
		flex: 0.6;
	}

	.col-2 {
		flex: 2.2;
	}

	.col-3 {
		flex: 2.2;
	}

	/* --- 4. 选择框 (Checkbox 风格) --- */
	.checkbox-icon {
		width: 44rpx;
		height: 44rpx;
		border-radius: 8rpx;
		box-sizing: border-box;
		transition: all 0.2s;
		display: flex;
		align-items: center;
		justify-content: center;
	}

	/* 未选中 */
	.checkbox-unchecked {
		border: 3rpx solid #dcdfe6;
		background-color: #fff;
	}

	/* 选中 */
	.checkbox-checked {
		border: 3rpx solid #007aff;
		background-color: #007aff;
	}

	/* 对勾符号 */
	.check-mark {
		color: white;
		font-size: 32rpx;
		font-weight: bold;
		line-height: 1;
	}

	/* --- 5. 加载提示 --- */
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
		background-color: #fff;
		padding-top: 20rpx;
		padding-bottom: calc(60rpx + constant(safe-area-inset-bottom));
		padding-bottom: calc(60rpx + env(safe-area-inset-bottom));
		box-shadow: 0 -4rpx 10rpx rgba(0, 0, 0, 0.03);
		z-index: 10;
	}

	.repair-button {
		width: 100%;
		height: 96rpx;
		line-height: 96rpx;
		background: linear-gradient(135deg, #007aff, #0062cc);
		color: white;
		border: none;
		border-radius: 48rpx;
		font-size: 32rpx;
		font-weight: 600;
		letter-spacing: 2rpx;
		box-shadow: 0 8rpx 20rpx rgba(0, 122, 255, 0.3);
		transition: all 0.2s;
	}

	.repair-button:active {
		transform: scale(0.98);
	}

	/* 禁用状态 */
	.repair-button.btn-disabled {
		background: #e4e7ed;
		color: #909399;
		box-shadow: none;
		pointer-events: none;
	}

	.repair-button:disabled {
		background: #e4e7ed;
		color: #909399;
		box-shadow: none;
	}
</style>