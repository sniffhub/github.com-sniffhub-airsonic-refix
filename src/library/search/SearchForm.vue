<template>
  <form class="search-terminal" @submit.prevent="search">
    <span class="search-terminal-prompt">&gt;</span>
    <input
      v-model="query"
      class="form-control" type="search"
      placeholder="search...">
    <span class="search-terminal-cursor" />
  </form>
</template>
<script lang="ts">
  import { defineComponent } from 'vue'

  export default defineComponent({
    data() {
      return {
        query: ''
      }
    },
    methods: {
      search(): void {
        this.$router.push({ name: 'search', query: { query: this.query } })
      }
    }
  })
</script>
