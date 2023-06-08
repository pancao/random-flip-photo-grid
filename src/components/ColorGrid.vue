<template>
  <div class="container" :style="{ '--item-size': itemSize + 'px' }">
    <div
      v-for="(image, index) in displayedImages"
      :key="image.id"
      ref="gridItemRefs"
      class="grid-item"
    >
      <div class="front" :style="{ backgroundImage: 'url(' + image + ')' }"></div>
      <div class="back" :style="{ backgroundImage: 'url(' + image + ')' }"></div>
    </div>
  </div>
</template>

<script lang="ts">
import { ref, onMounted, onBeforeUnmount, onUpdated, nextTick } from "vue";
import supabase from "../supabase";

export default {
  setup() {
    const displayedImages = ref<{id: string, url: string}[]>([]);
    const pendingImages = ref<string[]>([]);
    const itemSize = ref(100);
    const gridItems = ref([]);
    const gridItemRefs = ref<HTMLElement[]>([]);
    const timeouts = ref<NodeJS.Timeout[]>([]);

    const updateSize = () => {
      let potentialSize = 320;
      while (window.innerWidth % potentialSize !== 0 && potentialSize >= 100) {
        potentialSize--;
      }
      itemSize.value = potentialSize;
      displayedImages.value = []; // reset displayedImages
      generateImages();
    };

    const generateImages = async () => {
      const xCount = Math.floor(window.innerWidth / itemSize.value);
      const yCount = Math.floor(window.innerHeight / itemSize.value) + 1;
      const count = xCount * yCount;

      // Create an array with the correct length and fill it with empty strings
      displayedImages.value = new Array(count).fill('');

      let { data: images, error, count: totalImages } = await supabase
        .from('coverurl')
        .select('url', { count: 'exact' })
        .range(0, count - 1);

      if (error) {
        console.error('Error loading images:', error);
        return;
      }

      const loadPromises = images.map((image, i) => {
        if (image) {
          return new Promise<void>((resolve) => {
            const img = new Image();
            img.src = image.url;
            img.onload = () => {
              displayedImages.value[i] = { id: i.toString(), url: image.url };
              resolve();
            };
            img.onerror = () => {
              console.error(`Failed to load image at ${image.url}`);
              resolve();  // even if image loading failed, resolve this promise
            };
          });
        }
        return Promise.resolve();
      });

      const timeoutPromise = new Promise<void>((resolve) => {
        const id = setTimeout(() => {
          clearTimeout(id);
          resolve();  // resolve this promise after timeout
        }, 10000);  
      });


      await Promise.race([Promise.all(loadPromises), timeoutPromise]);

      if (totalImages > count) {
        let { data: remainingImages, error: remainingImagesError } = await supabase
          .from('coverurl')
          .select('url')
          .range(count, totalImages - 1);

        if (remainingImagesError) {
          console.error('Error loading remaining images:', remainingImagesError);
          return;
        }

        for (let i = 0; i < remainingImages.length; i++) {
          if (remainingImages[i]) {
            const image = new Image();
            image.src = remainingImages[i].url;
            image.onload = function() {
              pendingImages.value.push(remainingImages[i].url);
            };
          }
        }
      }

      // Generate images is completed, start the image change intervals
      startImageChangeIntervals();
    };


    
    const startImageChange = (index: number, delay: number) => {
      if (!gridItemRefs.value[index]) {
        return;
      }

      let imageUrl = '';

      if (pendingImages.value.length === 0) {
        pendingImages.value = displayedImages.value.map(item => item.url);
        displayedImages.value[index] = { id: displayedImages.value[index].id, url: '' };
      }

      imageUrl = pendingImages.value.shift() || '';

      const image = new Image();
      image.src = imageUrl;
      image.onload = function() {
        displayedImages.value[index] = { id: displayedImages.value[index].id, url: imageUrl };

        const backElement = gridItemRefs.value[index].querySelector(".back") as HTMLElement;
        const frontElement = gridItemRefs.value[index].querySelector(".front") as HTMLElement;

        backElement.style.backgroundImage = `url(${imageUrl})`;

        const timeoutId = setTimeout(() => {
          gridItemRefs.value[index].classList.add("animate-1");
          gridItemRefs.value[index].style.transform = "rotateY(90deg) scale(0.9)";

          gridItemRefs.value[index].addEventListener("transitionend", function callback() {
            if (!gridItemRefs.value[index]) {
              return;
            }

            gridItemRefs.value[index].removeEventListener("transitionend", callback);

            gridItemRefs.value[index].classList.remove("animate-1");
            gridItemRefs.value[index].classList.add("animate-2");
          
            frontElement.style.backgroundImage = `url(${imageUrl})`;
            gridItemRefs.value[index].style.transform = "rotateY(0deg) scale(1)";

            setTimeout(() => {
              gridItemRefs.value[index].classList.remove("animate-2");
            }, 1500);
          });

          const nextDelay = Math.random() * (displayedImages.value.length / 2) * 5000;
          startImageChange(index, delay + nextDelay);
        }, delay);

        timeouts.value.push(timeoutId);
      };
    };




    const startImageChangeIntervals = () => {
      const count = displayedImages.value.length;
      for (let i = 0; i < count; i++) {
        const delay = Math.random() * (count / 5) * 1000;
        startImageChange(i, delay);
      }
    };

    const stopImageChangeIntervals = () => {
      timeouts.value.forEach((timeoutId) => clearTimeout(timeoutId));
      timeouts.value = [];
    };

    window.addEventListener("resize", updateSize);

    onUpdated(async () => {
      if (gridItems.value.length > 0 && displayedImages.value.length === gridItems.value.length) {
        await nextTick();
        startImageChangeIntervals();
      }
    });

    onMounted(() => {
      updateSize();
    });

    onBeforeUnmount(() => {
      window.removeEventListener("resize", updateSize);
      stopImageChangeIntervals();
    });

    return {
      displayedImages,
      itemSize,
      gridItemRefs,
      gridItems
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
  transform-style: preserve-3d;
}

.grid-item {
  width: var(--item-size);
  height: var(--item-size);
  perspective: 400px;
}

.front,
.back {
  position: absolute;
  width: 100%;
  height: 100%;
  backface-visibility: hidden;
  background-size: cover;
  perspective: 900px;
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
  animation: shine 1.5s ease;
}

.fade-in {
  animation: fade-in 3s linear forwards;
  opacity: 0;
}

@keyframes shine {
  0% { 
    filter: brightness(1);
  }
  10% { 
    filter: brightness(1.5);
  }
  100% { 
    filter: brightness(1);
  }
}

@keyframes fade-in {
  0% { opacity: 0 }
  100% { opacity: 1 }
}
</style>
