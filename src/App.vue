<template>
  <!-- <img alt="Vue logo" src="./assets/logo.png">-->
  <HelloWorld :obj="obj" :title="title" />
  <p :style="{color:obj.color}">Message in parent: 《{{obj.msg}}》</p>
  <p :style="{color:obj.color}">Title in parent: 《{{title}}》</p>
  <TempateRef />

  <section id="app" class="todoapp">
    <header class="header">
      <!-- <h1>todos</h1> -->
      <input
        class="new-todo"
        placeholder="What needs to be done?"
        autocomplete="off"
        autofocus
        v-model="todoInput"
        @keyup.enter="addTodo"
      />
    </header>

    <section class="main" v-show="totalCount">
      <input id="toggle-all" class="toggle-all" type="checkbox" v-model="allDone" />
      <label for="toggle-all">Mark all as complete</label>
      <ul class="todo-list">
        <li
          v-for="(todo) in matchedTotos"
          :key="todo"
          :class="{editing: todo===currentEdit, completed: todo.completed}"
        >
          <div class="view">
            <input class="toggle" type="checkbox" v-model="todo.completed" />
            <label @dblclick="editTodo(todo)">{{todo.text}}</label>
            <button class="destroy" @click="remove(todo)"></button>
          </div>
          <input
            v-editing-focus="todo === currentEdit"
            class="edit"
            type="text"
            v-model="todo.text"
            @keyup.enter="doneEdit(todo)"
            @blur="doneEdit(todo)"
            @keyup.esc="cancelEdit(todo)"
          />
        </li>
      </ul>
    </section>
    <footer class="footer" v-show="totalCount">
      <span class="todo-count">
        <strong style="color:#F56C6C;">{{remainingCount}}</strong>
        {{remainingCount > 1 ? 'items': 'item'}} left
      </span>
      <ul class="filters">
        <li>
          <a :class="{selected:type==='all'}" href="#/all">All</a>
        </li>
        <li>
          <a :class="{selected:type==='active'}" href="#/active">Active</a>
        </li>
        <li>
          <a :class="{selected:type==='completed'}" href="#/completed">Completed</a>
        </li>
      </ul>
      <!-- 总个数大于未完成个数 => 有已完成的 -->
      <button
        class="clear-completed"
        @click="removeCompleted"
        v-show="totalCount > remainingCount"
      >Clear completed</button>
    </footer>
  </section>
  <footer class="info">
    <p>Double-click to edit a todo</p>
    <!-- Remove the below line ↓ -->
    <p>
      Template by
      <a href="http://sindresorhus.com">Sindre Sorhus</a>
    </p>
    <!-- Change this out with your name and url ↓ -->
    <p>
      Created by
      <a href="https://github.com/leslie1943">Leslie</a>
    </p>
    <p>
      Part of
      <a href="http://todomvc.com">TodoMVC</a>
    </p>
  </footer>
</template>

<script>
import HelloWorld from './components/HelloWorld'
import TempateRef from './components/TempateRef'
import './assets/index.css'
import useLocalStorage from './utils/useLocalStorage.js'
import { computed, onMounted, onUnmounted, reactive, ref, watchEffect } from 'vue'
const storage = useLocalStorage()

// 1: 添加代办
const useAdd = todos => {
  // 创建响应式数据
  const todoInput = ref('')
  const addTodo = () => {
    const text = todoInput.value && todoInput.value.trim()
    if (text.length === 0) return
    // todos作为参数传进来的,而todos是被ref定义的响应式数据,所以使用todos.value操作数组
    todos.value.unshift({ text: text, completed: false })
    todoInput.value = ''
  }
  return {
    todoInput,
    addTodo
  }
}

// 2:删除代办事项
/**
 * 封装一个 compositionAPI 的时候,外层的参数在主函数的setup中就传入了
 * 内部的方法实现可以直接使用,而不需要再次传递
 */
const useRemove = todos => {
  const remove = todo => {
    const index = todos.value.indexOf(todo)
    todos.value.splice(index, 1) // 删除一项
  }
  const removeCompleted = () => {
    todos.value = todos.value.filter(item => !item.completed)
  }
  return {
    remove,
    removeCompleted
  }
}

// 3: 编辑代办项: 将remove方法传递过来使用
const useEdit = (remove) => {
  let beforeEditingText = ''
  const currentEdit = ref(null)

  const editTodo = todo => {
    // 当前编辑项的文本
    beforeEditingText = todo.text
    // 当前编辑的记录
    currentEdit.value = todo
  }

  const doneEdit = (todo) => {
    if (!currentEdit.value) return
    todo.text = todo.text.trim()
    todo.text || remove(todo)
    currentEdit.value = null
  }

  const cancelEdit = todo => {
    currentEdit.value = null
    todo.text = beforeEditingText
  }

  return {
    currentEdit,
    editTodo,
    doneEdit,
    cancelEdit
  }
}

// 4: 切换代办项完成状态
const useFilter = todos => {
  // 是否全部完成
  const allDone = computed({
    get() {
      return todos.value.every(item => item.completed)
    },
    set(val) {
      todos.value.forEach(todo => {
        todo.completed = val
      })
    }
  })
  // 状态对应的方法
  const filter = {
    all: list => list,
    active: list => list.filter(todo => !todo.completed),
    completed: list => list.filter(todo => todo.completed)
  }
  // 当前类型
  const type = ref('all')
  const matchedTotos = computed(() => filter[type.value](todos.value))

  // 根据hash值设置当前类型
  const onHashChanged = () => {
    const hash = window.location.hash.replace('#/', '')
    if (filter[hash]) {
      type.value = hash
    } else {
      type.value = 'all'
      window.location.hash = ''
    }
  }

  onMounted(() => {
    window.addEventListener('hashchange', onHashChanged)
    onHashChanged()
  })

  onUnmounted(() => {
    window.removeEventListener('hashchange', onHashChanged)
  })

  // 计算未完成的个数
  // const remainingCount = computed(() => todos.value.length - todos.value.filter(item => item.completed).length)
  const remainingCount = computed(() => filter.active(todos.value).length)

  // 总个数
  const totalCount = computed(() => todos.value.length)

  return {
    allDone,
    matchedTotos,
    remainingCount,
    totalCount,
    type
  }
}

// 5: 存储代办事项
const useStorage = () => {
  const KEY = 'TODOKEYS'
  const todos = ref(storage.getItem(KEY) || [])

  watchEffect(() => {
    storage.setItem(KEY, todos.value)
  })
  return todos
}

// 主函数
export default {
  name: 'App',
  components: {
    HelloWorld, TempateRef
  },
  setup() {
    // console.info('setup props', props)
    // console.info('setup context', context)
    // 响应式
    const obj = reactive({ msg: 'Hello V3', color: '#409FEE', size: '16px' })
    const title = ref('')
    title.value = 'title in parent'
    const todos = useStorage()
    // 解构remove, 为edit传参
    const { remove, removeCompleted } = useRemove(todos)

    return {
      todos,
      remove,
      obj,
      title,
      removeCompleted,
      ...useAdd(todos),
      ...useEdit(remove),
      ...useFilter(todos)
    }
  },
  // 自定义指令
  directives: {
    editingFocus: (el, binding) => {
      binding.value && el.focus()
    }
  }
}
</script>

<style>
</style>
