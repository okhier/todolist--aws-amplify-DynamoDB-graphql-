<template>
  <div class="todo-main">
    <h1>TodoApp</h1>
    <v-text-field v-model="name" label="date-日付"></v-text-field>
    <v-text-field v-model="description" label="todo-やること"></v-text-field>
    <v-btn @click="createTodo">Create</v-btn>
    <ul class="todo-list-wrapper">
      <li
        v-for="todo in todos"
        :key="todo.id"
        class="todo-list"
        @click="handleCheckedToggle(todo.id, todo.checked)"
      >
        <span class="todo-list-item" :class="{ checked: todo.checked }"
          >{{ todo.name }} : {{ todo.description }}</span
        >
        <span class="todo-list-button">
          <v-icon v-if="todo.checked === true" class="todo-list-checker"
            >mdi-check-bold</v-icon
          >
          <v-icon
            class="todo-list-delete"
            @click="handleDelete(todo.id, $event)"
          >
            mdi-window-close</v-icon
          >
        </span>
      </li>
    </ul>
  </div>
</template>

<script>
import { API, graphqlOperation } from 'aws-amplify'
import { createTodo, deleteTodo, updateTodo } from '~/graphql/mutations'
import { listTodosByCreated } from '~/graphql/queries'
import {
  onCreateTodo,
  onDeleteTodo,
  onUpdateTodo,
} from '~/graphql/subscriptions'

export default {
  data() {
    return {
      name: '',
      id: '',
      description: '',
      todos: [],
      type: 'set',
      checked: false,
    }
  },
  created() {
    this.getTodos()
    this.subscribe()
  },
  methods: {
    async handleDelete(id, e) {
      e.stopPropagation()
      this.id = id
      await API.graphql({
        query: deleteTodo,
        variables: { input: { id } },
      })
    },
    async handleCheckedToggle(todoId, todoChecked) {
      this.id = todoId
      await API.graphql(
        graphqlOperation(updateTodo, {
          input: { id: todoId, checked: !todoChecked },
        })
      )
    },
    async createTodo() {
      const { name, description, type, checked } = this
      if (!name || !description) return false
      const todo = { name, description, type, checked }
      if (
        this.todos.some(
          (item) =>
            item.name === todo.name && item.description === todo.description
        )
      ) {
        alert('既に同じ内容が登録されております。')
        this.name = ''
        this.description = ''
        return
      }
      await API.graphql({
        query: createTodo,
        variables: { input: todo },
      })
      this.name = ''
      this.description = ''
    },
    async getTodos() {
      const todos = await API.graphql(
        graphqlOperation(listTodosByCreated, { type: 'set' })
      )
      this.todos = todos.data.listTodosByCreated.items
    },
    subscribe() {
      API.graphql({ query: onCreateTodo }).subscribe({
        next: (eventData) => {
          const todo = eventData.value.data.onCreateTodo
          if (
            this.todos.some(
              (item) =>
                item.name === todo.name && item.description === todo.description
            )
          )
            return
          this.todos = [...this.todos, todo]
        },
      })
      API.graphql({ query: onDeleteTodo }).subscribe({
        next: (eventData) => {
          const deletedIndex = this.todos.findIndex(
            (todo) => todo.id === this.id
          )
          this.todos.splice(deletedIndex, 1)
        },
      })
      API.graphql({ query: onUpdateTodo }).subscribe({
        next: (eventData) => {
          const toggledIndex = this.todos.findIndex(
            (todo) => todo.id === this.id
          )
          this.todos[toggledIndex].checked = !this.todos[toggledIndex].checked
        },
      })
    },
  },
}
</script>
