<template>
  <q-layout view="lHh Lpr lFf">
    <q-header elevated>
      <q-toolbar>
        <q-toolbar-title>
          News Crawler
        </q-toolbar-title>

        <q-tabs v-model="tab" shrink>
          <q-route-tab name="index" label="Home Page" to="/" exact />
          <q-route-tab name="storage" label="Storage" to="/storage" tag="a" target="_blank" exact />
        </q-tabs>
        <div>v0.1.20 Alpha</div>
      </q-toolbar>
    </q-header>

    <q-page-container>
      <router-view />
    </q-page-container>
  </q-layout>
</template>

<script>

export default {
  name: 'MainLayout',
  components: { },
  data () {
    return {
      tab: 'index'
    }
  }
}
</script>
