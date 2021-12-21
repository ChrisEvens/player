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
@import url('https://fonts.googleapis.com/css2?family=Rock+3D&display=swap');
*{
  font-family: 'Rock 3D', cursive;
  background-color: #EBA6A9;
}
</style>
