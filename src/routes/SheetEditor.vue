<template>
  <div>
    <!-- visualizer canvas -->
    <Visualizer
      ref="visualizer"
      :setBlur="blur"
      v-show="srcMode === 'url'"
    ></Visualizer>

    <div class="toolbar blurBackground" style="padding-left: 0;">
      <div class="logo">
        <img
          src="/assets/editor_logo.png"
          style="height: 8vh; cursor: pointer; pointer-events: all;"
          @click="goToMenu"
        />
        <div>Sheet Editor (Alpha)</div>
      </div>
      <div style="flex-grow: 1;"></div>
      <a href="#" @click.prevent="newEditor">New</a>
      <a
        href="#"
        :class="{ disabled: isDisabled }"
        @click.prevent="
          restartGame();
          songLoaded();
        "
        >Restart</a
      >
      <div style="display: flex;" :class="{ disabled: isDisabled }">
        <a
          href="#"
          @click.prevent="saveSheet"
          :class="{ disabled: !isSheetOwner }"
          style="position: relative;"
        >
          <span v-if="sheetChanged" class="saveIndicator">●</span>Save
        </a>
        <a href="#" @click.prevent="togglePlayMode(false)">
          {{ playMode ? "Edit" : "Test" }}
        </a>
        <a
          href="#"
          @click.prevent="showPublishModal"
          :class="{ disabled: !isSheetOwner }"
          >Publish</a
        >
        <a href="#" @click.prevent="exportChart" style="margin-left: 10px;">Export chart.json</a>
        <a href="#" @click.prevent="playChart" style="margin-left: 10px;">Play Chart</a>
      </div>
    </div>

    <div class="scrollBar">
      <div class="main hideOverflow">
        <div class="column side left blurBackground">
          <div class="tabs" :class="{ disabled: isDisabled }">
            <div
              class="tab"
              :class="{ active: leftTab === 1 }"
              @click="leftTab = 1"
            >
              Info
            </div>
            <div
              class="tab"
              :class="{ active: leftTab === 2 }"
              @click="leftTab = 2"
            >
              Options
            </div>
          </div>
          <!-- tab 1 - info editor -->
          <info-editor
            style="flex-grow: 1;"
            ref="info"
            v-show="leftTab === 1"
          ></info-editor>
          <!-- tab 2 - options -->
          <div v-if="leftTab === 2" style="flex-grow: 1; overflow: scroll;">
            <PlayControl
              style="padding: 0 30px; box-sizing: border-box;"
              ref="control"
              :playData="$data"
              :formStyle="true"
            ></PlayControl>
          </div>
          <!-- song control -->
          <SongListItem
            v-if="songInfo.id"
            :song="songInfo"
            :hideBg="true"
            @selected="updateSongDetail"
            style="cursor: pointer;"
          >
            <div style="padding-right: 20px; opacity: 0.5;">Edit</div>
          </SongListItem>

        </div>

        <div class="column middle" :class="{ disabled: !initialized }">
          <!-- game wrapper -->
          <div class="gameWrapper" ref="wrapper">
            <canvas ref="effectCanvas" id="effectCanvas"></canvas>
            <canvas
              ref="mainCanvas"
              id="gameCanvas"
              :class="{ perspective }"
            ></canvas>
          </div>
          <!-- mark indicator -->
          <MarkComboJudge
            style="z-index: 500;"
            ref="judgeDisplay"
            v-show="playMode && result.combo > 0"
          ></MarkComboJudge>

          <!-- center text -->
          <ZoomText style="z-index: 1000;" ref="zoom"></ZoomText>
        </div>

        <div
          class="column side right blurBackground"
          v-if="instance"
          :class="{ disabled: isDisabled }"
        >
          <div
            v-if="!disableMappingTable"
            @click="disableMappingTable = true"
            style="
              float: right;
              padding-top: 30px;
              opacity: 0.5;
              cursor: pointer;
            "
          >
            Disable
          </div>
          <h2>Mappings</h2>
          <SheetTable v-if="!disableMappingTable"></SheetTable>
          <div v-else>
            To improve editor performance, mapping table is disabled.
            <div
              class="btn-action btn-dark"
              style="margin: 10px 0; width: 100px;"
              @click="disableMappingTable = false"
            >
              Enable
            </div>
          </div>
        </div>
      </div>
    </div>

    <div
      class="toolbar blurBackground"
      v-if="instance"
      :class="{ disabled: isDisabled }"
    >
      <div style="font-size: 30px; width: 80px; text-align: center;">
        {{ currentTime }}
      </div>
      <div class="action_buttons">
        <v-icon
          class="vicon"
          name="undo"
          scale="1"
          @click="seekTo(Number(currentTime) - instance.noteDelay)"
        />
        <v-icon
          class="vicon"
          name="play"
          scale="1.5"
          @click="songLoaded"
          v-if="instance && instance.paused"
        />
        <v-icon
          class="vicon"
          name="pause"
          scale="1.5"
          @click="pauseGame"
          v-else
        />
        <v-icon
          class="vicon"
          name="redo"
          scale="1"
          @click="seekTo(Number(currentTime) + instance.noteDelay)"
        />
      </div>
      <div style="flex-grow: 1;">
        <vue-slider
          :tooltip-placement="'right'"
          :interval="0.001"
          :min="sliderMinLength"
          :max="sliderMaxLength"
          :value="currentTime"
          :contained="true"
          :lazy="true"
          @dragging="seeking"
          @change="seekTo"
        ></vue-slider>
      </div>
      <div style="width: 90px; margin-left: 30px;">
        <select id="songSelect" v-model="playbackSpeed">
          <option disabled>Playback Speed</option>
          <option value="0.25">0.25X</option>
          <option value="0.5">0.5X</option>
          <option value="0.75">0.75X</option>
          <option value="1">1.0X</option>
          <option value="1.5">1.5X</option>
        </select>
      </div>
    </div>

    <Loading style="z-index: 500;" :show="loading">Just a second...</Loading>
    <Loading style="z-index: 1000;" :show="!$store.state.initialized"
      >Communicating...</Loading
    >

    <Modal
      ref="publishModal"
      :hideFooter="true"
      titleText="Publish"
      style="text-align: center; z-index: 1000;"
    >
      <template>
        <publish></publish>
      </template>
    </Modal>
  </div>
</template>

<script>
import { getSong, getSheet, getGameSheet, updateSheet } from "../javascript/db";
import Visualizer from "../components/common/Visualizer.vue";
import InfoEditor from "../components/editor/InfoEditor.vue";
import SheetTable from "../components/editor/SheetTable.vue";
import SongListItem from "../components/menus/SongListItem.vue";
import PlayControl from "../components/common/PlayControl.vue";
import ZoomText from "../components/game/ZoomText.vue";
import MarkComboJudge from "../components/game/MarkComboJudge.vue";
import Modal from "../components/ui/Modal.vue";
import Publish from "../components/editor/Publish.vue";
import Loading from "../components/ui/Loading.vue";
import GameMixin from "../mixins/gameMixin";
import VueSlider from "vue-slider-component";
import { tween } from "shifty";
import "vue-slider-component/theme/antd.css";
import "vue-awesome/icons/redo";
import "vue-awesome/icons/undo";

// Импортируем ChartManager
class ChartManager {
  static getAllCharts() {
    try {
      const charts = localStorage.getItem('melodixCharts');
      return charts ? JSON.parse(charts) : {};
    } catch (error) {
      console.error('[ChartManager] Ошибка загрузки чартов:', error);
      return {};
    }
  }

  static saveChart(chartId, chartData) {
    try {
      const charts = this.getAllCharts();
      charts[chartId] = {
        ...chartData,
        dateUpdated: { seconds: Date.now() / 1000 },
        id: chartId
      };
      localStorage.setItem('melodixCharts', JSON.stringify(charts));
      console.log(`[ChartManager] Чарт ${chartId} сохранен`);
      return true;
    } catch (error) {
      console.error('[ChartManager] Ошибка сохранения чарта:', error);
      return false;
    }
  }
}

export default {
  name: "SheetEditor",
  components: {
    Visualizer,
    InfoEditor,
    VueSlider,
    SheetTable,
    SongListItem,
    Loading,
    Modal,
    Publish,
    PlayControl,
    ZoomText,
    MarkComboJudge,
  },
  mixins: [GameMixin],
  data() {
    return {
      wrapper: null,
      contentHeight: "86vh",
      playMode: false,
      songLength: 0,
      inEditor: true,
      disabled: false,
      songInfo: {
        id: null,
        title: '',
        artist: '',
        url: '',
      },
      sheetInfo: {
        id: null,
        title: '',
        keys: 4,
      },
      gameSheetInfo: null,
      loading: false,
      showExistingNote: true,
      selectedNotes: [],
      leftTab: 1,
      disableMappingTable: false,
      options: {
        soundEffect: true,
        lowerHitLine: true,
      },
      sheetChanged: false,
      srcMode: "url",
    };
  },
  computed: {
    currentTime() {
      return (this.instance.playTime - this.instance.noteDelay).toFixed(2);
    },
    isSheetOwner() {
      return true;
    },
    isSongOwner() {
      return true;
    },
    sliderMinLength() {
      return (
        (this.currentSong?.startAt ?? 0) - this.instance.noteDelay.toFixed(3)
      );
    },
    sliderMaxLength() {
      return this.currentSong?.endAt ?? this.songLength;
    },
    isDisabled() {
      return !this.initialized || this.disabled;
    },
  },
  watch: {
    playbackSpeed() {
      this.setPlaybackRate(this.playbackSpeed);
    },
    "options.lowerHitLine"() {
      this.instance.reposition();
    },
    "$store.state.initialized"() {
      this.checkLoggedIn();
    },
    "instance.timeArr"() {
      if (!this.sheetChanged && this.initialized) this.sheetChanged = true;
    },
    "$route.query.song"() {
      if (!this.$route.params.sheet) {
        this.checkSongQuery();
      }
    },
    songInfo: {
      handler(val) {
        console.log('[SheetEditor] watcher songInfo:', val);
        if (val && val.id && val.url) { // Проверяем что есть и id и url
          this.initialized = true;
          console.log('[SheetEditor] initialized = true (трек загружен)');
        }
      },
      deep: true
    },
  },
  async mounted() {
    console.log('[SheetEditor] mounted start');
    this.wrapper = this.$refs.wrapper;
    this.instance.reposition();
    const sheetId = this.$route.params.sheet;
    this.checkLoggedIn();

    // Всегда начинаем с пустого состояния
    this.songInfo = { id: null, title: '', artist: '', url: '', mode: '4K', keys: 4 };
    this.sheetInfo = { id: null, notes: [], timing: [{ time: 0, bpm: 120 }], mode: '4K', keys: 4 };
    this.gameSheetInfo = { sheetId: null, sheet: [], song: this.songInfo };
    this.initialized = false; // НЕ инициализирован до загрузки трека
    console.log('[SheetEditor] создано пустое начальное состояние');

    window.onbeforeunload = () => {
      if (this.sheetChanged) {
        return "If you leave this page you will lose your unsaved changes.";
      }
    };
  },
  methods: {
    checkLoggedIn() {
      if (this.$store.state.initialized && !this.$store.state.verified) {
        this.$router.push({ path: "/account/", query: { warn: true } });
      }
    },
    async checkSongQuery() {
      const songId = this.$route.query.song;
      if (songId) {
        this.songInfo = await getSong(songId);
        this.$refs.info.getSheets();
      }
    },
    goToMenu() {
      this.$router.push("/studio/");
    },
    async songLoaded() {
      console.log('[SheetEditor] songLoaded вызван, initialized:', this.initialized, 'started:', this.started);
      
      if (!this.initialized) {
        // Ждем загрузки Howler
        if (this.audio && this.audio.player) {
          // Проверяем что аудио действительно загружено
          const duration = this.audio.player.duration();
          if (duration && duration > 0) {
            this.songLength = duration;
            this.instance.resetPlaying();
            this.instance.pauseGame();
            this.initialized = true;
            console.log('[SheetEditor] Трек инициализирован, длительность:', this.songLength);
            this.$store.state.alert.success('Трек готов к редактированию!');
          } else {
            console.log('[SheetEditor] Аудио еще загружается...');
            // Попробуем еще раз через 100мс
            setTimeout(() => this.songLoaded(), 100);
            return;
          }
        } else {
          console.log('[SheetEditor] Аудио плеер не готов');
          return;
        }
      } else if (!this.started) {
        // Запуск воспроизведения
        this.instance.paused = false;
        this.instance.startSong();
        console.log('[SheetEditor] Воспроизведение запущено');
      } else {
        // Возобновление
        this.resumeGame();
        console.log('[SheetEditor] Воспроизведение возобновлено');
      }
    },
    getLength() {
      let length = 0;
      if (this.srcMode === 'file' || this.srcMode === 'url') {
        // Получаем длительность из Howler
        if (this.audio && this.audio.player) {
          length = this.audio.player.duration();
        }
      } else if (this.$refs.audio) {
        length = this.$refs.audio.duration;
      }
      Logger.log('getLength:', length);
      return Number(length.toFixed(3));
    },
    pauseGame() {
      if (!this.started) return;
      this.reorderSheet();
      this.clearFever();
      this.instance.pauseGame();
      this.disableMappingTable = false;
    },
    resumeGame() {
      this.instance.resumeGame();
    },
    async restartGame() {
      if (!this.started) return;
      this.started = false;
      await this.instance.pauseGame();
      this.clearResult();
      this.instance.resetPlaying();
    },
    async seeking(time) {
      if (!this.instance.paused) this.pauseGame();
      this.instance.seekingTime = time;
      this.instance.seeked();
      this.instance.repositionNotes();
      
      // Синхронизируем позицию только через Howler
      if (this.audio && this.audio.player) {
        this.audio.player.seek(time);
      }
    },
    seekTo(time) {
      if (time < this.instance.startSongAt) {
        this.restartGame();
        this.pauseGame();
      } else {
        // Синхронизируем позицию только через Howler
        if (this.audio && this.audio.player) {
          this.audio.player.seek(time);
        }
        this.instance.seekTo(time);
      }
    },
    setPlaybackRate(rate) {
      // Устанавливаем скорость воспроизведения только через Howler
      if (this.audio && this.audio.player) {
        this.audio.player.rate(Number(rate));
      }
      this.instance.reposition();
    },
    togglePlayMode(clean) {
      this.playMode = !this.playMode;
      if (clean) {
        this.instance.clearNotes();
        this.restartGame();
      }
      this.clearFever();
      this.instance.reposition();
      this.instance.clearHoldingStatus();
    },
    updateSongDetail() {
      this.$refs.info.openSongUpdate();
    },
    async newEditor() {
      if (await this.saveWarning()) {
        // Очищаем localStorage
        localStorage.removeItem('localChart');
        console.log('[SheetEditor] localStorage очищен');
        
        // Сбрасываем состояние
        this.songInfo = { id: null, title: '', artist: '', url: '', mode: '4K', keys: 4 };
        this.sheetInfo = { id: null, notes: [], timing: [{ time: 0, bpm: 120 }], mode: '4K', keys: 4 };
        this.gameSheetInfo = { sheetId: null, sheet: [], song: this.songInfo };
        this.initialized = false;
        this.sheetChanged = false;
        
        // Перезагружаем редактор
        this.reloadEditor();
        this.$store.state.alert.success('Создан новый чарт');
      }
    },
    showPublishModal() {
      this.$refs.publishModal.show();
    },
    saveWarning() {
      if (this.sheetChanged) {
        return this.$store.state.gModal.show({
          bodyText:
            "You have unsaved changes, are you sure you want to continue without saving?",
          isError: false,
          showCancel: true,
        });
      } else {
        return Promise.resolve(true);
      }
    },

    reorderSheet() {
      this.instance.timeArr.sort((a, b) => parseFloat(a.t) - parseFloat(b.t));
    },
    saveSheet() {
      this.reorderSheet();
      this.countTotal();
      
      // Преобразуем ноты из формата Rhythm+ в формат chart.json
      const keyToLane = { d: 0, f: 1, j: 2, k: 3 };
      const notes = (this.instance.timeArr || []).map(note => ({
        time: note.t,
        type: note.type || 1,
        lane: keyToLane[note.k] !== undefined ? keyToLane[note.k] : 0
      }));
      
      // Генерируем уникальный ID для чарта
      const chartId = `chart_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
      
      // Формируем структуру чарта
      const chart = {
        meta: {
          version: 1,
          title: this.songInfo.title || "Untitled",
          artist: this.songInfo.artist || "Unknown",
          difficulty: this.sheetInfo.difficulty || "Normal",
          previewStart: 0,
          previewEnd: 10,
          music: this.songInfo.normalPath || this.songInfo.url || "",
          mode: this.sheetInfo.mode || "4K",
          keys: this.sheetInfo.keys || 4
        },
        timing: this.sheetInfo.timing || [{ time: 0, bpm: 120 }],
        notes: notes
      };
      
      // Сохраняем через ChartManager
      const success = ChartManager.saveChart(chartId, chart);
      
      // Также сохраняем музыкальный файл, если он есть
      if (this.songInfo.file) {
        ChartManager.saveMusicFile(chartId, this.songInfo.file);
        console.log('[SheetEditor] Музыкальный файл сохранен для чарта:', chartId);
      }
      
      if (success) {
        this.sheetChanged = false;
        console.log('[SheetEditor] Чарт сохранен через ChartManager:', chart);
        console.log('[SheetEditor] Проверяем localStorage после сохранения:', ChartManager.getAllCharts());
        console.log('[SheetEditor] Все ключи localStorage:', Object.keys(localStorage));
        this.$store.state.alert.success(`Чарт "${chart.meta.title}" сохранён! Нот: ${notes.length}. ID: ${chartId}`);
        
        // Обновляем список песен в SongSelect (если он открыт)
        this.$root.$emit('chart-saved');
      } else {
        this.$store.state.alert.error('Ошибка сохранения чарта!');
      }
    },
    countTotal() {
      const lastNote = this.instance.timeArr[this.instance.timeArr.length - 1];
      const endAt =
        this.sheetInfo.endAt ??
        (lastNote
          ? Math.min.apply(Math, [this.songLength, lastNote.t + 5])
          : this.songLength);
      this.sheetInfo.length = endAt - (this.sheetInfo.startAt ?? 0);
      this.sheetInfo.noteCount = this.instance.timeArr.length;
    },

    reloadEditor() {
      this.instance.destroyInstance();
      this.$store.state.redirecting = true;
      this.$nextTick(() => {
        this.$store.state.redirecting = false;
      });
    },

    exportChart() {
      // Используем ту же логику что и в saveSheet
      this.reorderSheet();
      this.countTotal();
      
      // Преобразуем ноты из формата Rhythm+ в формат chart.json
      const keyToLane = { d: 0, f: 1, j: 2, k: 3 };
      const notes = (this.instance.timeArr || []).map(note => ({
        time: note.t,
        type: note.type || 1,
        lane: keyToLane[note.k] !== undefined ? keyToLane[note.k] : 0
      }));
      
      // Генерируем уникальный ID для чарта (если экспортируем впервые)
      const chartId = `chart_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
      
      // Формируем структуру chart.json
      const chart = {
        meta: {
          version: 1,
          title: this.songInfo.title || "Untitled",
          artist: this.songInfo.artist || "Unknown",
          difficulty: this.sheetInfo.difficulty || "Normal",
          previewStart: 0,
          previewEnd: 10,
          music: this.songInfo.normalPath || this.songInfo.url || "",
          mode: this.sheetInfo.mode || "4K",
          keys: this.sheetInfo.keys || 4
        },
        timing: this.sheetInfo.timing || [{ time: 0, bpm: 120 }],
        notes: notes
      };
      
      // АВТОСОХРАНЕНИЕ: Сохраняем чарт в ChartManager перед экспортом
      const autoSaved = ChartManager.saveChart(chartId, chart);
      
      // Также сохраняем музыкальный файл, если он есть
      if (this.songInfo.file) {
        ChartManager.saveMusicFile(chartId, this.songInfo.file);
        console.log('[SheetEditor] Музыкальный файл автосохранен при экспорте для чарта:', chartId);
      }
      
      if (autoSaved) {
        console.log('[SheetEditor] Чарт автоматически сохранен в ChartManager при экспорте');
        this.$root.$emit('chart-saved'); // Обновляем список песен
        this.sheetChanged = false;
      }
      
      // Генерируем имя файла на основе названия трека
      const sanitizedTitle = (this.songInfo.title || "chart").replace(/[^a-zA-Z0-9]/g, "_").toLowerCase();
      const fileName = `${sanitizedTitle}.json`;
      
      const blob = new Blob([JSON.stringify(chart, null, 2)], { type: "application/json" });
      const url = URL.createObjectURL(blob);
      const a = document.createElement("a");
      a.href = url;
      a.download = fileName;
      document.body.appendChild(a);
      a.click();
      setTimeout(() => {
        document.body.removeChild(a);
        URL.revokeObjectURL(url);
      }, 0);
      
      // Показываем сообщение пользователю
      this.$store.state.alert.success(`✅ Чарт "${chart.meta.title}" экспортирован И автосохранен! Нот: ${notes.length}. Теперь он доступен в списке песен!`);
    },
    loadGameSheetInfo(gameSheetInfo) {
      console.log('[SheetEditor] loadGameSheetInfo вызван:', gameSheetInfo);
      
      // Обновляем данные компонента
      this.songInfo = {
        id: 'local',
        title: gameSheetInfo.meta.title,
        artist: gameSheetInfo.meta.artist,
        url: gameSheetInfo.meta.music,
        srcMode: gameSheetInfo.srcMode || 'file',
        mode: gameSheetInfo.meta.mode,
        keys: gameSheetInfo.meta.keys
      };
      
      this.sheetInfo = {
        id: 'sheet_local',
        title: `${gameSheetInfo.meta.title} [${gameSheetInfo.meta.mode}]`,
        keys: gameSheetInfo.meta.keys,
        difficulty: gameSheetInfo.meta.difficulty || 'Normal',
        timing: gameSheetInfo.timing || [{ time: 0, bpm: 120 }],
        notes: gameSheetInfo.notes || [],
        mode: gameSheetInfo.meta.mode
      };
      
      // Формируем song-объект для движка Rhythm+
      const song = {
        title: gameSheetInfo.meta.title,
        artist: gameSheetInfo.meta.artist,
        url: gameSheetInfo.meta.music, // DataURL
        srcMode: gameSheetInfo.srcMode || 'file',
        mode: gameSheetInfo.meta.mode,
        keys: gameSheetInfo.meta.keys,
        sheet: gameSheetInfo.notes || [],
        timing: gameSheetInfo.timing || [{ time: 0, bpm: 120 }],
        startAt: 0,
        visualizerName: null
      };
      
      if (this.instance && typeof this.instance.loadSong === 'function') {
        this.instance.loadSong(song);
        console.log('[SheetEditor] instance.loadSong вызван с song:', song);
      }
    },
    playChart() {
      // Сначала сохраняем чарт
      this.saveSheet();
      
      // Получаем последний сохраненный чарт
      const charts = ChartManager.getAllCharts();
      const chartIds = Object.keys(charts);
      
      if (chartIds.length > 0) {
        // Берем последний по времени создания
        const latestChartId = chartIds.reduce((latest, current) => {
          return charts[current].dateUpdated.seconds > charts[latest].dateUpdated.seconds ? current : latest;
        });
        
        // Переходим к игре с этим чартом
        this.$router.push(`/game/${latestChartId}`);
        this.$store.state.alert.success('Переход к игре с вашим чартом!');
      } else {
        this.$store.state.alert.error('Сначала сохраните чарт!');
      }
    },
  },
  beforeRouteLeave: async function (to, from, next) {
    const canLeave = await this.saveWarning();
    next(canLeave);
  },
  beforeDestroy() {
    window.onbeforeunload = null;
    this.instance.destroyInstance();
  },
};
</script>

<style scoped>
/* The navbar container */
.toolbar {
  overflow: hidden;
  background-color: rgba(63, 63, 63, 0.8);
  height: 7vh;
  display: flex;
  align-items: center;
  padding: 0 30px;
  z-index: 200;
  overflow-x: scroll;
}

.logo {
  padding: 20px;
  padding-left: 0;
  display: flex;
  align-items: center;
}

.action_buttons {
  display: flex;
  align-items: center;
  margin: 0 30px;
  width: 100px;
  justify-content: space-between;
}

.toolbar a {
  display: block;
  color: #f2f2f2;
  text-align: center;
  padding: 14px 16px;
  text-decoration: none;
  transition: 0.5s;
}

/* Links - change color on hover */
.toolbar a:hover {
  background-color: #ddd;
  color: black;
}

.perspective {
  transform: rotateX(30deg) scaleY(1.5);
  transform-origin: 50% 100%;
}

.main {
  scroll-snap-type: x mandatory;
  display: flex;
}

.column {
  height: 86vh;
}

/* Left and right column */
.column.side {
  width: 25%;
  box-sizing: border-box;
  background: linear-gradient(
    -45deg,
    rgba(56, 56, 56, 0.8),
    rgba(0, 0, 0, 0.7)
  );
}

.left {
  display: flex;
  flex-direction: column;
}

.right {
  padding: 30px;
}

.ytPlayer_editor {
  max-width: 100%;
}

/* Middle column */
.column.middle {
  width: 50%;
}

.vicon {
  cursor: pointer;
}

.tabs {
  display: flex;
  justify-content: space-evenly;
}

.tab {
  padding: 20px;
  cursor: pointer;
  transition: background-color 0.5s;
  flex-grow: 1;
  text-align: center;
}

.active {
  border-bottom: solid 2px white;
}

.tab:hover {
  background-color: rgba(255, 255, 255, 0.2);
}

.saveIndicator {
  color: orange;
  position: absolute;
  font-size: 12px;
  bottom: 3px;
  left: 47%;
}

.hideOverflow {
  overflow-y: hidden;
}

@media screen and (max-width: 600px) {
  .main {
    flex-wrap: nowrap;
    overflow-x: auto;
  }
  .column {
    flex-grow: 1;
    /* 5px scroll bar */
  }
  .column,
  .column.side,
  .column.middle {
    flex: 0 0 auto;
    width: 100vw;
    scroll-snap-align: start;
  }
}
</style>
