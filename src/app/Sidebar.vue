<template>
  <div class="sidebar-container elevated">
    <div class="sidebar-fixed d-none d-md-block">
      <div class="sidebar-scroll">
        <SidebarNav />
      </div>
      <RadarSweep class="sidebar-radar" />
    </div>
    <div
      class="offcanvas offcanvas-start d-md-none"
      tabindex="-1"
      :class="store.menuVisible ? 'showing': 'hiding'"
      @click="store.hideMenu"
    >
      <div class="offcanvas-body p-0">
        <SidebarNav />
      </div>
    </div>
  </div>
</template>
<script lang="ts">
  import { defineComponent } from 'vue'
  import SidebarNav from './SidebarNav.vue'
  import RadarSweep from './RadarSweep.vue'
  import { useMainStore } from '@/shared/store'

  export default defineComponent({
    components: {
      SidebarNav,
      RadarSweep,
    },
    setup() {
      return {
        store: useMainStore(),
      }
    },
  })
</script>
<style>
  .sidebar-container .nav {
    padding-left: 0.5rem;
    padding-right: 0.5rem;
  }

  .sidebar-container .sidebar-fixed {
    width: 250px;
    position: sticky;
    top: 0;
    height: 100vh;
    display: flex;
    flex-direction: column;
  }

  .sidebar-container .sidebar-scroll {
    flex: 1 1 auto;
    min-height: 0;
    overflow-y: scroll;
    scrollbar-width: none;
    padding-bottom: 1rem;
  }

  .sidebar-container .sidebar-scroll::-webkit-scrollbar {
    display: none;
  }

  .sidebar-container .sidebar-radar {
    flex: 0 0 auto;
  }

  .sidebar-container .sidebar-brand {
    padding: 1rem 1rem 0.75rem;
  }

  .sidebar-container .sidebar-heading {
    margin-top: 1.25em;
    padding: 0.5rem 1rem;
    font-weight: bold;
    text-transform: uppercase;
    display: block;
  }

  .sidebar-container .nav-link {
    width: calc(100%);
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    border: 1px solid var(--term-amber-dim, #cc5500);
    margin-bottom: 4px;
    padding-top: 0.4rem;
    padding-bottom: 0.4rem;
  }

  .sidebar-container a.nav-link .icon {
    margin-right: 0.75rem;
  }

  .sidebar-container .nav-link:not(.router-link-active) .icon {
    color: var(--bs-secondary-color);
  }
  .sidebar-container .nav-link:hover {
    color: inherit;
    background-color: rgba(255, 122, 26, 0.08);
    border-color: var(--term-amber, #ff7a1a);
  }
  .sidebar-container .nav-link.router-link-active {
    color: var(--term-amber, #ff7a1a);
    background-color: rgba(255, 122, 26, 0.12);
    border-color: var(--term-amber, #ff7a1a);
  }
  .sidebar-container .nav-link.router-link-active:hover {
    color: var(--term-amber, #ff7a1a);
  }
</style>
