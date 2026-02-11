<template>
  <view class="container">
	<Navbar></Navbar>
    <view class="login-box">
      <picker mode="selector" :range="companyNames" @change="onCompanyChange">
        <view class="picker">
          {{ selectedCompanyName || '请选择公司' }}
        </view>
      </picker>
      <!-- 用户名输入框 -->
      <input type="text" placeholder="请输入用户名" v-model="username" class="input"/>
      <!-- 密码输入框 -->	
      <input type="password" placeholder="请输入密码" v-model="password" class="input"/>
      
      <button type="primary" @click="login" class="button">登录</button>
    </view>
  </view>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import Navbar from '@/components/Navbar.vue';
import checkUpdate from '@/uni_modules/uni-upgrade-center-app/utils/check-update';

export default {
	components: { Navbar },
  setup() {
    // 响应式数据
    const companyList = ref([])
    const selectedCompanyId = ref('')
    const selectedCompanyName = ref('')
    const username = ref('')
    const password = ref('')

    // 计算属性：公司名称列表
    const companyNames = computed(() => {
      return companyList.value.map(company => company.Name)
    })

    // 获取公司列表（核心修复：改回POST+添加空请求体）
    const fetchCompanyList = () => {
      uni.request({
        url: 'http://13.94.38.44:8000/AssetsRepair/Index',
        method: 'POST', 
        header: {
          'Content-Type': 'application/json' 
        },
        data: {}, // 关键：添加空请求体（让框架生成Content-Length: 2，避免411错误）
        success: (res) => {
          console.log('获取公司列表响应:', res)
          
          // 第一步：先判断HTTP状态码是否成功（200-299）
          if (res.statusCode < 200 || res.statusCode >= 300) {
            console.error('服务器返回错误状态码:', res.statusCode)
            uni.showToast({ title: `服务器错误(${res.statusCode})`, icon: 'none' })
            return
          }

          // 第二步：安全解析JSON数据
          let data;
          try {
            data = typeof res.data === 'string' ? JSON.parse(res.data) : res.data
          } catch (e) {
            console.error('JSON解析失败:', e, '原始数据:', res.data)
            uni.showToast({ title: '数据格式错误', icon: 'none' })
            return
          }

          // 第三步：检查是否有预期的list字段
          if (!data || !data.list) {
            console.error('数据缺少list字段:', data)
            uni.showToast({ title: '公司列表为空', icon: 'none' })
            return
          }

          // 成功：赋值公司列表
          companyList.value = data.list
          console.log('公司列表获取成功:', companyList.value)
        },
        fail: (err) => {
          console.error('网络请求失败:', err)
          uni.showToast({ title: '网络连接异常', icon: 'none' })
        }
      })
    }

    // 公司选择事件
    const onCompanyChange = (event) => {
      const index = event.detail.value
      selectedCompanyId.value = companyList.value[index].ID
      selectedCompanyName.value = companyList.value[index].Name
    }

    // 登录方法（保留原有逻辑，确保数据格式正确）
    const login = () => {
      if (!selectedCompanyId.value || !username.value || !password.value) {
        uni.showToast({ title: '请填写完整信息', icon: 'none' })
        return
      }

      uni.request({
        url: 'http://13.94.38.44:8000/AssetsRepair/SignIn',
        method: 'POST',
        header: { 'content-type': 'application/json' },
        data: { // 直接传对象（框架自动序列化+加Content-Length）
          connid: selectedCompanyId.value,
          strAccount: username.value,
          strPswd: password.value
        },
        success: (res) => {
          let result;
          try {
            result = typeof res.data === 'string' ? JSON.parse(res.data) : res.data
          } catch (e) {
            console.error('登录数据解析失败:', e)
            uni.showToast({ title: '登录响应异常', icon: 'none' })
            return
          }

          if (result.IsError === false) {
            uni.setStorageSync('userInfo', {
              token: result.token,
              username: result.EMP_NAME,
              connid: result.connid
            })
            uni.redirectTo({ url: '/pages/home/home' })
          } else {
            password.value = ''
            const msg = result.message || '登录失败，请重试'
            uni.showToast({ title: msg.includes('账号或密码错误') ? '账号或密码错误' : msg, icon: 'none' })
          }
        },
        fail: () => {
          password.value = ''
          uni.showToast({ title: '请求失败，请检查网络', icon: 'none' })
        }
      })
    }

    // 生命周期：挂载时获取公司列表+检查更新
    onMounted(() => {
      fetchCompanyList()
      checkUpdate()
    })

    // 暴露给模板的数据和方法
    return {
      companyList,
      selectedCompanyId,
      selectedCompanyName,
      username,
      password,
      companyNames,
      onCompanyChange,
      login
    }
  }
}
</script>

<style>
/* 样式保持不变，无需修改 */
.container {
 display: flex;
 flex-direction: column;
 align-items: center;
 min-height: 100vh;
 background-color: #f5f5f5;
 padding-top: 0;
}
.login-box {
  width: 600rpx;
  padding: 40rpx;
  background-color: #fff;
  box-shadow: 0 0 20rpx rgba(0,0,0,0.1);
  border-radius: 20rpx;
  margin-top: 250rpx;
}
.picker {
  width: 95%;
  padding: 20rpx;
  margin-bottom: 30rpx;
  border: 2rpx solid #ccc;
  border-radius: 10rpx;
  text-align: center;
}
.input {
  width: 95%;
  padding: 20rpx;
  margin-bottom: 30rpx;
  border: 2rpx solid #ccc;
  border-radius: 10rpx;
}
.button {
  width: 100%;
  padding: 20rpx;
  background-color: #007aff;
  color: #fff;
  text-align: center;
  border: none;
  border-radius: 10rpx;
  cursor: pointer;
}
.button:active {
  background-color: #005bb5;
}
</style>