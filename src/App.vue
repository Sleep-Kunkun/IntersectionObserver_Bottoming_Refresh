<template>
  <div id="app">
    <ul>
      <li class="liList" v-for="item in items" :key="item.id">{{ item.text }}</li>
    </ul>
    <div v-if="!noMore" class="sentry" ref="sentry">{{ loading ? "正在加载数据..." : "下滑加载更多" }}</div>
    <div v-else class="sentry">已无更多数据</div>
  </div>
</template>

<script setup>
import { onUnmounted } from "vue";
import { onMounted, reactive, ref, watch } from "vue";
//数据加载限制条件
const LIMIT = {
  MAX_Page: 3,//最大页码数(模拟)
  MAX_retryCount: 3,//最多重试请求次数
  MIN_TriggerTime: 400//最短连续触发时间 0.4s
}
let sentry = ref(null)//获取需要监视的哨兵元素
let page = ref(0)//页码
let pageSize = 15//每页的数据个数
let items = reactive([])//数据列表
let loading = ref(false)//节流（是否正在加载）
let lastLoadingTime = ref(0)//节流（记录最后加载的时间）
let noMore = ref(false)//判断数据是否加载完毕
//创建IntersectionObserver实例
var observe = new IntersectionObserver((entries) => {
  //仅仅在被监视元素可见时触发
  if (entries[0].isIntersecting) {
    //新增数据
    getMore()
  }
}, {
  root: null,//默认设置为Window窗口视图
  rootMargin: "0px",//定义根组件的边距 用于扩展/缩小检测范围
  threshold: 1//触发比例阈值 可以为数组[0，0.25，0.5，1]
})

//在dom挂载后对其进行监视
onMounted(() => {
  //如果dom元素存在则对其进行监视
  if (sentry) {
    observe.observe(sentry.value)//开始监视
  }
})
//监视noMore数据状态
watch(noMore, () => {
  if (observe) {
    observe.disconnect()
    observe = null//释放引用
    console.log('observer已销毁');
  }
})
//页面销毁时销毁IntersectionObserver实例
onUnmounted(() => {
  if (observe) {
    observe.disconnect()
  }
})
//获取数据（模拟）
const getData = (page) => {
  if (page >= LIMIT.MAX_Page) {
    console.log("没有更多数据");
    return
  }
  const promise = new Promise((resolve) => {
    //Array()创建pageSize（15）个元素的空数组
    //.fill()将这15个元素赋值为undefined
    //.map()对每个元素进行遍历
    //使用setTimeout模拟网络数据请求时间
    setTimeout(() => {
      let data = Array(pageSize).fill().map((_, index) => {
        return {
          id: page * pageSize + index + 1,
          text: '第' + (page * pageSize + index + 1) + '个元素'
        }
      })
      resolve(data)
    }, 800)
  })
  return promise
}

//加载更多数据        retryCount记录重试次数
const getMore = async (retryCount = 0) => {
  //记录当前时间
  let nowTime = Date.now()
  //调试日志
  console.log({
    '[时间节流状态]': {
      当前时间: nowTime,
      上次加载时间: lastLoadingTime.value,
      时间差: nowTime - lastLoadingTime.value,
      是否进行时间节流: (nowTime - lastLoadingTime.value < 1000)
    },
    '[加载节流状态]': {
      加载状态: loading.value,
      是否进行加载节流: loading.value
    },
    '[数据状态]': {
      数据是否加载完毕: noMore.value,
      是否进行拦截: noMore.value
    }
  })
  /*
  1.正在加载
  2.连续触发触底
  3.数据加载完毕
  则退出函数
  */
  if (loading.value || nowTime - lastLoadingTime.value < LIMIT.MIN_TriggerTime || noMore.value) return
  //进入函数将加载状态改为true
  loading.value = true
  try {
    let data = await getData(page.value)
    //判断data是否获取到数据
    if (data) {
      //将获取的数据渲染在页面上
      items.push(...data)
      //页码数加1
      page.value++
      //记录最后一次加载的时间
      lastLoadingTime.value = Date.now()
    } else {
      //将无更多数据状态改为true
      noMore.value = true
    }
    //修改加载状态改为false
    loading.value = false
  } catch (err) {
    //处理错误信息
    //获取数据失败时修改loading
    loading.value = false
    //允许三次重试
    if (retryCount < LIMIT.MAX_retryCount) {
      console.log("重新加载" + (retryCount + 1) + "次");
      //添加随机抖动减少服务器压力
      const waitTime = 1000 //定义最小等待时间1秒
      let jitter = Math.random() * 1000 //0-1秒随机时间
      //通过await Promise设置等待时长
      await new Promise(resolve => setTimeout(resolve, (waitTime + jitter)))
      //递归调用加载函数重试请求
      return getMore(retryCount + 1)
    }
    console.log(err)
  }
}

</script>

<style>
#app {
  width: 90%;
  margin: 0 auto;
  background-color: #f5f5f5;
  padding: 10px 0;
}

.sentry {
  text-align: center;
}

ul {
  display: flex;
  flex-wrap: wrap;
  list-style: none;
  gap: 10px;
  justify-content: space-between;
}

.liList {
  flex: 0 1 calc(33% - 10px);
  text-align: center;
  height: 300px;
  line-height: 300px;
  font-size: 24px;
  font-weight: 600;
  background-color: #ccbb;
}
</style>