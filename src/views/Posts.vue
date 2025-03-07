<script setup lang="ts">
import { reactive, ref } from 'vue'
import { useInfiniteScroll } from '@vueuse/core'
import { useLeanCloudStore } from '../stores/leanCloud'
import { getCounter, getPosts } from '../api'
import MarkdownIt from '../components/MarkdownIt.vue'
import type { Post } from '../types/index'

const posts: Post[] = reactive([])
let curPage = 1 // 当前页码，不需要响应式
const noMore = ref(false) // 没有更多文章，用于取消监听触底

const { needLeanCloud } = useLeanCloudStore()

/**
 * 获取更多文章
 * @param currentPage 当前页码
 * @returns 文章列表
 */
async function moreData(currentPage: number) {
  try {
    const res = await getPosts({ page: currentPage, pageSize: 12 })
    return res
  }
  catch (error) {
    console.error('Error fetching data:', error)
    return []
  }
}

/**
 * 设置文章浏览次数
 * 使用到了上面的变量 posts
 */
async function setPostTimes() {
  try {
    const ids = posts.map((post: Post) => post.id)
    const times = (needLeanCloud && posts.length > 0)
      ? await getCounter(ids) as { [key: string]: number }
      : {}
    if (times) {
      posts.forEach((post: Post) => {
        post.times = times[post.id] || 1
      })
    }
    else {
      console.error('getCounter failed')
    }
  }
  catch (error) {
    console.error('Error fetching data:', error)
  }
}

/**
 * 监听滚动事件，触底时加载更多文章
 */
useInfiniteScroll(
  document,
  async () => {
    const res = await moreData(curPage++)
    // 没有更多数据
    if (res.length === 0) {
      noMore.value = true
      return
    }
    posts.push(...res)
    setPostTimes()
  },
  {
    distance: 120,
    canLoadMore: () => !noMore.value,
  },
)
</script>

<template>
  <div class="home">
    <main v-if="posts?.length > 0" class="select-none line-height-none slide-enter-1000">
      <article v-for="post in posts" :key="post.id" v-slide-in class="group mb-12 cursor-pointer">
        <router-link :to="{ name: 'post', params: { num: Number(post.num) } }">
          <div v-if="post.milestone?.title || post.comments > 0" class="m-y-3 flex items-center">
            <span class="mr-3 border-rd bg-gray-100 p-x-2 p-y-1 font-size-3.4 text-gray-400 dark:bg-gray-800 dark:text-gray-600 group-hover:text-gray-500">{{ post.milestone.title }}</span>
            <span v-if="post.comments > 0" class="mr-3 text-gray-400 dark:text-gray-600 group-hover:text-gray-500"><i class="fa-regular fa-comments mr-1.4" />{{ post.comments }}</span>
            <span class="text-gray-400 dark:text-gray-600 group-hover:text-gray-500"><i class="fa-regular fa-eye mr-1.4" />{{ post.times }}</span>
          </div>
          <h2 class="m-0 font-size-8 text-gray-600 font-bold dark:text-gray-400 group-hover:text-gray-800 dark:group-hover:text-gray-300">
            {{ post.title }}
          </h2>
          <p class="m-y-2 font-size-4 text-gray-400 line-height-normal line-2 dark:text-gray-500 group-hover:text-gray-500 dark:group-hover:text-gray-400">
            <MarkdownIt :content="post.summary" />
          </p>
          <p class="m-y-4 font-size-3.4 text-gray-400 dark:text-gray-600 group-hover:text-gray-500">
            <span class="mr-2">{{ post.date }}</span>
            <template v-if="post.labels.length > 0">
              <span v-for="label in post.labels" :key="label.id" class="mr-1.4"><i class="fa-solid fa-hashtag mr-0.6 font-size-3 text-gray-300 dark:text-gray-700" />{{ label.name }}</span>
            </template>
          </p>
        </router-link>
      </article>
      <p v-if="!noMore" v-slide-in class="text-center">
        <i class="fa-solid fa-circle-notch fa-spin text-gray-400" />
      </p>
      <p v-else class="text-center text-gray-400">
        没有更多文章了~
      </p>
    </main>
    <p v-else class="animate-blink animate-iteration-infinite text-center font-size-4 text-gray-400">
      文章加载中……
    </p>
  </div>
</template>
