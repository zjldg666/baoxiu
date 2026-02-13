<template>
	<view class="company-bar">
		<picker mode="selector" :range="companyNames" :value="pickerIndex" @change="onChange">
			<view class="picker-wrap">
				<text class="picker-text">{{ currentCompanyName }}</text>
				<text class="picker-arrow">▼</text>
			</view>
		</picker>
	</view>
</template>

<script>
	export default {
		props: {
			list: {
				type: Array,
				default: () => []
			},
			currentId: {
				type: [String, Number],
				default: ''
			}
		},
		computed: {
			companyNames() {
				return this.list.map(item => item.Name);
			},
			pickerIndex() {
				if (!this.currentId || !this.list.length) return 0;
				const idx = this.list.findIndex(item => item.ID == this.currentId);
				return idx >= 0 ? idx : 0;
			},
			currentCompanyName() {
				const idx = this.pickerIndex;
				return this.list[idx] ? this.list[idx].Name : '请选择公司';
			}
		},
		methods: {
			onChange(e) {
				this.$emit('change', this.list[e.detail.value]);
			}
		}
	}
</script>

<style scoped>
	.company-bar {
		width: 100%;
		background-color: #f5f7fa;
		/* 浅灰色背景区别于Navbar */
		padding-top: var(--status-bar-height);
		/* 关键：适配刘海屏 */
		height: 88rpx;
		display: flex;
		align-items: center;
		justify-content: center;
		z-index: 999;
	}

	.picker-wrap {
		display: flex;
		align-items: center;
		padding: 20rpx;
	}

	.picker-text {
		font-size: 32rpx;
		font-weight: bold;
		max-width: 500rpx;
		overflow: hidden;
		white-space: nowrap;
		text-overflow: ellipsis;
	}

	.picker-arrow {
		margin-left: 10rpx;
		font-size: 24rpx;
		color: #666;
	}
</style>