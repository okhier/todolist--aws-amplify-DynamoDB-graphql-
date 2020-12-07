<template>
  <div class="todo-main">
    <h1>TodoApp</h1>
    <v-text-field v-model="name" label="Name"></v-text-field>
    <v-text-field v-model="description" label="やること"></v-text-field>
    <v-btn @click="createTodo">Create</v-btn>
    <ul class="todo-list-wrapper">
      <li
        v-for="todo in todos"
        :key="todo.id"
        class="todo-list"
        @click="handleCheckedTogle(todo.id, todo.checked)"
      >
        {{ todo.name }} : {{ todo.description }}
        <v-icon v-if="todo.checked === true">mdi-check-bold</v-icon>
        <v-icon class="todo-list-delete" @click="handleDelete(todo.id, $event)">
          mdi-window-close</v-icon
        >
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
    async handleCheckedTogle(todoId, todoChecked) {
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
            return // remove duplications
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
          this.todos = this.getTodos()
        },
      })
    },
  },
}
</script>
