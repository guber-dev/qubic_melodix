<template>
  <div>
    <div class="center_logo" v-if="$store.state">
      <img
        class="logo"
        src="assets/melodix_logo.png"
      />
      <StartButton text="Start Game" @click="goToGame"></StartButton>
    </div>
  </div>
</template>

<script>
import StartButton from "../components/ui/StartButton.vue";

export default {
  name: "Home",
  components: {
    StartButton,
  },
  data() {
    return {};
  },
  computed: {},
  watch: {},
  mounted() {},
  methods: {
    goToGame() {
      this.$router.push("/menu");
    },
  },
};
</script>

<style scoped>
.leftBottom {
  position: fixed;
  left: 5px;
  bottom: 5px;
  opacity: 0.3;
}

@media only screen and (max-width: 1000px) {
  .leftBottom {
    display: none;
  }
}
</style>
