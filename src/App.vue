<template>
  <div id="app">

    <ul>
      <li class="liList" v-for="item in items" :key="item.id">{{ item.text }}</li>
    </ul>
    <div v-if="!noContent" ref="monitor" class="text"> {{ loading ? '加载中...' : '下滑加载更多' }}</div>
    <div v-else class="text">已经到底啦</div>
  </div>
</template>

<script setup>
import { onUnmounted } from 'vue'
import { onMounted } from 'vue'
import { ref, reactive, watch } from 'vue'
let loading = ref(false)//节流
let noContent = ref(false)//定义变量判断获取的内容是否存在
let items = reactive([])//数据列表
let page = ref(0)//页码
const pageSize = 15//每页元素个数
let monitor = ref(null)//被监视的div用于触发滚动
let observer = null//定义IntersectionObserver，在销毁时使用
onMounted(() => {
  //形参传入需要监视的元素
  observer = new IntersectionObserver((element) => {
    //函数中的形参为IntersectionObserverEntry对象
    console.log(element[0]);
    if (element[0].isIntersecting) {
      loadMore()
    }
  }, {
    root: null,
    rootMargin: '0px',
    threshold: 1
  })
  //如果要见是的元素存在，就启动监视
  if (monitor) {
    observer.observe(monitor.value)
  }
})


//监听noContent，如果无内容加载就销毁IntersectionObserver
watch(noContent, () => {
  //如果IntersectionObserver实例存在
  if (observer) {
    //销毁实例
    observer.disconnect()
    observer = null//释放引用
    console.log('observer已销毁');
  }
})
//页面关闭销毁IntersectionObserver实例
onUnmounted({
  if(observer) {
    //销毁实例
    observer.disconnect()
    observer = null//释放引用
    console.log('observer已销毁');
  }
})
//模拟发送请求获取数据
function getMoreData(page) {
  //模拟发送请求
  if (page >= 3) {
    console.log('已无最新数据');
    noContent.value = true
    return
  }
  const promise = new Promise((resolve, reject) => {
    setTimeout(() => {
      resolve('加载数据~')
    }, 1000);
  }).then(res => {
    //请求成功处理数据
    console.log(res);
    return Array(pageSize).fill().map((_, index) => {
      return {
        id: page * 15 + index + 1,
        text: `第${page * 15 + index + 1}个元素`
      }
    })
  })
  return promise
}
//加载更多代码
const loadMore = async () => {
  // console.log(loading.value);

  //如果loading为true（正在加载）或noContent为true（已无内容）则return
  if (loading.value || noContent.value) return
  //将loading设置为正在加载
  loading.value = true
  try {
    var data = await getMoreData(page.value)
    if (data.length) {
      items.push(...data)
      //获取数据后让页码自动+1 
      page.value++
    }
  } catch (err) {
    //错误处理函数
    // console.log('加载错误' + err);
  } finally {
    //无论成功失败都取消loading状态
    loading.value = false
  }


}


</script>

<style>
#app {
  width: 90%;
  height: auto;
  margin: 0 auto;
  background-color: #f5f5f5;
  padding: 10px 0;
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

.text {
  text-align: center;

}
</style>