<template>
  <ChatView />
</template>

<script>
import ChatView from './components/ChatView.vue'

export default {
  name: 'App',
  components: {
    ChatView
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
}
</style>
