<template>
  <div
    class="layout bgimage"
    :class="{
      dark: themeMode === 'dark',
      light: themeMode === 'light'
    }"
  >
    <div class="header">
      <input class="address" placeholder="请输入知乎章节地址" v-model="address" />
      <div class="config_list">
        <div class="config_item">
          <div class="label">首页行数：</div>
          <input type="number" class="value" placeholder="请设置首页行数" v-model="firstPageSize" />
        </div>
        <div class="config_item">
          <div class="label">最大行数：</div>
          <input type="number" class="value" placeholder="请设置一页最大行数" v-model="pageSize" />
        </div>
        <div class="config_item">
          <div class="label">最大页码：</div>
          <input type="number" class="value" placeholder="请设置截图最大页码" v-model="maxPage" />
        </div>
        <div class="config_item">
          <div class="label">尾页行数：</div>
          <input type="number" class="value" placeholder="请设置尾页行数" v-model="lastPageSize" />
        </div>
        <div class="config_item">
          <div class="label">主题模式：</div>
          <div class="value radio_wrapper">
            <label>
              <input
                type="radio"
                name="theme"
                value="light"
                :checked="themeMode === 'light'"
                @change="onThemeChange"
              />
              <span class="light_theme"></span>
            </label>
            <label>
              <input
                type="radio"
                name="theme"
                value="dark"
                @change="onThemeChange"
                :checked="themeMode === 'dark'"
              />
              <span class="dark_theme"></span>
            </label>
          </div>
        </div>
        <div class="config_item">
          <div class="label">网格背景：</div>
          <div class="value radio_wrapper">
            <label>
              <input
                type="radio"
                name="grid"
                value="1"
                :checked="gridMode === '1'"
                @change="onGridChange"
              />是
            </label>
            <label>
              <input
                type="radio"
                name="grid"
                value="0"
                :checked="gridMode === '0'"
                @change="onGridChange"
              />否
            </label>
          </div>
        </div>
        <div class="config_item">
          <div class="label">首页截图：</div>
          <div class="value radio_wrapper">
            <label>
              <input
                type="radio"
                name="first-page"
                value="1"
                :checked="firstPage === '1'"
                @change="onfirstPageChange"
              />是
            </label>
            <label>
              <input
                type="radio"
                name="first-page"
                value="0"
                :checked="firstPage === '0'"
                @change="onfirstPageChange"
              />否
            </label>
          </div>
        </div>
        <div class="config_item">
          <div class="label">尾页截图：</div>
          <div class="value radio_wrapper">
            <label>
              <input
                type="radio"
                name="last-page"
                value="1"
                :checked="lastPage === '1'"
                @change="onlastPageChange"
              />是
            </label>
            <label>
              <input
                type="radio"
                name="last-page"
                value="0"
                :checked="lastPage === '0'"
                @change="onlastPageChange"
              />否
            </label>
          </div>
        </div>
        <div class="config_item">
          <!-- 保持页面布局占位符 -->
        </div>
        <!-- <div class="config_item">
          <div class="label">页面比例：</div>
          <select class="value select_item">
            <option value="1440">3:4(1080:1440)</option>
            <option value="1620">2:3(1080:1620)</option>
            <option value="1500">自定义(1080:1500)</option>
          </select>
        </div> -->
        <div class="config_item full_item">
          <div class="label">敏感词：</div>
          <input class="value" placeholder="格式: AA|A*,BB|B*" v-model="sensitiveWords" />
        </div>
        <div class="config_item full_item">
          <div class="label">关键字：</div>
          <input class="value" placeholder="请输入关键字" v-model="watermark" />
        </div>
      </div>
      <button class="btn plain" @click="localHtml">本地链接</button>
      <button class="btn" @click="onDownload()">下载图片</button>
    </div>
    <div class="preview">
      <img :src="imgPreview" alt="" />
    </div>
    <div class="page_list" ref="pagesRef"></div>
    <div class="container" v-html="content"></div>
  </div>
</template>

<script setup>
import html2canvas from 'html2canvas'
import axios from 'axios'
import { ref, nextTick } from 'vue'
import { addGridBg, addWaterMark, tagAToDownload } from './utils'

// 默认使用用户名作为水印
const username = '不要芋泥'

// 内置敏感词
const defaultWords = [
  ['霸凌', '霸0'],
  ['死', 'si'],
  ['杀人', '沙人'],
  ['吸毒', '吸du'],
  ['强奸', '强*'],
  ['自杀', '自沙'],
  ['裸体', '果体'],
  ['全裸', '全果'],
  ['娇喘', '姣喘'],
  ['警察', '警查']
]
// 页面比例
// const aspectRatio = 3 / 4
// const height = 1080 / aspectRatio
const height = 1440
// const height = 1500
// 一行最大字数 34px:22 | 35px:21 | 36px:20
const maxTextLength = 22

// 是否打印网格背景(默认显示)
const gridMode = ref('0')
// 主题模式(默认浅色)
const themeMode = ref('light')
// 是否只截首页
const firstPage = ref('0')
// 是否只截尾页
const lastPage = ref('0')
// 水印内容, 为空时不打印水印
const watermark = ref(username)
// 敏感词, 多个词用问号拼接
const sensitiveWords = ref('')
// 首页显示行数
const firstPageSize = ref(5)
// 尾页显示行数
const lastPageSize = ref(5)
// 页面显示行数
const pageSize = ref(8)
// 截取的最大页码数
const maxPage = ref(60)
// 文章链接地址
const address = ref()
// const address = ref(
//   'https://www.zhihu.com/market/paid_column/1638186406705827840/section/1641877398021672960'
// )

// 爬虫获取的知乎文章内容
const content = ref(null)
// 文章解析后要插入的 DOM 实例
const pagesRef = ref(null)
// 预览地址
const imgPreview = ref('')
// 是否显示网格
const onGridChange = (e) => {
  gridMode.value = e.target.value
}
// 主题切换
const onThemeChange = (e) => {
  themeMode.value = e.target.value
}
// 是否只截首页
const onfirstPageChange = (e) => {
  firstPage.value = e.target.value
  lastPage.value = '0'
}
// 是否只截尾页
const onlastPageChange = (e) => {
  lastPage.value = e.target.value
  firstPage.value = '0'
}

// 本地链接
const localHtml = () => {
  const { protocol, hostname } = window.location
  address.value = `${protocol}//${hostname}:4000/local.html`
  onDownload()
}

// 下载事件
const onDownload = () => {
  if (!address.value || !address.value.trim()) {
    alert('请输入文章地址')
    return
  }
  pagesRef.value.innerHTML = ''
  const { protocol, hostname } = window.location
  axios({
    method: 'get',
    url: `${protocol}//${hostname}:5002/zhihu`,
    params: {
      url: address.value
    }
  })
    .then((res) => {
      let contentValue = res.data
      // 处理用户敏感词
      const customWords = []
      if (sensitiveWords.value.trim()) {
        sensitiveWords.value.split(',').forEach((text) => {
          customWords.push(text.split('|'))
        })
      }
      // 合并内置敏感词
      const words = [].concat(defaultWords, customWords)
      words.forEach((w) => {
        contentValue = contentValue.replaceAll(w[0], w[1])
      })

      content.value = contentValue
      nextTick(() => {
        const isOnlyLast = lastPage.value === '1'
        insertPages(isOnlyLast)
        downloadImages(isOnlyLast)
      })
    })
    .catch((err) => {
      console.log(err)
      alert(err)
    })
}
// 下载图片
const downloadImages = async (isOnlyLast) => {
  const pageTags = document.querySelectorAll('.page')
  const pageList = Array.from(pageTags).slice(0, maxPage.value || 60)
  pageList.forEach((p, i) => {
    // 只下载首页
    if (firstPage.value === '1') {
      i < 1 && getImage(p, i + 1)
      return
    }
    // 只下载最后一页
    if (isOnlyLast) {
      if (i === pageList.length - 1) {
        getImage(p, i + 1)
      }
      return
    }
    // 下载所有
    setTimeout(() => {
      getImage(p, i + 1)
    }, i * 500)
  })
}

// 解析 pages DOM
function insertPages(isOnlyLast) {
  // 所有的 P 标签
  const pList = document.querySelectorAll('p')
  // 组装一页的 P 标签
  let container = []
  // 一页的 P 数量
  let pCount = 0
  // 页码
  let pageNumber = 1
  pList.forEach((p, i) => {
    container.push(p)
    const textLength = p.innerText.length

    // 大概比例为
    pCount = pCount + (Math.ceil(textLength / maxTextLength) + 1) * 0.5

    // 手动设定最后一页

    // 到达最大页码退出
    if (pageNumber > maxPage.value + 1) {
      return
    }

    // 最大行数限制条件
    let maxPCount = pCount >= pageSize.value

    // 单独下载设置最后一页时
    if (isOnlyLast) {
      if (pageNumber === maxPage.value || i === pList.length - 1) {
        maxPCount = pCount >= lastPageSize.value
      }
    }

    // 第一页只截 firstPageSize 行 || 当页已达到最大行数 || 最后一页
    if (
      (pageNumber === 1 && i === firstPageSize.value - 1) ||
      maxPCount ||
      i === pList.length - 1
    ) {
      // 组装页
      const div = document.createElement('div')
      div.innerHTML = `<span class="page_number"> - ${pageNumber} -</span>`
      div.setAttribute('class', 'page')
      div.setAttribute('style', `height:${height}px`)
      // 将组装好的虚拟 page 添加到真实 dom 中
      container.forEach((c) => div.appendChild(c))
      pagesRef.value.appendChild(div)
      container = []
      pCount = 0
      pageNumber++
    }
  })
  content.value = ''
}
// 生成图片
function getImage(targetEle, index) {
  return html2canvas(targetEle, {
    width: 1080,
    height: height
    // backgroundColor: 'transparent'
  }).then((canvas) => {
    // 添加网格背景
    gridMode.value === '1' && addGridBg(canvas, themeMode.value)

    // 添加关键字水印
    const text = watermark.value.trim()
    text && addWaterMark(canvas, text, themeMode.value)

    const base64Url = canvas.toDataURL('image/png')
    // jpeg 格式可以设置图片质量
    // const base64Url = canvas.toDataURL('image/jpeg', 0.8)

    // 调试用代码
    imgPreview.value = base64Url

    const result = { url: base64Url, title: `${themeMode.value}_${index}.png` }
    tagAToDownload(result)
    return base64Url
  })
}
</script>
