<template>
	<view class="container">
		<CompanySelector :list="companyList" :currentId="currentConnId" @change="onCompanyChange" />
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
				<view class="segment-item" :class="{ active: moduleType === 1 }" @click="switchTab(1)">
					<text class="segment-text">设备报修</text>
				</view>
				<view class="segment-item" :class="{ active: moduleType === 2 }" @click="switchTab(2)">
					<text class="segment-text">设施报修</text>
				</view>
			</view>
		</view>

	</view>
</template>

<script>
	import Navbar from '@/components/Navbar.vue'
	import CompanySelector from '@/components/CompanySelector.vue';
	export default {
		components: {
			Navbar,
			CompanySelector
		},
		data() {
			return {
				username: '',
				moduleType: 1, // 1代表设备报修，2代表设施报修
			}

		},
		onLoad() {
			this.initData();
			// 当页面加载时尝试从本地存储获取用户名
			const userInfo = uni.getStorageSync('userInfo');
			if (userInfo && userInfo.username) {
				this.username = userInfo.username;
			}
		},
		methods: {

			initData() {
				// 1. 获取用户信息
				const userInfo = uni.getStorageSync('userInfo');

				if (userInfo) {
					this.username = userInfo.username;
					this.currentConnId = userInfo.connid; // 获取登录时选的ID

					// 2. 直接从本地缓存读取公司列表
					// (前提是你 Login 页面已经执行了 uni.setStorageSync('company_list_cache', list))
					const cachedList = uni.getStorageSync('company_list_cache');

					if (cachedList && cachedList.length > 0) {
						this.companyList = cachedList;
						console.log('已加载缓存中的公司列表');
					} else {
						console.log('警告：本地缓存中没有公司列表数据');
						// 如果缓存真的空了，为了防止报错，你可能还是得考虑要不要踢回登录页
					}
				} else {
					uni.redirectTo({
						url: '/pages/login/login'
					});
				}
			},

			onCompanyChange(item) {
			  console.log('准备切换到公司:', item.Name, 'ID:', item.ID);
			
			  // 1. 获取登录页面存下的账号密码
			  const credentials = uni.getStorageSync('login_credentials');
			  
			  // 如果没有密码缓存（比如用户清了缓存），就踢回登录页
			  if (!credentials || !credentials.password) {
			    uni.showToast({ title: '登录信息已过期，请重新登录', icon: 'none' });
			    setTimeout(() => {
			      uni.reLaunch({ url: '/pages/login/login' });
			    }, 1500);
			    return;
			  }
			
			  uni.showLoading({ title: '正在切换...' });
			
			  // 2. 调用登录接口 (使用新公司的 ID)
			  uni.request({
			    url: 'http://13.94.38.44:8000/AssetsRepair/SignIn',
			    method: 'POST',
			    header: { 'content-type': 'application/json' },
			    data: {
			      connid: item.ID,              // 【关键】使用新选择的公司 ID
			      strAccount: credentials.account, // 使用缓存的账号
			      strPswd: credentials.password    // 使用缓存的密码
			    },
			    success: (res) => {
			      uni.hideLoading();
			      let result;
			      try {
			        result = typeof res.data === 'string' ? JSON.parse(res.data) : res.data;
			      } catch (e) {
			        uni.showToast({ title: '响应解析失败', icon: 'none' });
			        return;
			      }
			
			      // 3. 判断切换是否成功
			      if (result.IsError === false) {
			        
			        // A. 更新页面状态
			        this.currentConnId = item.ID;
			        this.username = result.EMP_NAME; // 更新显示的用户名(如果不同公司名字不一样的话)
			
			        // B. 【核心】更新本地存储的 userInfo
			        // 这样后续的扫码、报修请求都会带上这个新的 token 和 connid
			        const newUserInfo = {
			          token: result.token,      // 新的 Token
			          username: result.EMP_NAME,
			          connid: item.ID           // 确保这里存的是新公司的 ID
			        };
			        uni.setStorageSync('userInfo', newUserInfo);
			
			        uni.showToast({ title: '切换成功', icon: 'success' });
			
			        // C. (可选) 如果你的首页有数据列表，建议在这里调用刷新方法
			        // this.loadDashboardData(); 
			
			      } else {
			        // 登录失败（可能是该账号在那个公司没有权限）
			        uni.showToast({ 
			          title: result.message || '切换失败，该账号无法登录此公司', 
			          icon: 'none',
			          duration: 3000
			        });
			        
			        // 失败后，你可以选择把下拉框的值变回原来的，或者保持原样
			      }
			    },
			    fail: () => {
			      uni.hideLoading();
			      uni.showToast({ title: '网络请求失败', icon: 'none' });
			    }
			  });
			},
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
						if (!result.isError && !result.IsError) {
							// 将资产信息作为参数传递给repair页面
							uni.navigateTo({
								url: `/pages/repair/repair?data=${JSON.stringify(result)}`
							});
						} else {
							uni.showToast({
								title: result.msg || '获取资产信息失败', // 优先显示后端返回的具体错误原因
								icon: 'none',
								duration: 3000 // 错误提示稍微长一点，方便用户看清
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
		background-color: #f7f8fa;
		/* 浅灰底色 */
		overflow: hidden;
	}

	/* --- 1. 中间九宫格区域 --- */
	.content-area {
		flex: 1;
		/* 占据剩余空间 */
		width: 100%;
		box-sizing: border-box;
	}

	.grid-container {
		padding: 40rpx 30rpx;
		display: flex;
		flex-direction: column;
		gap: 30rpx;
		/* 行间距 */
	}

	.grid-row {
		display: flex;
		justify-content: space-between;
		gap: 30rpx;
		/* 列间距 */
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
		box-shadow: 0 8rpx 24rpx rgba(0, 0, 0, 0.03);
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
		font-size: 32rpx;
		/* 字体稍微大一点，因为没有描述了 */
		font-weight: 600;
		color: #333;
		margin-bottom: 0;
		/* 不需要底边距了 */
	}

	/* --- 2. 底部 Tab 切换区 (固定) --- */
	.bottom-tab-container {
		flex-shrink: 0;
		/* 禁止被压缩 */
		background-color: #fff;
		padding: 20rpx 40rpx;
		/* 适配全面屏底部安全区 */
		padding-bottom: calc(30rpx + constant(safe-area-inset-bottom));
		padding-bottom: calc(30rpx + env(safe-area-inset-bottom));

		border-top-left-radius: 30rpx;
		border-top-right-radius: 30rpx;
		box-shadow: 0 -4rpx 20rpx rgba(0, 0, 0, 0.05);
		/* 向上投影 */
		z-index: 10;
	}

	/* 分段控制器容器 */
	.segment-control {
		display: flex;
		background-color: #f0f2f5;
		/* 灰色背景条 */
		border-radius: 50rpx;
		/* 全胶囊圆角 */
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
		border-radius: 40rpx;
		/* 内层圆角 */
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
		box-shadow: 0 4rpx 10rpx rgba(0, 0, 0, 0.1);
	}

	.segment-item.active .segment-text {
		color: #007aff;
		/* 蓝色高亮 */
		font-weight: bold;
		font-size: 30rpx;
	}
</style>