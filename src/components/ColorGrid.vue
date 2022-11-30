<template>
  <div class="container" :style="{ '--item-size': itemSize + 'px' }">
    <div
      v-for="(color, index) in colors"
      :key="index"
      ref="gridItems"
      class="grid-item"
    >
      <div class="front" :style="{ backgroundColor: color }"></div>
      <div class="back" :style="{ backgroundColor: color }"></div>
    </div>
  </div>
</template>


<script lang="ts">
import { ref, onMounted, onBeforeUnmount, onUpdated, nextTick } from "vue";

export default {
  setup() {
    const colors = ref<string[]>([]);
    const itemSize = ref(100);
    const intervals = ref<number[]>([]);
    const gridItems = ref([]);
    onUpdated(() => {
      gridItems.value = document.querySelectorAll('.grid-item');
    });

    const generateRandomColor = () =>
      "#" + Math.floor(Math.random() * 16777215).toString(16);

    const generateColors = () => {
      const xCount = Math.floor(window.innerWidth / itemSize.value);
      const yCount = Math.floor(window.innerHeight / itemSize.value) + 1;
      
      const count = xCount * yCount;

      for (let i = 0; i < count; i++) {
        colors.value.push(generateRandomColor());
      }
    };

    const updateSize = () => {
      let potentialSize = 320;
      while (window.innerWidth % potentialSize !== 0 && potentialSize >= 60) {
        potentialSize--;
      }
      itemSize.value = potentialSize;
      colors.value = [];
      generateColors();
    };

    const startColorChange = (index: number, delay: number) => {
      setTimeout(() => {
        gridItems.value[index].classList.add("animate-1");
        gridItems.value[index].style.transform = "rotateY(90deg)";

        gridItems.value[index].addEventListener("transitionend", function callback() {
          gridItems.value[index].removeEventListener("transitionend", callback);

          gridItems.value[index].classList.remove("animate-1");
          gridItems.value[index].classList.add("animate-2");

          const color = generateRandomColor();
          gridItems.value[index].querySelector(".front").style.backgroundColor = color;
          gridItems.value[index].querySelector(".back").style.backgroundColor = color;
          gridItems.value[index].style.transform = "rotateY(0deg)";

          setTimeout(() => {
            gridItems.value[index].classList.remove("animate-2");
          }, 1500);
        });

        const nextDelay = Math.random() * (colors.value.length / 2) * 5000;
        startColorChange(index, delay + nextDelay);
      }, delay);
    };


    const startColorIntervals = () => {
      const count = colors.value.length;
      for (let i = 0; i < count; i++) {
        const delay = Math.random() * (count / 5) * 1000;
        startColorChange(i, delay);
      }
    };

    const stopColorIntervals = () => {
      intervals.value.forEach((interval) => clearInterval(interval));
      intervals.value = [];
    };

    window.addEventListener("resize", updateSize);

    onMounted(() => {
      updateSize();
      startColorIntervals();
    });

    onBeforeUnmount(() => {
      window.removeEventListener("resize", updateSize);
      stopColorIntervals();
    });

    return {
      colors,
      itemSize,
      gridItems,
    };
  },
};
</script>


<style scoped>
body,
html {
  margin: 0;
  padding: 0;
  height: 100vh;
  width: 100vw;
  display: flex;
  justify-content: center;
  align-items: center;
  overflow: hidden; 
}

.container {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(var(--item-size), 1fr));
  grid-auto-rows: var(--item-size);
  grid-gap: 0px;
}

.grid-item {
  width: var(--item-size);
  height: var(--item-size);
  transform-style: preserve-3d;
}

.front,
.back {
  position: absolute;
  width: 100%;
  height: 100%;
  backface-visibility: hidden;
}

.front {
  z-index: 2;
}

.back {
  transform: rotateY(180deg);
}

.animate-1 {
  transition: transform 1s cubic-bezier(.61,.2,.95,.79);
}

.animate-2 {
  transition: transform 1.5s cubic-bezier(.08,.63,.4,.91);
}
</style>
