<template>
  <div id="app">
    <v-app>
      <v-app-bar app dark color="primary" flat>
        <v-toolbar-title>Voting App</v-toolbar-title>
      </v-app-bar>
      <v-main>
        <v-container fluid>
          <v-row justify="center">
            <v-col md="4">
              <canditates-list></canditates-list>
            </v-col>
          </v-row>
        </v-container>
      </v-main>
    </v-app>
  </div>
</template>

<script>

import CanditatesList from './components/CanditatesList.vue';

export default {
  name: 'App',
  components: {
    CanditatesList
  }
}
</script>
