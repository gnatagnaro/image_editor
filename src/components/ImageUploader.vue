<template>
  <div class="canvasmain">
    <div class="text-top">
      <v-form v-if="type === 'local'">
        <v-file-input label="Выберите файл" @change="uploadLocal"></v-file-input>
      </v-form>
      <v-form v-if="type === 'url'">
        <v-text-field label="Введите URL изображения" v-model="imageUrl"></v-text-field>
        <v-btn @click="uploadFromUrl">Загрузить</v-btn>
      </v-form>
      <v-btn color="grey" dark @click="resizeDialog = true" class="mt-3">Настроить размер</v-btn>
      <div v-if="image">
        <p>Исходное количество пикселей: {{ originalPixels }} Мп</p>
        <p>Количество пикселей после масштабирования: {{ scaledPixels }} Мп</p>
      </div>
    </div>

    <v-dialog v-model="resizeDialog" persistent max-width="600px">
      <v-card>
        <v-card-title class="headline">Настройки изменения размера</v-card-title>
        <v-card-text>
          <v-select
            v-model="resizeMode"
            :items="resizeModes"
            label="Изменить размер в"
            class="mt-3"
          ></v-select>
          <v-checkbox
            v-model="maintainAspectRatio"
            label="Сохранять пропорции"
            class="mt-3"
          ></v-checkbox>
          <div v-if="resizeMode === 'percent'">
            <v-text-field
              v-model="percentWidth"
              label="Ширина (%)"
              type="number"
              class="mt-3"
              @input="onWidthChange('percent')"
            ></v-text-field>
            <v-text-field
              v-model="percentHeight"
              label="Высота (%)"
              type="number"
              class="mt-3"
              @input="onHeightChange('percent')"
            ></v-text-field>
          </div>
          <div v-else>
            <v-text-field
              v-model="scaledWidth"
              label="Ширина (px)"
              type="number"
              class="mt-3"
              @input="onWidthChange('pixels')"
            ></v-text-field>
            <v-text-field
              v-model="scaledHeight"
              label="Высота (px)"
              type="number"
              class="mt-3"
              @input="onHeightChange('pixels')"
            ></v-text-field>
          </div>
          <v-slider
            v-model="slider"
            :max="max"
            :min="min"
            class="align-center"
            hide-details
            @input="onSliderChange"
          >
            <template v-slot:append>
              <v-text-field
                v-model="sliderPercent"
                density="compact"
                style="width: 40px"
                type="text"
                hide-details
                single-line
              ></v-text-field>
            </template>
          </v-slider>
          <v-tooltip bottom>
            <template v-slot:activator="{ on, attrs }">
              <v-select
                v-model="interpolation"
                :items="interpolationMethods"
                label="Алгоритм интерполяции"
                class="mt-3"
                v-bind="attrs"
                v-on="on"
              ></v-select>
            </template>
            <span>
              Ближайший сосед: простой и быстрый алгоритм интерполяции, который выбирает ближайший к целевому пиксель в исходном изображении. Это может привести к блочным артефактам.
            </span>
          </v-tooltip>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="primary" @click="closeResizeDialog">Применить</v-btn>
          <v-btn color="grey" @click="closeResizeDialog">Отменить</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
    
    <image-pixel-search 
      ref="pixelSearch"
      :imageWidth="imageWidth"
      :imageHeight="imageHeight">
    </image-pixel-search>

    <image-colors></image-colors>
  </div>
</template>

<script>
import ImagePixelSearch from './ImagePixelSearch.vue';
import ImageColors from './ImageColors.vue';

export default {
  components: {
    ImagePixelSearch,
    ImageColors
  },
  props: {
    type: String
  },
  data() {
    return {
      min: 12,
      max: 300,
      slider: 100,
      imageUrl: "",
      image: null,
      originalPixels: 0,
      scaledPixels: 0,
      imageWidth: 0,
      imageHeight: 0,
      resizeDialog: false,
      resizeMode: 'percent',
      resizeModes: [
        { text: 'Проценты', value: 'percent' },
        { text: 'Пиксели', value: 'pixels' }
      ],
      scaledWidth: 0,
      scaledHeight: 0,
      percentWidth: 100,
      percentHeight: 100,
      maintainAspectRatio: true,
      interpolation: 'nearest',
      interpolationMethods: [
        { text: 'Ближайший сосед', value: 'nearest' },
        { text: 'Без интерполяции', value: 'none' }
      ]
    };
  },
  computed: {
    sliderPercent() {
      return `${this.slider}%`;
    },
    aspectRatio() {
      return this.image ? this.image.width / this.image.height : 1;
    }
  },
  methods: {
    uploadLocal(file) {
      const reader = new FileReader();
      reader.onload = (e) => {
        const img = new Image();
        img.onload = () => {
          this.image = img;
          this.imageWidth = img.width;
          this.imageHeight = img.height;
          this.scaledWidth = img.width;
          this.scaledHeight = img.height;
          this.percentWidth = 100;
          this.percentHeight = 100;
          this.slider = 100;
          this.calculatePixels();
          this.drawImageToCanvas();
        };
        img.src = e.target.result;
      };
      reader.readAsDataURL(file);
    },
    uploadFromUrl() {
      const img = new Image();
      img.crossOrigin = 'Anonymous';
      img.onload = () => {
        this.image = img;
        this.imageWidth = img.width;
        this.imageHeight = img.height;
        this.scaledWidth = img.width;
        this.scaledHeight = img.height;
        this.percentWidth = 100;
        this.percentHeight = 100;
        this.slider = 100;
        this.calculatePixels();
        this.drawImageToCanvas();
      };
      img.src = this.imageUrl;
    },
    calculatePixels() {
      this.originalPixels = (this.image.width * this.image.height / 1000000).toFixed(2);
      let scaledWidth, scaledHeight;
      if (this.resizeMode === 'percent') {
        scaledWidth = this.image.width * (this.percentWidth / 100);
        scaledHeight = this.image.height * (this.percentHeight / 100);
      } else {
        scaledWidth = this.scaledWidth;
        scaledHeight = this.scaledHeight;
      }
      this.scaledPixels = (scaledWidth * scaledHeight / 1000000).toFixed(2);
    },
    drawImageToCanvas() {
      if (!this.image) return;

      const canvas = this.$refs.pixelSearch.$refs.canvas;
      const ctx = canvas.getContext('2d');
      const canvasWidth = canvas.parentElement.clientWidth;
      const canvasHeight = canvas.parentElement.clientHeight;

      let scaledWidth, scaledHeight;
      if (this.resizeMode === 'percent') {
        scaledWidth = this.image.width * (this.percentWidth / 100);
        scaledHeight = this.image.height * (this.percentHeight / 100);
      } else {
        scaledWidth = this.scaledWidth;
        scaledHeight = this.scaledHeight;
      }

      const x = (canvasWidth - scaledWidth) / 2;
      const y = (canvasHeight - scaledHeight) / 2;

      canvas.width = canvasWidth;
      canvas.height = canvasHeight;

      ctx.clearRect(0, 0, canvas.width, canvas.height);

      if (this.interpolation === 'nearest') {
        this.drawNearestNeighbor(ctx, this.image, scaledWidth, scaledHeight);
      } else if (this.interpolation === 'none') {
        ctx.drawImage(this.image, x, y, scaledWidth, scaledHeight);
      }

      this.calculatePixels();
    },
    drawNearestNeighbor(ctx, img, width, height) {
      const imgCanvas = document.createElement('canvas');
      const imgCtx = imgCanvas.getContext('2d');
      imgCanvas.width = img.width;
      imgCanvas.height = img.height;
      imgCtx.drawImage(img, 0, 0);

      const imgData = imgCtx.getImageData(0, 0, img.width, img.height).data;
      const scaledImgData = ctx.createImageData(width, height);

      for (let y = 0; y < height; y++) {
        for (let x = 0; x < width; x++) {
          const srcX = Math.floor(x * img.width / width);
          const srcY = Math.floor(y * img.height / height);
          const srcIndex = (srcY * img.width + srcX) * 4;
          const destIndex = (y * width + x) * 4;
          scaledImgData.data[destIndex] = imgData[srcIndex];
          scaledImgData.data[destIndex + 1] = imgData[srcIndex + 1];
          scaledImgData.data[destIndex + 2] = imgData[srcIndex + 2];
          scaledImgData.data[destIndex + 3] = imgData[srcIndex + 3];
        }
      }

      ctx.putImageData(scaledImgData, (ctx.canvas.width - width) / 2, (ctx.canvas.height - height) / 2);
    },
    onWidthChange(mode) {
      if (this.maintainAspectRatio) {
        if (mode === 'percent') {
          this.percentHeight = (this.percentWidth / this.aspectRatio).toFixed(2);
        } else {
          this.scaledHeight = Math.round(this.scaledWidth / this.aspectRatio);
        }
      }
      this.drawImageToCanvas();
    },
    onHeightChange(mode) {
      if (this.maintainAspectRatio) {
        if (mode === 'percent') {
          this.percentWidth = (this.percentHeight * this.aspectRatio).toFixed(2);
        } else {
          this.scaledWidth = Math.round(this.scaledHeight * this.aspectRatio);
        }
      }
      this.drawImageToCanvas();
    },
    onSliderChange() {
      const scale = this.slider / 100;
      this.percentWidth = (scale * 100).toFixed(2);
      this.percentHeight = (scale * 100).toFixed(2);
      this.scaledWidth = Math.round(this.image.width * scale);
      this.scaledHeight = Math.round(this.image.height * scale);
      this.drawImageToCanvas();
    },
    closeResizeDialog() {
      this.resizeDialog = false;
    }
  }
}
</script>

<style>
/* .canvasmain {
  display: flex;
  flex-direction: column;
  align-items: center;
} */
.text-top {
  margin-bottom: 20px;
  text-align: center;
}
canvas {
  width: 1500px;
  height: 100vh;
  padding: 50px;
}
</style>
