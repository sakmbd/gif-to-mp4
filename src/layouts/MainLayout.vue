<template>
  <q-layout view="lHh Lpr lFf">
    <q-header elevated class="bg-primary">
      <q-toolbar>
        <q-icon name="fa-solid fa-film" size="28px" class="q-mr-sm" color="white" />
        <q-toolbar-title class="text-weight-bold">
          <span class="text-white">GIF</span>
          <span class="text-yellow"> to </span>
          <span class="text-white">MP4</span>
        </q-toolbar-title>
      </q-toolbar>
    </q-header>

    <q-page-container>
      <router-view />
    </q-page-container>
  </q-layout>
</template>
