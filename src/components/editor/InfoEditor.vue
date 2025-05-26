<template>
  <div class="padding" style="height: 100%; overflow: scroll;">
    <div v-if="!$parent.songInfo.id && welcomeScreen">
      <h2>Welcome to R+ Sheet Editor</h2>
      <p>
        It is very easy to create a beatmap in Rhythm+, just create as you are
        playing one!
      </p>
      <p>What would you like to create today?</p>
      <input
        type="submit"
        value="Create a new sheet"
        @click="welcomeScreen = false"
      />
      <input
        type="submit"
        value="Choose existing or Continue your work"
        @click="continueExisting"
      />
    </div>

    <div v-else>
      <!-- song create update form -->
      <div v-if="!$parent.songInfo.id || songFormOptions.isUpdate">
        <h2>Выберите трек для создания чарта</h2>
        <div style="text-align: center; padding: 20px;">
          <input
            type="file"
            accept="audio/*"
            @change="onAudioFileChange"
            name="audioFile"
            id="audioFile"
            style="display: none;"
          />
          <label for="audioFile" class="file-select-btn">
            📁 Выбрать аудиофайл
          </label>
          <div v-if="songFormData.fileName" class="file-info" style="margin-top: 15px;">
            <strong>{{ songFormData.title }}</strong><br>
            <em>{{ songFormData.artist }}</em><br>
            <small>{{ songFormData.fileName }}</small>
          </div>
        </div>
      </div>

      <div v-if="$parent.songInfo.id">
        <div style="text-align: center; padding: 20px;">
          <h2>🎵 Трек загружен!</h2>
          <div class="loaded-song-info">
            <strong>{{ $parent.songInfo.title }}</strong><br>
            <em>{{ $parent.songInfo.artist }}</em>
          </div>
          <div style="margin: 20px 0;">
            <div class="chart-info">
              Режим: <strong>4 клавиши</strong><br>
              Сложность: <strong>Normal</strong>
            </div>
          </div>
          <button
            @click="change(false)"
            class="change-song-btn"
            style="margin-top: 15px;"
          >
            🔄 Выбрать другой трек
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import {
  createSong,
  createSheet,
  getSongList,
  getSong,
  getSheetList,
  updateSong,
  updateSheet,
  getTags,
} from "../../javascript/db";
import InfoForm from "./InfoForm.vue";

export default {
  name: "InfoEditor",
  components: {
    InfoForm,
  },
  data() {
    return {
      songFormData: {
        title: null,
        artist: null,
        image: null,
        srcMode: "file",
        youtubeId: null,
        url: null,
        file: null,
        fileName: null,
        tags: [],
        normalPath: null,
      },
      songFormOptions: {
        isYoutubeMode: false,
        tab: "form",
        publicList: null,
        privateList: null,
        selected: null,
        isUpdate: false,
      },
      sheetFormData: {
        title: null,
        difficulty: null,
        visualizerName: null,
        keys: null,
      },
      sheetFormOptions: {
        isYoutubeMode: false,
        tab: "form",
        publicList: null,
        privateList: null,
        selected: null,
        isUpdate: false,
      },
      welcomeScreen: false, // Сразу показываем форму выбора файла
      tags: [],
    };
  },
  computed: {
    isSheetOwner() {
      return true;
    },
    isSongOwner() {
      return true;
    },
  },
  watch: {
    "$parent.sheetInfo"() {
      let sheetInfo = this.$parent.sheetInfo;
      if (sheetInfo.id != null) {
        this.sheetFormData = this.$parent.sheetInfo;
        this.sheetFormOptions.isUpdate = true;
        this.sheetFormOptions.tab = "form";
      }
    },
  },
  async mounted() {
    // Не загружаем ничего с сервера, только локальные данные
    this.tags = [];
    if (this.$route.query.song) {
      this.sheetFormOptions.tab = "choose";
    }
  },
  methods: {
    continueExisting() {
      this.welcomeScreen = false;
      this.songFormOptions.tab = "choose";
      this.sheetFormOptions.tab = "choose";
    },
    submitSongForm() {
      if (!this.songFormData.title || !this.songFormData.artist || !this.songFormData.file) {
        this.$store.state.alert.error('Заполни все поля и выбери аудиофайл!', 4000);
        return;
      }
      if (!this.songFormData.id) this.songFormData.id = 'local';
      this.$parent.songInfo = { ...this.songFormData };
      console.log('[InfoEditor] submitSongForm, songInfo:', this.$parent.songInfo);
      this.$store.state.alert.success("Song info updated (local only)");
    },
    submitExistingSong() {
      // Просто копируем выбранную песню в songInfo
      let selectedSong = this.songFormOptions.selected;
      if (selectedSong) {
        if (!selectedSong.id) selectedSong.id = 'local';
        this.$parent.songInfo = selectedSong;
        console.log('[InfoEditor] submitExistingSong, songInfo:', this.$parent.songInfo);
      }
    },
    submitSheetForm() {
      if (!this.sheetFormData.title || !this.sheetFormData.keys) {
        this.$store.state.alert.error('Заполни все поля Sheet Detail!', 4000);
        return;
      }
      if (!this.sheetFormData.id) this.sheetFormData.id = 'sheet_local';
      this.$parent.sheetInfo = { ...this.sheetFormData };
      // Формируем структуру gameSheetInfo максимально похожую на Rhythm+
      const gameSheetInfo = {
        meta: {
          version: 1,
          title: this.$parent.songInfo.title,
          artist: this.$parent.songInfo.artist,
          difficulty: 'Normal',
          music: this.$parent.songInfo.url,
          mode: `${this.sheetFormData.keys}K`,
        },
        timing: [{ time: 0, bpm: 120 }],
        notes: [],
        srcMode: 'file',
      };
      this.$parent.gameSheetInfo = gameSheetInfo;
      console.log('[InfoEditor] submitSheetForm, gameSheetInfo:', gameSheetInfo);
      if (this.$parent && typeof this.$parent.loadGameSheetInfo === 'function') {
        this.$parent.loadGameSheetInfo(gameSheetInfo);
      }
      this.$store.state.alert.success("Sheet info updated (local only)");
    },
    submitExistingSheet() {
      let selectedSheet = this.sheetFormOptions.selected;
      if (selectedSheet) {
        if (!selectedSheet.id) selectedSheet.id = 'local';
        this.$parent.sheetInfo = selectedSheet;
        console.log('[InfoEditor] submitExistingSheet, sheetInfo:', this.$parent.sheetInfo);
      }
    },
    handleFormError(err) {
      this.$store.state.alert.error("Please fill in required fields", 5000);
      Logger.error(err);
    },
    openSongUpdate() {
      this.songFormOptions.tab = "form";
      this.songFormOptions.isUpdate = true;
      this.songFormData = { ...this.$parent.songInfo };
    },
    async change(isChangeSheet) {
      if (await this.$parent.saveWarning()) {
        const songId = isChangeSheet ? "?song=" + this.$parent.songInfo.id : "";
        this.$router.push("/editor/" + songId);
        this.$parent.reloadEditor();
      }
    },
    onAudioFileChange(e) {
      const file = e.target.files[0];
      if (file) {
        this.songFormData.file = file;
        this.songFormData.fileName = file.name;
        
        // Автоматически извлекаем название и артиста из имени файла
        this.extractTitleAndArtist(file.name);
        
        // Сохраняем файл в папку /public/songs/test/ и используем обычный путь
        const fileName = file.name;
        const normalizedPath = `/songs/test/${fileName}`;
        
        // Создаем FormData для отправки файла на сервер
        const formData = new FormData();
        formData.append('audioFile', file);
        formData.append('fileName', fileName);
        
        // Пока что используем временный blob URL для редактора, 
        // но в экспорте будем использовать нормальный путь
        const tempBlobUrl = URL.createObjectURL(file);
        
        this.songFormData.url = tempBlobUrl;
        this.songFormData.normalPath = normalizedPath; // Сохраняем нормальный путь отдельно
        this.songFormData.srcMode = 'file';
        
        // Обновляем songInfo родителя
        this.$parent.songInfo = {
          id: 'local',
          title: this.songFormData.title,
          artist: this.songFormData.artist,
          url: tempBlobUrl, // Для воспроизведения в редакторе
          normalPath: normalizedPath, // Для экспорта
          file: file,
          fileName: file.name,
          srcMode: 'file',
          mode: '4K',
          keys: 4
        };
        
        // Копируем файл в public/songs/test/ 
        this.copyFileToPublicFolder(file, fileName);
        
        console.log('[InfoEditor] Файл загружен, songInfo обновлён:', this.$parent.songInfo);
        this.$store.state.alert.success(`Трек загружен: ${this.songFormData.title} - ${this.songFormData.artist}`);
        
        // Автоматически переходим к редактированию
        this.autoStartEditing();
      }
    },
    
    copyFileToPublicFolder(file, fileName) {
      // Поскольку мы работаем в браузере, мы не можем напрямую копировать файлы
      // Но мы можем сохранить информацию о файле и показать пользователю инструкции
      console.log(`[InfoEditor] Нужно скопировать файл ${fileName} в папку /public/songs/test/`);
      
      // Показываем уведомление пользователю
      this.$store.state.alert.info(
        `Не забудьте скопировать файл "${fileName}" в папку /public/songs/test/ для корректной работы экспорта!`, 
        8000
      );
    },
    
    extractTitleAndArtist(fileName) {
      // Убираем расширение файла
      const nameWithoutExt = fileName.replace(/\.(mp3|wav|ogg|m4a|flac)$/i, '');
      
      // Пробуем разные форматы: "Artist - Title", "Title - Artist", или просто "Title"
      if (nameWithoutExt.includes(' - ')) {
        const parts = nameWithoutExt.split(' - ');
        if (parts.length >= 2) {
          this.songFormData.artist = parts[0].trim();
          this.songFormData.title = parts.slice(1).join(' - ').trim();
        }
      } else {
        // Если нет разделителя, используем имя файла как название
        this.songFormData.title = nameWithoutExt.trim();
        this.songFormData.artist = 'Unknown Artist';
      }
    },
    
    autoStartEditing() {
      // Автоматически устанавливаем sheet данные и переходим к редактированию
      this.sheetFormData.title = `${this.songFormData.title} [4K]`;
      this.sheetFormData.keys = 4;
      this.sheetFormData.difficulty = 'Normal';
      
      // Сразу создаём sheet info
      this.$parent.sheetInfo = {
        id: 'sheet_local',
        title: this.sheetFormData.title,
        keys: 4,
        difficulty: 'Normal',
        timing: [{ time: 0, bpm: 120 }],
        notes: [],
        mode: '4K'
      };
      
      // Создаём gameSheetInfo для движка
      const gameSheetInfo = {
        meta: {
          version: 1,
          title: this.songFormData.title,
          artist: this.songFormData.artist,
          difficulty: 'Normal',
          music: this.songFormData.normalPath || this.songFormData.url, // Используем normalPath для экспорта
          mode: '4K',
          keys: 4
        },
        timing: [{ time: 0, bpm: 120 }],
        notes: [],
        srcMode: 'file',
      };
      
      this.$parent.gameSheetInfo = gameSheetInfo;
      console.log('[InfoEditor] Автоматически созданы sheet и gameSheet:', gameSheetInfo);
      
      // Загружаем в движок
      if (this.$parent && typeof this.$parent.loadGameSheetInfo === 'function') {
        this.$parent.loadGameSheetInfo(gameSheetInfo);
      }
    },
  },
};
</script>

<style>
.padding {
  padding: 30px;
}
.switch_tab {
  text-align: center;
  opacity: 0.5;
  font-size: 14px;
  margin: 10px;
  cursor: pointer;
}
.changeBtn {
  float: right;
  line-height: 30px;
  opacity: 0.5;
  cursor: pointer;
}

.file-select-btn {
  display: inline-block;
  padding: 15px 30px;
  background: linear-gradient(45deg, #667eea 0%, #764ba2 100%);
  color: white;
  border-radius: 8px;
  cursor: pointer;
  font-size: 16px;
  font-weight: bold;
  transition: all 0.3s ease;
  border: none;
  box-shadow: 0 4px 15px rgba(0,0,0,0.2);
}

.file-select-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(0,0,0,0.3);
  background: linear-gradient(45deg, #764ba2 0%, #667eea 100%);
}

.file-info {
  background: rgba(255,255,255,0.1);
  border-radius: 8px;
  padding: 15px;
  border-left: 4px solid #667eea;
}

.loaded-song-info {
  background: rgba(102, 126, 234, 0.2);
  border-radius: 8px;
  padding: 15px;
  margin: 15px 0;
  border: 1px solid rgba(102, 126, 234, 0.3);
}

.chart-info {
  background: rgba(255,255,255,0.05);
  border-radius: 6px;
  padding: 10px;
  font-size: 14px;
}

.change-song-btn {
  padding: 10px 20px;
  background: rgba(255,255,255,0.1);
  color: white;
  border: 1px solid rgba(255,255,255,0.2);
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.3s ease;
}

.change-song-btn:hover {
  background: rgba(255,255,255,0.2);
  transform: translateY(-1px);
}
</style>
