<template>
  <div class="form">
    <div v-if="formOption.tab === 'form'">
      <slot></slot>
      <div class="tags">
        <div
          v-for="tag in tags"
          :key="tag"
          :class="{ tag: true, active: formData.tags.includes(tag) }"
          @click="toggleTag(tag)"
        >
          {{ tag }}
        </div>
      </div>
      <input
        type="submit"
        :value="formOption.isUpdate ? 'Update' : 'Create'"
        @click="submitForm"
      />
    </div>
    <div v-else>
      <div class="tabs">
        <div
          class="tab"
          :class="{ active: formOption.tab === 'choose' }"
          @click="formOption.tab = 'choose'"
        >
          Choose
        </div>
        <div
          class="tab"
          :class="{ active: formOption.tab === 'form' }"
          @click="formOption.tab = 'form'"
        >
          Create
        </div>
      </div>
      <div v-if="formOption.tab === 'choose'">
        <div class="list">
          <div
            v-for="item in formOption.publicList"
            :key="item.id"
            :class="{ item: true, active: formOption.selected === item }"
            @click="selectItem(item)"
          >
            <div class="title">{{ item.title }}</div>
            <div class="artist">{{ item.artist }}</div>
          </div>
        </div>
        <input
          type="submit"
          value="Select"
          @click="submitExisting"
          :disabled="!formOption.selected"
        />
      </div>
    </div>
  </div>
</template>

<script>
import "vue-select/dist/vue-select.css";
import vSelect from "vue-select";

export default {
  name: "InfoForm",
  props: {
    formData: {
      type: Object,
      required: true,
    },
    formOption: {
      type: Object,
      required: true,
    },
    itemType: {
      type: String,
      required: true,
    },
    tags: {
      type: Array,
      default: () => [],
    },
  },
  components: {
    vSelect,
  },
  computed: {
    actionText() {
      return this.formOption.isUpdate ? "Update" : "Create";
    },
  },
  methods: {
    toggleTag(tag) {
      const index = this.formData.tags.indexOf(tag);
      if (index === -1) {
        this.formData.tags.push(tag);
      } else {
        this.formData.tags.splice(index, 1);
      }
    },
    selectItem(item) {
      this.formOption.selected = item;
    },
    submitForm() {
      this.$emit("submitForm");
    },
    submitExisting() {
      this.$emit("submitExisting");
    },
  },
};
</script>

<style scoped>
.form {
  padding: 20px;
}

.tabs {
  display: flex;
  justify-content: space-evenly;
  margin-bottom: 20px;
}

.tab {
  padding: 10px;
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

.list {
  max-height: 300px;
  overflow-y: auto;
}

.item {
  padding: 10px;
  cursor: pointer;
  transition: background-color 0.5s;
}

.item:hover {
  background-color: rgba(255, 255, 255, 0.2);
}

.item.active {
  background-color: rgba(255, 255, 255, 0.3);
}

.title {
  font-size: 1.2em;
  font-weight: bold;
}

.artist {
  opacity: 0.7;
}

.tags {
  display: flex;
  flex-wrap: wrap;
  margin: 10px 0;
}

.tag {
  padding: 5px 10px;
  margin: 5px;
  border-radius: 15px;
  cursor: pointer;
  transition: background-color 0.5s;
  background-color: rgba(255, 255, 255, 0.1);
}

.tag.active {
  background-color: rgba(255, 255, 255, 0.3);
}

.tag:hover {
  background-color: rgba(255, 255, 255, 0.2);
}

input[type="submit"] {
  width: 100%;
  padding: 10px;
  margin-top: 10px;
  background-color: rgba(255, 255, 255, 0.1);
  border: none;
  color: white;
  cursor: pointer;
  transition: background-color 0.5s;
}

input[type="submit"]:hover {
  background-color: rgba(255, 255, 255, 0.2);
}

input[type="submit"]:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.file-info {
  margin-top: 10px;
  padding: 10px;
  background-color: rgba(255, 255, 255, 0.1);
  border-radius: 5px;
}
</style>
