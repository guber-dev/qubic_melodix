<template>
  <div>
    <v-bar class="fullPage">
      <div class="pageTitle">Song Select</div>

      <div class="list_and_detail">
        <!-- song list -->
        <div
          class="song_list_wrapper"
          :class="{ list_collapsed: selectedSong }"
        >
          <div class="reflow2 song_list">
          </div>

          <SongList
            class="song_list"
            :songs="songList"
            @selected="selectedSong = $event"
            ref="list"
            :sorter="false"
          >
          </SongList>
        </div>

        <!-- detail panel -->
        <div :class="{ detail: true, detail_collapsed: !selectedSong }">
          <transition name="slide-fade">
            <SongDetailPanel
              v-if="selectedSong"
              :song="selectedSong"
              :sheets="sheetList"
              @cancel="selectedSong = null"
            ></SongDetailPanel>
          </transition>
        </div>
      </div>

      <!-- loading -->
      <Loading :show="!songList || songList.length === 0" :delay="true"
        >Fetching Songs...</Loading
      >
      
      <!-- Debug panel -->
      <div style="position: fixed; top: 100px; right: 20px; background: rgba(0,0,0,0.8); padding: 15px; color: white; font-size: 12px; z-index: 1000; border-radius: 5px;" v-if="true">
        <div><strong>DEBUG INFO</strong></div>
        <div>Всего песен: {{ songList ? songList.length : 0 }}</div>
        <div>Пользовательских чартов: {{ debugChartsCount }}</div>
        <div>LocalStorage ключи: {{ debugLocalStorageCount }}</div>
        <button @click="showLocalStorageContent" style="margin: 5px; padding: 5px; font-size: 10px;">Показать localStorage</button>
        <button @click="clearAllUserCharts" style="margin: 5px; padding: 5px; font-size: 10px; background: red; color: white;">Очистить чарты</button>
        <button @click="forceShowTestSongs" style="margin: 5px; padding: 5px; font-size: 10px; background: green; color: white;">Показать тестовые</button>
      </div>

      <!-- song suggestion modal -->
      <Modal
        ref="suggest"
        :showOk="false"
        cancelText="Done"
        titleText="Suggest a Song"
      >
        <template>
          <iframe
            src="https://docs.google.com/forms/d/e/1FAIpQLSf4nNnTn0vmYjWYbq3TeC6epuN8xkEhxlWONrtIMMZbgLJ38w/viewform?embedded=true"
            style="width: 100%"
            height="500"
            frameborder="0"
            marginheight="0"
            marginwidth="0"
            >Loading…</iframe
          >
        </template>
      </Modal>
    </v-bar>
  </div>
</template>

<script>
import SongDetailPanel from "../components/menus/SongDetailPanel.vue";
import SongList from "../components/menus/SongList.vue";
import Loading from "../components/ui/Loading.vue";
import Modal from "../components/ui/Modal.vue";
import {
  // getSheetList, // Оставляем, так как может понадобиться для загрузки чартов при выборе песни
  // getSongList, // Удаляем
  // getPlaylist, // Удаляем
  // getSongsInIdArray, // Удаляем
} from "../javascript/db";
import "vue-awesome/icons/lightbulb";
import "vue-awesome/icons/question-circle";
import { logEvent } from "../helpers/analytics";
import "vue-awesome/icons/info-circle";
import smoothReflow from "vue-smooth-reflow";

// Система управления чартами
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

  static deleteChart(chartId) {
    try {
      const charts = this.getAllCharts();
      delete charts[chartId];
      localStorage.setItem('melodixCharts', JSON.stringify(charts));
      console.log(`[ChartManager] Чарт ${chartId} удален`);
      return true;
    } catch (error) {
      console.error('[ChartManager] Ошибка удаления чарта:', error);
      return false;
    }
  }

  static chartExists(chartId) {
    const charts = this.getAllCharts();
    return charts.hasOwnProperty(chartId);
  }

  static getChartsAsSongList() {
    const charts = this.getAllCharts();
    return Object.values(charts).map(chart => ({
      id: chart.id,
      title: chart.meta.title || "Untitled",
      artist: chart.meta.artist || "Unknown Artist",
      image: "/songs/test/local.jpg", // Дефолтная картинка
      dateUpdated: chart.dateUpdated,
      sheetList: [
        {
          sheetId: chart.id,
          songId: chart.id,
          difficulty: chart.meta.difficulty || "Normal",
          mode: chart.meta.mode || "4K",
          difficultyRating: 3,
          length: 30
        }
      ]
    }));
  }

  static saveMusicFile(chartId, file) {
    try {
      const reader = new FileReader();
      reader.onload = (e) => {
        const dataUrl = e.target.result;
        localStorage.setItem(`melodix_music_${chartId}`, dataUrl);
        console.log(`[ChartManager] Музыка для чарта ${chartId} сохранена как DataURL`);
      };
      reader.readAsDataURL(file);
    } catch (error) {
      console.error('[ChartManager] Ошибка сохранения музыки:', error);
    }
  }

  static getMusicDataUrl(chartId) {
    try {
      return localStorage.getItem(`melodix_music_${chartId}`);
    } catch (error) {
      console.error('[ChartManager] Ошибка загрузки музыки:', error);
      return null;
    }
  }

  // Функция миграции старых чартов
  static migrateOldCharts() {
    try {
      const oldChart = localStorage.getItem('localChart');
      if (oldChart) {
        const chart = JSON.parse(oldChart);
        const chartId = 'migrated_' + Date.now();
        
        // Сохраняем старый чарт в новой системе
        const success = this.saveChart(chartId, chart);
        if (success) {
          console.log('[ChartManager] Старый чарт мигрирован:', chartId);
          // Удаляем старый чарт
          localStorage.removeItem('localChart');
          return chartId;
        }
      }
      return null;
    } catch (error) {
      console.error('[ChartManager] Ошибка миграции:', error);
      return null;
    }
  }
}

// Тестовые данные (будут показываться только если нет пользовательских чартов)
const testSongsData = [
  {
    id: "tame_impala",
    title: "The Less I Know The Better",
    artist: "Tame Impala",
    image: "/songs/test/local.jpg",
    dateUpdated: { seconds: Date.now() / 1000 - 86400 }, // Вчера
    sheetList: [
      {
        sheetId: "tame_impala",
        songId: "tame_impala", 
        difficulty: "Normal",
        mode: "4K",
        difficultyRating: 3,
        length: 30
      }
    ]
  },
  {
    id: "bronx",
    title: "Stupid",
    artist: "Toni Braxton",
    image: "/songs/test/cover.jpg",
    dateUpdated: { seconds: Date.now() / 1000 - 172800 }, // Позавчера
    sheetList: [
      {
        sheetId: "bronx",
        songId: "bronx", 
        difficulty: "Normal",
        mode: "4K",
        difficultyRating: 3,
        length: 30
      }
    ]
  },
  {
    id: "vladimir",
    title: "Vladimirskiy Tsentral", 
    artist: "Mikhail Krug",
    image: "/songs/test/vladimir.jpg",
    dateUpdated: { seconds: Date.now() / 1000 - 259200 }, // Три дня назад
    sheetList: [
      {
        sheetId: "vladimir",
        songId: "vladimir",
        difficulty: "Normal",
        mode: "4K",
        difficultyRating: 3,
        length: 30
      }
    ]
  }
];

export default {
  name: "SongSelect",
  components: {
    SongDetailPanel,
    Loading,
    Modal,
    SongList,
  },
  data() {
    return {
      allSongs: null, // Будет содержать тестовые данные
      songList: [],
      latestSongList: null,
      sheetList: [],
      selectedSong: null,
    };
  },
  computed: {
    debugChartsCount() {
      try {
        const charts = ChartManager.getAllCharts();
        return Object.keys(charts || {}).length;
      } catch (error) {
        return 0;
      }
    },
    debugLocalStorageCount() {
      try {
        return Object.keys(localStorage || {}).length;
      } catch (error) {
        return 0;
      }
    }
  },
  mixins: [smoothReflow],
  async mounted() {
    // Мигрируем старые чарты, если они есть
    const migratedChartId = ChartManager.migrateOldCharts();
    if (migratedChartId) {
      console.log('[SongSelect] Мигрирован старый чарт:', migratedChartId);
    }
    
    // ВРЕМЕННО: всегда показываем тестовые данные для отладки
    this.allSongs = [...testSongsData]; // Создаем копию массива
    console.log('[SongSelect] Показываем тестовые песни для отладки');
    
    // Загружаем чарты из localStorage через ChartManager
    // const userCharts = ChartManager.getChartsAsSongList();
    
    // Показываем тестовые данные только если нет пользовательских чартов
    // if (userCharts && userCharts.length > 0) {
    //   this.allSongs = userCharts;
    //   console.log('[SongSelect] Загружено пользовательских чартов:', userCharts.length);
    // } else {
    //   this.allSongs = [...testSongsData]; // Создаем копию массива
    //   console.log('[SongSelect] Показываем тестовые песни');
    // }
    
    this.songList = this.songListSortByDate();
    
    // Сортируем по дате (самые новые сверху)
    if (this.$refs.list) {
      this.$refs.list.sort("date");
      this.$refs.list.reverseSort = true;
    }

    // Слушаем событие сохранения чарта для автоматического обновления
    this.$root.$on('chart-saved', this.refreshSongList);
  },
  beforeDestroy() {
    // Убираем слушатель при уничтожении компонента
    this.$root.$off('chart-saved', this.refreshSongList);
  },
  methods: {
    handleHover() {
      this.$store.state.audio.playHoverEffect("ui/ta");
    },
        async getAllSongs(getUserOwned) {
      // ВРЕМЕННО: всегда показываем тестовые данные для отладки
      this.allSongs = testSongsData;
      console.log('[SongSelect] Показываем тестовые песни для отладки');
      
      // Загружаем чарты из localStorage через ChartManager
      // const userCharts = ChartManager.getChartsAsSongList();
      
      // Показываем тестовые данные только если нет пользовательских чартов
      // if (userCharts.length > 0) {
      //   this.allSongs = userCharts;
      //   console.log('[SongSelect] Обновлено пользовательских чартов:', userCharts.length);
      // } else {
      //   this.allSongs = testSongsData;
      //   console.log('[SongSelect] Пользовательских чартов нет, показываем тестовые:', testSongsData.length);
      // }
      
      this.songList = this.songListSortByDate();
       if (this.$refs.list) {
         this.$refs.list.sort("date");
         this.$refs.list.reverseSort = true;
       }
    },

    songListSortByDate() {
      if (!this.allSongs || this.allSongs.length === 0) {
        return [];
      }
      // Сортировка по дате: b.dateUpdated.seconds - a.dateUpdated.seconds для новых сверху
      return [...this.allSongs].sort(
        (a, b) => b.dateUpdated.seconds - a.dateUpdated.seconds
      );
    },

    // Добавляю watcher для selectedSong, который был закомментирован
    async getSheetsForSelectedSong() {
       this.sheetList = null; // Сбрасываю список чартов при выборе новой песни
       if (this.selectedSong && this.selectedSong.sheetList) {
         // Для локальных тестовых песен берем sheetList из тестовых данных
         this.sheetList = this.selectedSong.sheetList;
         logEvent("song_selected", { id: this.selectedSong.id });
       } else if (this.selectedSong) {
         // Если это не локальная тестовая песня (например, если добавите другие позже)
         // пытаемся загрузить чарты из базы (если нужно)
          // this.sheetList = await getSheetList(this.selectedSong.id, true, true); // Закомментировано для полной изоляции
          // logEvent("song_selected", { id: this.selectedSong.id });
          this.sheetList = []; // Просто пустой список для не-тестовых песен
       }
     },

    // Добавляем метод для обновления списка (будет вызываться после сохранения чарта)
    refreshSongList() {
      this.getAllSongs();
    },

    // Метод для очистки всех пользовательских чартов (для отладки)
    clearAllUserCharts() {
      if (confirm('Удалить ВСЕ пользовательские чарты? Это действие нельзя отменить!')) {
        localStorage.removeItem('melodixCharts');
        localStorage.removeItem('localChart'); // На всякий случай
        this.refreshSongList();
        this.$store.state.alert.success('Все пользовательские чарты удалены');
      }
    },

    // Дебаг методы
    getAllChartsDebug() {
      return ChartManager.getAllCharts();
    },

    showLocalStorageContent() {
      const charts = ChartManager.getAllCharts();
      const allKeys = Object.keys(localStorage);
      console.log('=== СОДЕРЖИМОЕ LOCALSTORAGE ===');
      console.log('melodixCharts:', charts);
      console.log('Все ключи localStorage:', allKeys);
      allKeys.forEach(key => {
        console.log(`${key}:`, localStorage.getItem(key));
      });
      alert(`Пользовательских чартов: ${Object.keys(charts).length}\nВсе ключи: ${allKeys.join(', ')}\nПодробности в консоли`);
    },

    forceShowTestSongs() {
      console.log('[SongSelect] Принудительно показываем тестовые песни');
      this.allSongs = testSongsData;
      this.songList = this.songListSortByDate();
      if (this.$refs.list) {
        this.$refs.list.sort("date");
        this.$refs.list.reverseSort = true;
      }
    },
  },
    watch: {
      // Переношу watcher для selectedSong сюда
     selectedSong: 'getSheetsForSelectedSong',
  },
};
</script>

<style scoped>
.subtitle {
  font-size: 2em;
}

.song_list {
  min-width: 300px;
  margin: 0 20px;
}

.latest_song_list {
  margin: 0;
  margin-bottom: 39px !important;
  clear: right;
}

.song_list_wrapper {
  transition: 1s;
  z-index: 100;
  min-width: 350px;
  max-width: 800px;
  width: 100%;
}

.list_and_detail {
  display: flex;
  justify-content: center;
}

.detail {
  transition: 1s;
  width: 350px;
}

.btn-more {
  float: right;
  margin: 0;
}

.big-add {
  width: 100%;
  line-height: 80px;
  max-width: 100%;
  box-sizing: border-box;
  position: relative;
  padding-left: 10%;
}

.add-icon {
  position: absolute;
  left: 10%;
  top: 30px;
}

.cat {
  margin: auto;
  width: fit-content;
}

.cat_tab {
  margin: 10px;
  font-weight: bolder;
  padding-bottom: 5px;
  transition: all 0.2s;
  opacity: 0.6;
  color: white;
  border-bottom: 2px solid transparent;
  cursor: pointer;
}

.cat_tab:hover {
  opacity: 0.8;
}

.active {
  opacity: 0.9;
  border-bottom: 2px solid white;
}

.new-info {
  max-width: 500px;
  margin: 10px auto;
  opacity: 0.5;
  text-align: left;
  font-size: 14px;
  width: 90%;
}

@media only screen and (max-width: 800px) {
  /* mobile */
  .list_collapsed {
    width: 0;
    visibility: hidden;
    margin: 0;
    padding: 0;
    min-width: 0;
    transform: translateX(-120vw);
  }
}

@media only screen and (min-width: 800px) {
  /* desktop */
  .detail {
    /* transform: rotateY(-30deg); */
    margin: 0 20px;
  }
  .list_and_detail {
    margin: 50px;
  }
  .subtitle {
    font-size: 3em;
  }
  .btn-more {
    margin-top: 10px;
  }
}

@media only screen and (max-width: 1000px) {
  /* tablet */
  .list_and_detail {
    margin: 30px 0;
  }
}

.detail_collapsed {
  width: 0;
  visibility: hidden;
  margin: 0;
}

.slide-fade-enter-active,
.slide-fade-leave-active {
  transition: all 0.7s ease;
}
.slide-fade-enter,
.slide-fade-leave-to {
  transform: translateX(10px);
  opacity: 0;
}
</style>
