<template>
  <div class="todo-list">
    <h1>Welcome, {{ currentUser ? currentUser.userDetails : 'Guest' }}</h1>
    <input type="text" v-model="newTodo" @keyup.enter="addTodo" class="rounded-input">
    <button @click="addTodo" class="rounded-button">Add Todo</button>
    <ul>
      <li v-for="(todo, index) in todos" :key="index" class="rounded-todo">
        <input type="checkbox" v-model="todo.completed" @change="toggleCompleted(index)">
        <span v-if="!todo.editing" :class="{ 'completed': todo.completed }">{{ todo.description }}</span>
        <label>{{ todo.title }}</label>
        <button @click="removeTodo(index)">Remove</button>
      </li>
    </ul>
  </div>
</template>

<script>
export default {
  data() {
    return {
      newTodo: '',
      todos: [],
      currentUser: null
    };
  },
  created() {
    this.getCurrentUser();
  },
  methods: {
    async getCurrentUser() {
      try {
        const response = await fetch('/.auth/me');
        if (response.ok) {
          const data = await response.json();
          this.currentUser = data.clientPrincipal;
          if (this.currentUser && this.currentUser.roles.includes('can_save')) {
            await this.loadTodos();
          }
        } else {
          console.error('Failed to fetch user information');
        }
      } catch (error) {
        console.error('Error fetching user information:', error);
      }
    },
    async loadTodos() {
      try {
        const response = await fetch('/api/todos', {
          headers: {
            Authorization: `Bearer ${this.currentUser.identityProviderAccessToken}`
          }
        });
        if (response.ok) {
          const data = await response.json();
          this.todos = data.todos;
        } else {
          console.error('Failed to load todos');
        }
      } catch (error) {
        console.error('Error loading todos:', error);
      }
    },
    async addTodo() {
      if (this.newTodo.trim() !== '') {
        const todo = {
          description: this.newTodo,
          completed: false
        };
        if (this.currentUser && this.currentUser.roles.includes('can_save')) {
          todo.userId = this.currentUser.userId;
          await this.saveTodo(todo);
        } else {
          this.todos.push(todo);
        }
        this.newTodo = '';
      }
    },
    async saveTodo(todo) {
      try {
        const response = await fetch('/api/todos', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            Authorization: `Bearer ${this.currentUser.identityProviderAccessToken}`
          },
          body: JSON.stringify(todo)
        });
        if (!response.ok) {
          console.error('Failed to save todo');
        }
      } catch (error) {
        console.error('Error saving todo:', error);
      }
    },
    removeTodo(index) {
      const todo = this.todos[index];
      if (todo._id) {
        this.deleteTodoFromServer(todo);
      }
      this.todos.splice(index, 1);
    },
    async deleteTodoFromServer(todo) {
      try {
        const response = await fetch(`/api/todos/${todo._id}`, {
          method: 'DELETE',
          headers: {
            Authorization: `Bearer ${this.currentUser.identityProviderAccessToken}`
          }
        });
        if (!response.ok) {
          console.error('Failed to delete todo from server');
        }
      } catch (error) {
        console.error('Error deleting todo from server:', error);
      }
    },
    toggleCompleted(index) {
      if (this.todos[index]._id) {
        this.updateTodoOnServer(this.todos[index]);
      }
    },
    async updateTodoOnServer(todo) {
      try {
        const response = await fetch(`/api/todos/${todo._id}`, {
          method: 'PUT',
          headers: {
            'Content-Type': 'application/json',
            Authorization: `Bearer ${this.currentUser.identityProviderAccessToken}`
          },
          body: JSON.stringify({ completed: todo.completed })
        });
        if (!response.ok) {
          console.error('Failed to update todo on server');
        }
      } catch (error) {
        console.error('Error updating todo on server:', error);
      }
    },
    editTodo(index) {
      this.todos.forEach((todo, i) => {
        if (i === index) {
          todo.editing = true;
        } else {
          todo.editing = false;
        }
      });
    },
    async updateTodo(index) {
      if (this.todos[index].description.trim() === '') {
        this.removeTodo(index);
      } else {
        this.todos[index].editing = false;
        if (this.todos[index]._id) {
          this.updateTodoOnServer(this.todos[index]);
        }
      }
    }
  }
};
</script>

<style scoped>
.todo-list {
  margin: 20px auto;
  max-width: 400px;
  padding: 20px;
  background-color: #f5f5f5;
  border-radius: 10px;
}

.rounded-input,
.rounded-button,
.rounded-todo {
  border-radius: 5px;
  margin-bottom: 5px;
}

.rounded-input,
.rounded-button,
.edit-todo-input {
  border: none;
  padding: 5px 10px;
}

.rounded-todo:hover {
  transform: scale(1.15);
}

.completed {
  text-decoration: line-through;
  transition: text-decoration 0.3s ease-in-out;
}

.edit-todo-input {
  margin-bottom: 5px;
  width: 90%;
}

ul {
  list-style-type: none;
}
</style>
