### 创建项目

```shell
npm create vite
## 选择 vue 和 ts 模版

npm i 

npm i pinia
```

* 删除模版代码
* 导入静态页面html和css 到assets目录下
* 引入pinia 依赖，修改main.ts文件 

```ts
import { createApp } from "vue";
import "./assets/style.css";
import App from "./App.vue";
import { createPinia } from "pinia";

const pinia = createPinia();
createApp(App).use(pinia).mount("#app");

```

### 拆分静态html 文件 为vue组件

在asset目录下赋值html body内容到App.vue的  templete标签下

```vue
// App.vue
<template>
  <div>
    <h2>记账</h2>
    <div class="container">
      <Balance></Balance>
      <History></History>
      <AddTransaction></AddTransaction>
    </div>
  </div>
</template>

<script setup lang="ts">
import History from './components/History.vue';
import Balance from './components/Balance.vue';
import AddTransaction from './components/AddTransaction.vue';
</script>

```

创建store/transcations.ts

* 创建记账的数据模型Transcation
* 创建响应式reactive 的Transcation对象
* 提供添加记账addTranscation和删除记账deleteTranscation的操作函数

```ts
import { defineStore } from "pinia";
import { reactive } from "vue";

interface Transcation {
  id: number;
  amount: number;
  title: string;
  type: "plus" | "minus";
}
let Id = 0;
export const useTranscationStore = defineStore("transcation", () => {
  const transcations = reactive<Transcation[]>([
    {
      id: 1,
      amount: 100,
      title: "Salary",
      type: "plus",
    },
    {
      id: 2,
      amount: 50,
      title: "Food",
      type: "minus",
    },
    {
      id: 3,
      amount: 20,
      title: "Transport",
      type: "minus",
    },
  ]);

  const addTranscation = (title: string, amount: number) => {
    let id = Id++;
    const transcation: Transcation = {
      id,
      amount,
      title,
      type: amount > 0 ? "plus" : "minus",
    };
    transcations.push(transcation);
    return transcation;
  };

  const deleteTranscation = (id: number) => {
    const index = transcations.findIndex(
      (transcation) => transcation.id === id
    );
    if (index !== -1) {
      transcations.splice(index, 1);
    }
  };

  return { transcations, addTranscation, deleteTranscation };
});

```

### 创建单元测试

安装vitest

```shell
npm i vitest -D
```

创建transcation.spec.ts

```ts
import { beforeEach, it, expect, describe } from "vitest";
import { useTranscationStore } from "./transcation";
import { setActivePinia, createPinia } from "pinia";

describe("transcation store", () => {
  beforeEach(() => {
    setActivePinia(createPinia());
  });
  describe("add transaction", () => {
    it("should add a new transcation", () => {
      const store = useTranscationStore();
      const newTranscation = store.addTranscation("New transcation", 100);
      expect(newTranscation).toEqual(
        expect.objectContaining({
          title: "New transcation",
          amount: 100,
          type: "plus",
        })
      );
    });
  });
});

```

package.json添加test

```json
 "scripts": {
    "dev": "vite",
    "build": "vue-tsc -b && vite build",
    "preview": "vite preview",
    "test": "vitest"
  },
```

### 将数据绑定到HistoryVue

* v-for遍历store.transcations  展示title 和amount
* scripte脚本中创建const store = useTranscationStore()

```vue
<template>
    <div>
        <h3>History</h3>
        <ul class="list">
            <template v-for="transcation in store.transcations" :key="transcation.id">
                <li :class="transcation.type">
                    {{ transcation.title }}
                    <span>{{ transcation.amount }}</span>
                    <button class="delete-btn" @click="handleDelete(transcation.id)">x</button>
                </li>
            </template>
        </ul>
    </div>
</template>

<script setup lang="ts">
import { useTranscationStore } from '../store/transcation';

const handleDelete = (id: number) => {
    store.deleteTranscation(id)
}
</script>

<style lang="scss" scoped></style>
```









