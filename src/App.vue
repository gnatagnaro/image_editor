<template>
  <v-app style="background-color: darkorange;">
    <v-container fill-height>
      <v-row align="center" justify="center">
        <v-dialog v-model="dialog" persistent max-width="auto">
          <template v-slot:activator="{ on }">
            <v-btn color="grey" dark v-on="on" class="large-button">Загрузить изображение</v-btn>
          </template>
          <v-card>
            <v-card-title class="headline">Хотите загрузить изображение?</v-card-title>
            <v-card-text>Выберите способ загрузки изображения:</v-card-text>
            <v-card-actions>
              <v-spacer></v-spacer>
              <v-btn color="lime" flat @click="setUploadType('local')">Локально</v-btn>
              <v-btn color="white" flat @click="setUploadType('url')">URL</v-btn>
              <v-btn color="red" flat @click="closeDialog">Отменить</v-btn>
              <v-spacer></v-spacer>
            </v-card-actions>
          </v-card>
        </v-dialog>
      </v-row>
      <v-row justify="center">
        <image-uploader v-if="uploadType" :type="uploadType"></image-uploader>
      </v-row>
    </v-container>
  </v-app>
</template>

<script>
import ImageUploader from './components/ImageUploader.vue';

export default {
  components: {
    ImageUploader
  },
  data () {
    return {
      dialog: false,
      uploadType: null,
    }
  },
  methods: {
    setUploadType(type) {
      this.uploadType = type;
      this.dialog = false;
    },
    closeDialog() {
      this.dialog = false;
    }
  }
}
</script>

<style>
html, body, #app, .v-application {
  height: 100%;
  margin: 0;
}

.large-button {
  font-size: 24px; /* Увеличиваем размер текста */
  padding: 20px 40px; /* Увеличиваем внутренние отступы */
  width: 500px;
}
</style>
