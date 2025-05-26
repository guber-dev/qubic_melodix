<template>
  <div id="app" class="unselectable">
    <PageBackground
      v-if="
        $store.state.audio &&
        $route.meta.requireBg &&
        showOnPageRequireSignin &&
        !maintenanceMsg
      "
    ></PageBackground>
    <ModalGlobal ref="gm"></ModalGlobal>
    <FloatingAlert ref="alert"></FloatingAlert>
    <!-- <QubicWallet class="qubic-wallet-container" /> -->
    <transition name="fade" v-if="$store.state.audio">
      <div class="center" v-if="maintenanceMsg">
        <h1>{{ maintenanceMsg.title }}</h1>
        <div v-html="maintenanceMsg.body"></div>
        <img src="/assets/melodix_logo.png" class="maintenance_logo" />
        <div
          v-if="maintenanceMsg.showUpdateButton"
          class="btn-action btn-dark"
          @click="reload"
        >
          Update
        </div>
        <div style="opacity: 0.2">
          {{ maintenanceMsg.currentVersion }} - {{ maintenanceMsg.build }}
        </div>
      </div>
      <keep-alive
        :include="['SongSelect', 'MyStudio']"
        v-else-if="showOnPageRequireSignin && !$store.state.redirecting"
      >
        <router-view class="routerView" :key="$route.path" />
      </keep-alive>
      <div v-else>
        <div class="center blink_me">
          <img src="/assets/melodix_logo.png" class="loading_logo" />
          <div>Logging you in...</div>
        </div>
      </div>
    </transition>
  </div>
</template>

<script>
import Audio from "./javascript/audio.js";
import ModalGlobal from "./components/ui/ModalGlobal.vue";
import FloatingAlert from "./components/ui/FloatingAlert.vue";
import PageBackground from "./components/common/PageBackground.vue";
import { logEvent } from "./helpers/analytics";
// semver убран так как проверка версий отключена
import "vue-awesome/icons/volume-up";
import "vue-awesome/icons/volume-mute";
import "vue-awesome/icons/expand";
import "vue-awesome/icons/compress";
import "vue-awesome/icons/plus";
import "vue-awesome/icons/redo";
import "vue-awesome/icons/cog";
import "vue-awesome/icons/sign-out-alt";
import "vue-awesome/icons/play";
import "vue-awesome/icons/pause";
import "vue-awesome/icons/arrow-right";
import QubicWallet from "./components/wallet/QubicWallet.vue";

export default {
  name: "App",
  components: {
    ModalGlobal,
    FloatingAlert,
    PageBackground,
    QubicWallet,
  },
  mounted() {
    this.$store.commit("setAudio", new Audio());
    this.$store.commit("setGlobalModal", this.$refs.gm);
    this.$store.commit("setFloatingAlert", this.$refs.alert);
    this.listenToUpdates();
    window.addEventListener("online", this.updateOnlineStatus);
    window.addEventListener("offline", this.updateOnlineStatus);
    this.updateOnlineStatus();
  },
  beforeDestroy() {
    window.removeEventListener("online", this.updateOnlineStatus);
    window.removeEventListener("offline", this.updateOnlineStatus);
  },
  data() {
    return {
      refreshing: false,
      registration: null,
      maintenanceMsg: null,
    };
  },
  methods: {
    updateOnlineStatus(e) {
      if (!e?.type) return;
      const isOnline = e.type === "online" || window.navigator.onLine;
      if (isOnline) this.$store.state.alert.success("You are back online!");
      else this.$store.state.alert.error("No internet connection");
      logEvent("online_status_changed", isOnline, "system");
    },
    listenToUpdates() {
      document.addEventListener("swUpdated", this.updateAvailable, {
        once: true,
      });

      navigator.serviceWorker.addEventListener("controllerchange", () => {
        if (this.refreshing) return;
        this.refreshing = true;
        console.log("Game is updating...");
        logEvent("game_updated", null, "system");
        window.location.reload();
      });
    },
    reload() {
      window.location.reload(true);
      logEvent("game_force_updated", null, "system");
    },
    // Store the SW registration so we can send it a message
    // We use `updateExists` to control whatever alert, toast, dialog, etc we want to use
    // To alert the user there is an update they need to refresh for
    async updateAvailable(event) {
      logEvent("update_available", null, "system");
      this.registration = event.detail;
      this.$store.state.alert
        .info("A new version of the game is found!", 0, "Update Now")
        .then(() => {
          Logger.log("update accepted");
          logEvent("update_accepted", null, "system");
          // Make sure we only send a 'skip waiting' message if the SW is waiting
          if (!this.registration || !this.registration.waiting) return;
          // send message to SW to skip the waiting and activate the new SW
          this.registration.waiting.postMessage({ type: "SKIP_WAITING" });
        });
    },
  },
  computed: {
    showOnPageRequireSignin() {
      return (
        !this.$route.meta.requireSignin ||
        (this.$store.state.initialized && this.$route.meta.requireSignin)
      );
    },
  },
  watch: {
    // Отключили проверку версий с серверов Rhythm Plus для независимости
    // async "$store.state.remoteConfig"(config) { ... },
    $route(to) {
      const pageTitle = to.meta.title ? to.meta.title + " - " : "";
      document.title = pageTitle + "Melodix - Rhythm Game";
    },
  },
};
</script>

<style>
.fade-enter-active,
.fade-leave-active {
  transition: 0.5s;
}
.fade-enter, .fade-leave-to /* .fade-leave-active below version 2.1.8 */ {
  opacity: 0;
}

.routerView {
  position: fixed;
  width: 100%;
  height: 100%;
  top: 0;
  left: 0;
  overflow-y: scroll;
}

.maintenance_logo {
  margin-top: 50px;
  max-width: 150px;
  opacity: 0.5;
}

.center a {
  color: rgb(127, 255, 255);
  text-decoration: underline;
}

.qubic-wallet-container {
  position: fixed;
  top: 20px;
  right: 20px;
  z-index: 1000;
}
</style>
