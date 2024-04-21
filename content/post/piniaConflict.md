+++
title = "使用piniaPluginPersistedstate导致多账号登录网页时信息冲突bug"
date = 2024-04-20T17:44:18+08:00
draft = false
author = "JJ"

categories = [
    "前端",
]

tags = [
    "vue",
]
image = "/cover/cover1.webp"
+++
> 我尝试使用pinia储存用户的登录状态，使用pinia.use(piniaPluginPersistedstate)，使得pinia能够永久化存储，以便当页面刷新时pinia数据不会丢失。但是当登录两个用户时，两个用户的数据发生了混杂。

## bug复现
为了复现以上bug,使用 以下demo

每次打开这个新页面都会在屏幕上显示一个随机得数字。使用pinia.use(piniaPluginPersistedstate)使这个页面刷新时，这个数字不会发生改变。但是现在出现了一个bug，当新开一个界面的时候，上一次的界面点击刷新会变成新界面的数字。

```html
<style scoped>
/*@/views/problem2/problem3.vue*/
</style>

<template>
<h1>problem3</h1>
  <div>
    <h2>number: {{ store.problem3Number.number }}</h2>
  </div>
</template>

<script setup lang="js">
import {onMounted} from "vue";
import useProblemStore from "@/stores/problemStore.js";

const store = useProblemStore()

onMounted(() => {
  if(!sessionStorage.getItem('hasEntered')){
    store.problem3Number.number = Math.floor(Math.random() * 100)
    sessionStorage.setItem('hasEntered',true)
  }
})
</script>
```

```js
//@/stores/index.js
import {createPinia} from "pinia";
import piniaPluginPersistedstate from "pinia-plugin-persistedstate";

const pinia = createPinia();

pinia.use(piniaPluginPersistedstate);
export default pinia;

//@/stores/problemStore.js
import {defineStore} from "pinia";
import {ref} from "vue";

const useProblemStore = defineStore(
    "problem",
    () => {
        const problem3Number = ref({
            number: 0,
        });
        return {
           problem3Number,
        };
    },
    {
        persist: true,
    }
);
export default useProblemStore;
```
## 解析
当我使用pinia.use(piniaPluginPersistedstate)时，实际上是将pinia存在了浏览器中的localstorage中，但是一个浏览器只有一个localstorage，所以两个不同的页面对pinia修改发生了数据的冲突，为了使这个问题解决。我们需要把pinia的储存更改到sessionstorage中，因为这个位置才是每个界面单独拥有的。
```js
import {createPinia} from "pinia";
import piniaPluginPersistedstate from "pinia-plugin-persistedstate";
import {watch} from "vue";

const pinia = createPinia();

pinia.use(({store}) => {
    const savedState = sessionStorage.getItem(store.$id)
    if (savedState) {
        store.$patch(JSON.parse(savedState))
    }

    watch(() => store.$state, (newState) => {
        sessionStorage.setItem(store.$id, JSON.stringify(newState))
    }, {deep: true})
})
// pinia.use(piniaPluginPersistedstate);
export default pinia;
```