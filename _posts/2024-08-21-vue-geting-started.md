---
title: Vue2 快速入门
description:  参考别人写的快速入门文章，自己记录下实践过程
categories: [Vue, 快速入门]
tags: [前端]
author: [xiao_e]
---

本文主要记录总结性的知识点，初学者请参考文末链接。

## 总结主要知识点

### 组件

定义组件：[components/quiNav.vue](https://github.com/cxyxq/Vue2-Started/blob/main/src/components/quiButton.vue)

组件支持自定义事件:

```javascript
this.$emit('navClickEvent');
```


使用组件:  [pageQuiButton.vue](https://github.com/cxyxq/Vue2-Started/blob/main/src/pages/pageQuiButton.vue)

```
<template>
    <div id="pageQuiButton">
		<!-- 使用自定义的btn组件  -->
		<qui-btn msg="确定" class="small" v-on:btnClickEvent="doSomething"> // [3]使用组件
			<img slot="icon" class="icon" src="" />
		</qui-btn>
	</div>
</template>
<script>
	import quiBtn from '../components/quiButton.vue' // [1]引入组件
	export default {
		name: 'pageQuiButton',
		components: {
			'qui-btn': quiBtn // [2]声明组件
		},
		methods: {
			doSomething: function() {
				alert('父组件pageQuiButton自定义事件');
			}
		}
	}
</script>
```

> pageQuiButton.vue在使用qui-btn组件时，可以自定义click事件：
> v-on:btnClickEvent="doSomething" 改变事件逻辑
{: .prompt-tip}


### 路由

```javascript
//定义路由地址

import Vue from 'vue'
import Router from 'vue-router'
import index from '../pages/index.vue' //引入首页页面
import pageQuiButton from '../pages/pageQuiButton.vue' //引入相关页面
import pageQuiList from '../pages/pageQuiList.vue' //引入相关页面
import pageQuiNav from '../pages/pageQuiNav.vue' //引入相关页面

Vue.use(Router)

export default new Router({
  routes: [
    {
      path: '/',
      name: 'index',
      component: index
    },
    {
      path: '/btn', //定义访问url
      name: 'btn',
      component: pageQuiButton
    },
    {
      path: '/list',
      name: 'list',
      component: pageQuiList
    },
    {
      path: '/nav',
      name: 'nav',
      component: pageQuiNav
    }
  ]
})

```

## 代码库


## 参考文档
1. [包学会之浅入浅出Vue.js：开学篇](https://cloud.tencent.com/developer/article/1020337)
2. [包学会之浅入浅出Vue.js：升学篇](https://cloud.tencent.com/developer/article/1020338)
3. [包学会之浅入浅出Vue.js：结业篇](https://cloud.tencent.com/developer/article/1020416)
4. [Vue2官网手册](https://v2.cn.vuejs.org/v2/guide/)