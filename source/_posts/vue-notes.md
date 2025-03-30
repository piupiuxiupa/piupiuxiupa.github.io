---
title: Vue Notes
date: 2025-03-30 16:49:47
categories: Notes
excerpt: Vue's notes
---

## 变量

```html
<script setup>
import { ref } from 'vue';

const str = ref('hello world')
console.log(str.value) // 在script标签中可以直接使用
</script>

<template>
  // 双大括号绑定变量
  <div>{{ str }}</div>
</template>
```

## 事件监听

```html

<script setup>
import { ref } from 'vue';

const str = ref('hello world')

function add(){
    console.log(str.value)
}
</script>

<template>
  // 点击事件
  <div @click="add">{{ str }}</div>
  // 鼠标移入事件
  <div @mouseenter="add">{{ str }}</div>
  // 鼠标移出事件
  <div @mouseleave="add">{{ str }}</div>
</template>
```

## 双向绑定

```html
<script setup>
import { ref } from 'vue';
const str = ref('hello world')
function add(){
    console.log(str.value)
    // 修改变量的值
    // 点击事件后，输入框的值会自动修改为888
    str.value = '888'
}
</script>
<template>
  // 输入框的值会自动绑定到str变量上
  <input type="text" v-model="str">
  <div @click="add">{{ str }}</div>
</template>
```

## 动态绑定

```html
<script setup>
import { ref } from 'vue';
const str = ref('completed')
function add(){
    console.log(str.value)
    str.value = 'item'
}
</script>
<template>
  // 动态绑定class
  <div :class="str">{{ str }}</div>
  <div @click="add">{{ str }}</div>
</template>
```

```html
<script setup>
import { ref } from 'vue';
const str = ref(true)
function add(){
    console.log(str.value)
    str.value = !str.value
}
</script>
<template>
  // 动态绑定class
  <div @click="add">{{ str }}</div>
  <div :class="[str ? 'completed' : 'item']"></div>
</template>
```

## 列表渲染

```html
<script setup>
import { ref } from 'vue';

const list = ref([
    {
        id: 1,
        name: '张三',
        age: 18
    },
    {
        id: 2,
        name: '李四',
        age: 20
    },
    {
        id: 3,
        name: '王五',
        age: 22
    }
])
</script>
<template>
  // 列表渲染
  <div v-for="(item, index) in list" :key="item.id">
    <div>{{ item.name + index }}</div>
    <div>{{ item.age + index }}</div>
  </div>
</template>
```

## 监听器

```html
<script setup>
import { ref, watch } from 'vue';
const str = ref('')      
function add(newValue, oldValue){
    console.log(newValue, oldValue)
}
// 监听变量的变化
watch(str, add)  
</script>
<template>
<input type="text" v-model="str">
  <div @click="add">{{ str }}</div>
</template>
```

## 监听对象

```html
<script setup>
import { ref, watch } from 'vue';
const obj = ref({
    name: '张三',
    age: 18
})
function add(newValue, oldValue){
    console.log(newValue, oldValue)
}
// 监听对象的变化
watch(obj, add, {deep: true})
</script>
<template>
<input type="text" v-model="obj.name">
  <div @click="add">{{ obj.name }}</div>
</template>
```

## 条件渲染

```html
<script setup>
import { ref } from 'vue';
const bool = ref(true)
</script>
<template>
  // 条件渲染
  // v-show 会创建div，但是会根据bool的值来显示或者隐藏div
  <div v-show="bool">
    <div>Hello vue</div>
  </div>
  // v-if 会根据bool的值来判断是否创建div
  <div v-if="bool">
    <div>Hello vue</div>
  </div>
</template>
```

## 组件

一个vue文件可以看做一个组件

1. 父组件给子组件传值

    ```html
    <script setup>
    import {defineProps} from 'vue'
    const props = defineProps(['text'])
    </script>
    <template>
      // 组件
      <div class='button'>
        {{ props.text }}
      </div>
    </template>
    <style scoped>
    .button{
      width: 100px;
      height: 100px;
      background-color: red;
    }
    </style>
    ```

    引入组件

    ```html
    <script setup>
    // 假设文件叫myButton.vue，在components目录下
    import myButton from './components/myButton.vue'
    </script>
    <template>
      <div>
        // 父组件给子组件传值
        <myButton text="hello" />
      </div>
    </template>
    ```

2. 子组件往父组件传值

    ```html
    <script setup>
    import {defineProps, defineEmits} from 'vue'
    const props = defineProps(['text'])
    const emit = defineEmits(['ok']) 
    function send(){
        emit('ok', 'hello')
    }
    </script>
    <template>
      <div class='button' @click="send">
        {{ props.text }}
      </div>
    </template>
    <style scoped>
    .button{
      width: 100px;
      height: 100px;
      background-color: red;    
    }
    </style>
    ```

    父组件接收子组件的值

    ```html
    <script setup>
    // 假设文件叫myButton.vue，在components目录下
    import myButton from './components/myButton.vue'
    function add(str){
        console.log(str)
    }
    </script>
    <template>
      <div>
        // 给子组件传值
        <myButton @ok="add" text="Hi" />
      </div>
    </template>
    ```

## axios

```html
<script setup>
import { ref } from 'vue';
import axios from 'axios'

const value = ref('')
const list = ref([])

async function getList(){
  const res = await axios({
    url: "https:// example.com",
    method: "GET",
  })
  list.value = res.data.list
  console.log(res, "Data from backend")

}

async function getData(){
  const res = await axios({
    url: "https://xxx.com",
    method: "POST",
    data:{
        value: value.value,
        flag: false
    }   
  })
  getList()
}

async function update(id){
    await axios({
      url: "https://example.com/update",
      method: "POST",
      data: {
        id,
      }
    })
    getList()
}
// 调用函数
getData()
</script>

<template>
  <div v-for="(item, index) in list" :class="[item.flag ? 'completed' : 'item' ]">
    <div>
        <input @click="update(item.id)" v-model="item.flag" type="checkbox" />
        <span>{{ item.value }}</span>
    </div>
  </div>
</template>
```
