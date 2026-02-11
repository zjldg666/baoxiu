<template>
	<view class="container">
		<!-- 1. 顶部导航 -->
		<navbar :username="username" @logout="logout" />

		<!-- 2. 中间滚动区域 (flex: 1 自动占据剩余空间) -->
		<scroll-view scroll-y class="main-content">
			<form @submit="submitRepair">

				<!-- A. 基础信息卡片 (紧凑型列表) -->
				<view class="card compact-card">
					<view class="card-title">基础信息</view>

					<!-- 单行展示：标签左对齐，内容右对齐 -->
					<view class="info-row">
						<text class="label">报修单号</text>
						<text class="value">{{ docnum }}</text>
					</view>
					<view class="info-row">
						<text class="label">资产编码</text>
						<text class="value">{{ AssetsCode }}</text>
					</view>
					<view class="info-row">
						<text class="label">资产名称</text>
						<text class="value highlight">{{ AssetsName }}</text>
					</view>
					<view class="info-row">
						<text class="label">使用部门</text>
						<text class="value">{{ UserDept }}</text>
					</view>
					<view class="info-row no-border">
						<text class="label">资产类型</text>
						<text class="value">{{ AssetsType }}</text>
					</view>
				</view>

				<!-- B. 填写区域卡片 -->
				<view class="card">
					<view class="card-title">报修信息</view>

					<!-- 日期选择 -->
					<view class="form-row">
						<text class="form-label required">希望完工日期</text>
						<picker mode="date" @change="onDateChange" class="picker-flex">
							<view :class="FinishDate ? 'picker-text' : 'picker-placeholder'">
								{{ FinishDate || '请选择日期 >' }}
							</view>
						</picker>
					</view>
					<view class="form-row-label">
						<text class="form-label required">故障描述：</text>
					</view>
					<!-- 故障描述 (限制高度) -->
					<view class="form-col">
						<textarea v-model="Discription" placeholder="请简要描述故障情况..." class="mini-textarea"
							placeholder-class="ph-style"></textarea>
					</view>
				</view>

				<!-- C. 图片区域 (横向排列) -->
				<view class="card image-card">
					<view class="card-title-row">
						<text class="card-title">相关图片</text>
						<text class="img-count">{{ images.length }}</text>
					</view>

					<scroll-view scroll-x class="image-scroll">
						<view class="image-flex-container">
							<!-- 已上传图片 -->
							<block v-for="(src, index) in images" :key="index">
								<view class="image-wrapper">
									<image :src="src" mode="aspectFill" class="thumb-img" @click="previewImage(index)">
									</image>
									<view class="close-badge" @click.stop.prevent="removeImage(index)">×</view>
								</view>
							</block>
							<!-- 添加按钮 -->
							<view class="add-btn" @click="chooseImage" v-if="images.length < 9">
								<text class="plus-sign">+</text>
							</view>
						</view>
					</scroll-view>
				</view>

				<!-- 底部垫高 -->
				<view style="height: 20rpx;"></view>
			</form>
		</scroll-view>

		<!-- 3. 底部按钮区 (固定) -->
		<view class="footer-fixed">
			<button class="action-btn secondary" @click="showConfirmDialog('稍后')">稍后报修</button>
			<button class="action-btn primary" @click="showConfirmDialog('立即')">立即报修</button>
		</view>
		<ConfirmPopup v-model:visible="showPopup" title="操作提示" @confirm="handlePopupConfirm">
			<view class="popup-content">
				确定要<text class="highlight-text">{{ currentAction }}</text>报修吗？
			</view>
		</ConfirmPopup>
	</view>
</template>

<script>
	import Navbar from '@/components/Navbar.vue';
	import ConfirmPopup from '@/components/ConfirmPopup.vue';
	export default {
		components: {
			Navbar,
			ConfirmPopup
		},
		data() {
			return {
				username: '',
				connid: '',
				token: '',
				docnum: '',
				AssetsCode: '',
				AssetsName: '',
				UserDept: '',
				AssetsType: '',
				FinishDate: '',
				Discription: '',
				images: [],
				showPopup: false, // 控制弹窗显示
				currentAction: '' // 记录当前操作：'立即' 或 '稍后'
			};
		},
		//页面刷新时，将数据赋值，以便此页面使用
		onLoad(options) {
			if (options.data) {
				const assetInfo = JSON.parse(decodeURIComponent(options.data));
				console.log(assetInfo);
				this.connid = assetInfo.connid;
				this.token = assetInfo.token;
				this.docnum = assetInfo.docnum;
				this.AssetsCode = assetInfo.AssetsCode;
				this.AssetsName = assetInfo.AssetsName;
				this.UserDept = assetInfo.UserDept || '';
				this.AssetsType = assetInfo.AssetsType;

				// 检查是否有草稿数据
				const draftData = uni.getStorageSync('repairDraft_' + this.AssetsCode);
				console.log(draftData);
				if (draftData) {
					this.FinishDate = draftData.FinishDate;
					this.Discription = draftData.Discription;
					this.images = draftData.images;
				}
				// 当页面加载时尝试从本地存储获取用户名
				const userInfo = uni.getStorageSync('userInfo');
				if (userInfo && userInfo.username) {
					this.username = userInfo.username;
				}
			}
		},
		methods: {
			logout() {
				uni.removeStorageSync('userInfo'); // 修改为移除整个userInfo缓存
				uni.redirectTo({
					url: '/pages/login/login'
				});
			},
			//选择完工日期
			onDateChange(e) {
				this.FinishDate = e.detail.value;
			},
			chooseImage() {
				uni.chooseImage({
					count: 1,
					success: (res) => {
						this.images.push(...res.tempFilePaths);
					}
				});
			},
			// 新增图片预览方法
			previewImage(index) {
				uni.previewImage({
					current: index, // 当前点击的图片索引
					urls: this.images // 所有图片的 URL 数组
				});
			},
			//删除照片
			removeImage(index) {
				this.images.splice(index, 1);
			},


			showConfirmDialog(action) {
				// 1. 验证必填项
				if (!this.FinishDate || !this.Discription) {
					uni.showToast({
						title: '请确保所有必填项已填写',
						icon: 'none'
					});
					return;
				}

				// 2. 不再直接调用 uni.showModal，而是记录操作并打开弹窗
				this.currentAction = action;
				this.showPopup = true;
			},
			async handlePopupConfirm() {
				// 关闭弹窗
				this.showPopup = false;

				// 根据之前的选择执行逻辑
				if (this.currentAction === '立即') {
					await this.submitRepair(true);
				} else {
					await this.saveAsDraft();
					uni.redirectTo({
						url: '/pages/home/home'
					});
				}
			},

			showModalAsync(content) {
				return new Promise((resolve) => {
					uni.showModal({
						title: '提示',
						content: content,
						success: (res) => resolve(res.confirm),
						fail: () => resolve(false)
					});
				});
			},

			async submitRepair(isImmediate) {
				try {
					const type = isImmediate ? "1" : "0";
					const base64Images = await Promise.all(this.images.map(src => this.urlToBase64(src)));
					console.log('Base64 Images:', base64Images); // 确认所有图片都被正确转换

					const requestData = {
						connid: this.connid,
						AssetsCode: this.AssetsCode,
						token: this.token,
						docnum: this.docnum,
						Discription: this.Discription,
						FinishDate: this.FinishDate,
						type: type,
						FileList: base64Images
					};

					console.log('请求数据:', requestData);

					const res = await new Promise((resolve, reject) => {
						uni.request({
							url: 'http://13.94.38.44:8000/AssetsRepair/SaveNewInfo', // 提交检修的接口
							method: 'POST',
							data: requestData,
							success: resolve,
							fail: reject
						});
					});

					console.log('提交成功:', res.data);

					// 成功提交后删除草稿数据
					uni.removeStorageSync('repairDraft_' + this.AssetsCode);

					uni.showToast({
						title: '提交成功',
						icon: 'success',
						duration: 2000, // 显示时长为2秒
						complete: () => {
							uni.redirectTo({
								url: '/pages/home/home'
							}); // 在提示信息显示完后返回首页
						}
					});
				} catch (err) {
					console.error('提交失败:', err);
					uni.showToast({
						title: '提交失败',
						icon: 'none'
					});
				}
			},

			async saveAsDraft() {
				try {
					const draftData = {
						connid: this.connid,
						token: this.token,
						docnum: this.docnum,
						AssetsCode: this.AssetsCode,
						AssetsName: this.AssetsName,
						UserDept: this.UserDept,
						AssetsType: this.AssetsType,
						FinishDate: this.FinishDate,
						Discription: this.Discription,
						images: this.images
					};

					console.log('准备保存草稿:', draftData); // 添加日志输出

					uni.setStorageSync('repairDraft_' + this.AssetsCode, draftData); // 使用docnum作为键值，以便于查找

					console.log('草稿保存成功'); // 添加日志输出

					uni.showToast({
						title: '已保存为草稿',
						icon: 'success',
						duration: 2000, // 显示时长为2秒
						complete: () => {
							console.log('提示框显示完毕，即将返回首页'); // 添加日志输出
							uni.redirectTo({
								url: '/pages/home/home'
							});
						}
					});
				} catch (err) {
					console.error('保存草稿失败:', err);
					uni.showToast({
						title: '保存草稿失败',
						icon: 'none'
					});
				}
			},

			urlToBase64(url, type = 'png') {
				if (typeof url !== 'string' || url.trim() === '') {
					return Promise.reject(new Error('Invalid URL'));
				}

				return new Promise((resolve, reject) => {
					if (url.startsWith('http')) {
						// 网络地址 或者H5端本地相对路径 可使用request方式
						uni.request({
							url: url,
							method: 'GET',
							responseType: 'arraybuffer',
							success: (res) => {
								const base64 =
									`data:image/${type};base64,${uni.arrayBufferToBase64(res.data)}`;
								resolve(base64);
							},
							fail: (err) => {
								reject(new Error(`Request failed: ${err}`));
							}
						});
					} else {
						// #ifdef APP-PLUS
						uni.compressImage({
							src: url,
							quality: 100, // 图片质量压缩0~100，100表示图片质量保持原样
							success: (res) => {
								const tempUrl = res
									.tempFilePath; // 使用compressImage获取到安卓本地路径file:///...
								plus.io.resolveLocalFileSystemURL(tempUrl, (entry) => {
									entry.file((e) => {
										let fileReader = new plus.io.FileReader();
										fileReader.onload = (r) => {
											resolve(r.target.result);
										};
										fileReader.readAsDataURL(e);
									}, (error) => {
										reject(new Error(
											`File resolution failed: ${error}`
										));
									});
								}, (error) => {
									reject(new Error(`File resolution failed: ${error}`));
								});
							},
							fail: (err) => {
								reject(new Error(`Compress image failed: ${err}`));
							}
						});
						// #endif

						// 如果是在非APP环境下（例如H5），可以直接使用Canvas进行转换
						// #ifndef APP-PLUS
						const img = new Image();
						img.src = url;
						img.onload = () => {
							const canvas = document.createElement('canvas');
							canvas.width = img.width;
							canvas.height = img.height;
							const ctx = canvas.getContext('2d');
							ctx.drawImage(img, 0, 0);
							const dataURL = canvas.toDataURL(`image/${type}`);
							resolve(dataURL);
						};
						img.onerror = reject;
						// #endif
					}
				});
			},
			// handleInvalidImagePath(path) {
			//   console.warn(`Invalid image path found: ${path}, removing it from images list.`);
			//   // 清除无效的图片路径
			//   this.images = this.images.filter(src => src !== path);
			//   uni.showToast({
			//     title: '发现无效图片路径，请重新上传相关图片。',
			//     icon: 'none',
			//   duration: 2000, // 显示时长为2秒
			//   });
			// },

		}
	};
</script>

<style scoped>
	/* --- 全局布局 --- */
	.container {
		display: flex;
		flex-direction: column;
		height: 100vh;
		/* 强制占满全屏 */
		background-color: #f5f7fa;
		overflow: hidden;
	}

	.main-content {
		flex: 1;
		/* 占据中间所有空间 */
		height: 0;
		/* 启用内部滚动 */
		width: 100%;
		padding: 20rpx 24rpx;
		box-sizing: border-box;
	}

	/* --- 通用卡片样式 --- */
	.card {
		background-color: #fff;
		border-radius: 16rpx;
		padding: 20rpx 24rpx;
		margin-bottom: 20rpx;
		box-shadow: 0 2rpx 6rpx rgba(0, 0, 0, 0.02);
	}

	.card-title {
		font-size: 28rpx;
		font-weight: bold;
		color: #333;
		margin-bottom: 16rpx;
		padding-left: 10rpx;
		border-left: 6rpx solid #007aff;
		/* 蓝色竖条装饰 */
		line-height: 1;
	}

	/* --- A. 紧凑型信息列表 --- */
	.compact-card {
		padding-bottom: 10rpx;
	}

	.info-row {
		display: flex;
		justify-content: space-between;
		align-items: center;
		padding: 26rpx 0;
		/* 减小行间距 */
		border-bottom: 1rpx solid #f5f5f5;
	}

	.form-row-label {
		margin-top: 20rpx;
		margin-bottom: 10rpx;
	}

	.no-border {
		border-bottom: none;
	}

	.label {
		font-size: 26rpx;
		color: #909399;
		/* 浅灰色标签 */
	}

	.value {
		font-size: 26rpx;
		color: #333;
		/* 深色内容 */
		font-weight: 500;
		text-align: right;
		flex: 1;
		margin-left: 20rpx;
		overflow: hidden;
		text-overflow: ellipsis;
		white-space: nowrap;
	}

	.highlight {
		color: #007aff;
		font-weight: bold;
	}

	/* --- B. 表单样式 --- */
	.form-row {
		display: flex;
		align-items: center;
		justify-content: space-between;
		height: 80rpx;
		border-bottom: 1rpx solid #f5f5f5;
		margin-bottom: 10rpx;
	}

	.form-label {
		font-size: 28rpx;
		color: #333;
	}

	.required::before {
		content: '*';
		color: #ff4d4f;
		margin-right: 4rpx;
	}

	.picker-flex {
		flex: 1;
		text-align: right;
	}

	.picker-text {
		font-size: 28rpx;
		color: #333;
	}

	.picker-placeholder {
		font-size: 28rpx;
		color: #c0c4cc;
	}

	/* 文本域 */
	.mini-textarea {
		width: 100%;
		height: 140rpx;
		/* 固定高度，防止太高 */
		background-color: #f9f9f9;
		border-radius: 8rpx;
		padding: 16rpx;
		box-sizing: border-box;
		font-size: 28rpx;
		line-height: 1.4;
		color: #333;
	}

	/* --- C. 图片区域 (横向滚动) --- */
	.card-title-row {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 16rpx;
	}

	.img-count {
		font-size: 24rpx;
		color: #999;
	}

	.image-scroll {
		width: 100%;
		white-space: nowrap;
		/* 关键：不换行 */
	}

	.image-flex-container {
		display: flex;
		/* 让图片横向排列 */
		align-items: center;
		padding-bottom: 10rpx;
		/* 预留一点滚动条空间 */
	}

	.image-wrapper {
		position: relative;
		display: inline-block;
		width: 130rpx;
		height: 130rpx;
		margin-right: 20rpx;
		flex-shrink: 0;
	}

	.thumb-img {
		width: 100%;
		height: 100%;
		border-radius: 8rpx;
	}

	.close-badge {
		position: absolute;
		top: -10rpx;
		right: -10rpx;
		width: 36rpx;
		height: 36rpx;
		background-color: rgba(0, 0, 0, 0.6);
		color: white;
		border-radius: 50%;
		text-align: center;
		line-height: 34rpx;
		font-size: 24rpx;
		z-index: 10;
	}

	.add-btn {
		display: inline-flex;
		align-items: center;
		justify-content: center;
		width: 130rpx;
		height: 130rpx;
		background-color: #f5f7fa;
		border: 2rpx dashed #dcdfe6;
		border-radius: 8rpx;
		flex-shrink: 0;
	}

	.plus-sign {
		font-size: 60rpx;
		color: #c0c4cc;
		margin-top: -6rpx;
	}

	/* --- 3. 底部按钮固定区 --- */
	.footer-fixed {
		flex-shrink: 0;
		background-color: #fff;
		padding: 20rpx 30rpx;
		/* 适配安全区域 */
		padding-bottom: calc(20rpx + constant(safe-area-inset-bottom));
		padding-bottom: calc(20rpx + env(safe-area-inset-bottom));
		box-shadow: 0 -2rpx 10rpx rgba(0, 0, 0, 0.05);

		display: flex;
		justify-content: space-between;
		gap: 30rpx;
		z-index: 99;
	}

	.action-btn {
		flex: 1;
		height: 80rpx;
		line-height: 80rpx;
		border-radius: 40rpx;
		font-size: 30rpx;
		font-weight: 600;
		border: none;
	}

	.action-btn::after {
		border: none;
	}

	.secondary {
		background-color: #f0f2f5;
		color: #606266;
	}

	.primary {
		background: linear-gradient(135deg, #007aff, #0062cc);
		color: white;
		box-shadow: 0 4rpx 12rpx rgba(0, 122, 255, 0.3);
	}

	.primary:active {
		opacity: 0.9;
	}

	/* 弹窗内容样式 */
	.popup-content {
		font-size: 30rpx;
		color: #333;
		text-align: center;
		padding: 20rpx 0;
	}

	.highlight-text {
		color: #007aff;
		/* 蓝色高亮 */
		font-weight: bold;
		margin: 0 8rpx;
	}
</style>