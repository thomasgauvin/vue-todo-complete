<template>
  <div class="todo-list">
    <h1>Welcome, {{ currentUser ? currentUser.userDetails : 'Guest' }}</h1>
    <input type="text" v-model="newTodo" @keyup.enter="addTodo" class="rounded-input">
    <button @click="addTodo" class="rounded-button">Add Todo</button>
    <ul>
      <li v-for="(todo, index) in todos" :key="index" class="rounded-todo">
        <input type="checkbox" v-model="todo.complete" @change="toggleCompleted(index)">
        <span v-if="!todo.editing" :class="{ 'completed': todo.complete }">{{ todo.description }}</span>
        <label>{{ todo.label }}</label>
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
          if (this.currentUser && this.currentUser.userRoles.includes('can_save')) {
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
      //call list function
      const todos = await this.list();
      this.todos = todos;

    },
    async addTodo() {
      if (this.newTodo.trim() !== '') {
        const todo = {
          complete: false,
          label: this.newTodo,
          id: Math.random().toString(36).substr(2, 9),
        };
        console.log(this.currentUser);
        if (this.currentUser && this.currentUser.userRoles.includes('can_save')) {
          todo.userId = this.currentUser.userId;
          await this.saveTodo(todo);
        } else {
          this.todos.push(todo);
        }
        this.newTodo = '';
      }
    },
    async saveTodo(todo) {
      //make call to create function
      const newTodo = await this.create(todo);
      this.todos.push(newTodo);
    },
    removeTodo(index) {
      const todo = this.todos[index];
      if (this.currentUser && this.currentUser.userRoles.includes('can_save')) {
        this.deleteTodoFromServer(todo);
      }
      this.todos.splice(index, 1);
    },
    async deleteTodoFromServer(todo) {
      //make call to delete todo
      await this.delete(todo.id);
    },
    toggleCompleted(index) {
      if (this.currentUser && this.currentUser.userRoles.includes('can_save')) {
        this.updateTodoOnServer(this.todos[index]);
      }
    },
    async updateTodoOnServer(todo) {
      //make call to update todo
      console.log(todo);
      const updatedTodo = await this.update(todo.id, todo);
      console.log(updatedTodo);
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
    },
    async list() {
      const query = `
        query GetTodoItems {
          todoItems {
            items {
              id
              userId
              label
              complete
            }
          }
        }`;
          
      const endpoint = "/data-api/graphql";
      const response = await fetch(endpoint, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ query: query })
      });
      
      const result = await response.json();
      return result.data.todoItems.items;
    },
    async get(id) {
      const gql = `
        query GetTodoItem($id: ID!) {
          todoItems_by_pk(id: $id) {
            id
            userId
            label
            complete
          }
        }`;

      const query = {
        query: gql,
        variables: {
          id: id,
        },
      };

      const endpoint = "/data-api/graphql";
      const response = await fetch(endpoint, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(query),
      });

      const result = await response.json();
      console.table(result.data.todoItems_by_pk);
    },
    async update(id, newData) {
      const gql = `
        mutation UpdateTodoItem($id: ID!, $_partitionKeyValue: String!, $item: UpdateTodoItemsInput!) {
          updateTodoItems(id: $id, _partitionKeyValue: $_partitionKeyValue, item: $item) {
            id
            userId
            label
            complete
          }
        }`;

      const query = {
        query: gql,
        variables: {
          id: id,
          _partitionKeyValue: id,
          item: newData
        }
      };

      const endpoint = "/data-api/graphql";
      const response = await fetch(endpoint, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(query)
      });

      const result = await response.json();
      return result.data.updateTodoItems;
    },
    async create(data) {
      const gql = `
        mutation CreateTodoItem($item: CreateTodoItemsInput!) {
          createTodoItems(item: $item) {
            id
            userId
            label
            complete
          }
        }`;

      const query = {
        query: gql,
        variables: {
          item: data
        }
      };

      const endpoint = "/data-api/graphql";
      const response = await fetch(endpoint, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(query)
      });

      const result = await response.json();
      return result.data.createTodoItems;
    },
    async delete(id) {
      const gql = `
        mutation DeleteTodoItem($id: ID!, $_partitionKeyValue: String!) {
          deleteTodoItems(id: $id, _partitionKeyValue: $_partitionKeyValue) {
            id
          }
        }`;

      const query = {
        query: gql,
        variables: {
          id: id,
          _partitionKeyValue: id
        }
      };

      const endpoint = "/data-api/graphql";
      const response = await fetch(endpoint, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(query)
      });

      const result = await response.json();
      return result.data.deleteTodoItems;
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
