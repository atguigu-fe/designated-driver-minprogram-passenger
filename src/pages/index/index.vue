<template>
  <tm-app>
    <map
      id="map"
      :longitude="mapProps.longitude"
      :latitude="mapProps.latitude"
      scale="15"
      :enable-traffic="false"
      :show-location="true"
      :enable-poi="true"
      class="map"
    >
      <!-- <cover-view @click="moveToLocationHandle()" class="location">
        <theme-icon custom-prefix="iconfont" type="iconfontditudingwei" size="30"></theme-icon>
      </cover-view> -->

      <cover-view class="location-panel">
        <cover-view class="location-panel-container">
          <cover-view class="from els" @click="setFromOrToLocation('from')">
            <cover-view>
              {{ takeCarInfo.from.address || '请选择出发地' }}
            </cover-view>
          </cover-view>
          <cover-view class="to els" @click="setFromOrToLocation('to')">
            <cover-view>{{ takeCarInfo.to.address || '请选择目的地' }}</cover-view>
          </cover-view>
        </cover-view>
      </cover-view>
    </map>
    <!-- <view class="location-panel">
      <tm-sheet :round="3" :shadow="2">
        <view class="from" @tap="setFromOrToLocation('from')">
          <tm-input disabled v-model="takeCarInfo.from.address" :followTheme="false" placeholder="请选择出发地" :margin="[0, 24]"></tm-input>
        </view>
        <view class="to" @tap="setFromOrToLocation('to')">
          <tm-input disabled v-model="takeCarInfo.to.address" :followTheme="false" placeholder="请选择目的地" :margin="[0, 24]"></tm-input>
        </view>
      </tm-sheet>
    </view> -->
    <tabbar-nav>111</tabbar-nav>
  </tm-app>
</template>
<script setup lang="ts">
import { ref } from 'vue'
import { useQqMapSdk } from '@/hooks/useQqMapSdk'
import { IMapProps } from '@/api/index/types'
import { useTakeCarInfoStore } from '@/store/modules/takeCarInfo'
import { QqMapkey } from '@/config/config'
import { findCustomerCurrentOrder } from '@/api/order'
import { getToken } from '@/utils/storage'
//#region <map相关>
const map = uni.createMapContext('map')
// 选择地点插件
const chooseLocation = requirePlugin('chooseLocation')
// 打车相关信息仓库
const takeCarInfo = useTakeCarInfoStore()
const mapProps = ref<Pick<IMapProps, 'longitude' | 'latitude'>>({
  // 中心经度
  longitude: 116.397505,
  // 中心纬度
  latitude: 39.908675
})
// 选择地点的标识
const flag = ref<'from' | 'to'>('from')

//  回到初始位置
function moveToLocationHandle() {
  map.moveToLocation(mapProps.value)
}
// 设置出发地、目的地位置
function setFromOrToLocation(type: 'from' | 'to') {
  // 设置标识
  flag.value = type
  const key = QqMapkey
  const referer = import.meta.env.VITE_APP_TITLE //调用插件的app的名称
  const location = JSON.stringify({
    latitude: mapProps.value.latitude,
    longitude: mapProps.value.longitude
  })
  uni.navigateTo({
    url: `plugin://chooseLocation/index?key=${key}&referer=${referer}&location=${location}`
  })
  console.log(key, referer)
}

// 样式
//#endregion
onShow(async () => {
  // 隐藏tabbar
  uni.hideTabBar()
  if (getToken()) {
    // 判断已经存在订单，如果存在订单，则提示是否去往导航页
    const { data } = await findCustomerCurrentOrder()
    if (data.isHasCurrentOrder) {
      uni.showModal({
        title: '提示',
        content: '您有未完成的订单，是否去往导航页？',
        success: function (res) {
          if (res.confirm) {
            uni.navigateTo({
              url: '/pages/creatOrder/creatOrder?orderId=' + data.orderId
            })
          }
        }
      })
      return
    }
  }
  console.log('onShow-chooseLocation', chooseLocation)
  const location = chooseLocation.getLocation() // 如果点击确认选点按钮，则返回选点结果对象，否则返回null
  console.log('location', location)
  // 如果没有选择地点，重置takeCarInfo中所有信息
  if (!location) {
    takeCarInfo.$reset()
    return
  }
  // 如果有选择地点，则设置出发地或目的地
  if (flag.value === 'from') {
    takeCarInfo.from = location
  } else {
    takeCarInfo.to = location
  }
  // 如果出发地和目的地都有值，则跳转到下单页面
  if (takeCarInfo.from.address && takeCarInfo.to.address) {
    // 判断已经存在订单，如果存在订单，则提示是否去往导航页
    const { data } = await findCustomerCurrentOrder()
    if (data.isHasCurrentOrder) {
      uni.showModal({
        title: '提示',
        content: '您有未完成的订单，是否去往导航页？',
        success: function (res) {
          if (res.confirm) {
            uni.navigateTo({
              url: '/pages/creatOrder/creatOrder?orderId=' + data.orderId
            })
          } else if (res.cancel) {
            // todo 方便调试生成订单 正式环境注释掉
            uni.navigateTo({
              url: '/pages/creatOrder/creatOrder'
            })
          }
        }
      })
      return
    } else {
      uni.navigateTo({
        url: '/pages/creatOrder/creatOrder'
      })
    }
  }
})
onLoad(() => {
  //   获取当前位置信息
  uni.getLocation({
    type: 'gcj02',
    success: function (res) {
      mapProps.value.longitude = res.longitude
      mapProps.value.latitude = res.latitude
    }
  })
})
onHide(() => {
  chooseLocation.setLocation(null)
})
onUnload(() => {
  chooseLocation.setLocation(null)
})
</script>

<style scoped lang="scss">
.map {
  width: 100%;
  height: 100vh;
}
.location {
  position: absolute;
  right: 45rpx;
  bottom: 450rpx;
  width: 60rpx;
  height: 60rpx;
}
.location-panel {
  position: absolute;
  width: 100%;
  bottom: 160rpx;
  display: flex;
  justify-content: center;
}

.location-panel-container {
  width: 560rpx;
  background-color: #fff;
  padding: 30rpx 20rpx;
  border-radius: 10rpx;
}

.from {
  margin-bottom: 30rpx;
}

.from,
.to {
  width: 520rpx;
  height: 40rpx;
  background-color: #f5f5f5;
  color: #808080;
  padding: 10rpx 20rpx;
  font-size: 28rpx;
  line-height: 40rpx;
  border-radius: 10rpx;
}

.els {
  text-overflow: ellipsis;
  overflow: hidden;
  white-space: nowrap;
}
</style>
