<template>
  <v-app>
    <v-main class="main">
      <Player/>
    </v-main>
  </v-app>
</template>

<script>
import Player from './components/Player';

export default {
  name: 'App',

  components: {
    Player,
  },

  data: () => ({
    //
  }),
};
</script>

<style scoped>
.main{
  background-color: #EBA6A9;
}
</style>
