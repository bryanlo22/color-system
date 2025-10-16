<script lang="ts" setup>
import { ref } from "vue";
import chroma from "chroma-js";

import "modern-normalize";

const colorSteps = ref(7);
const colors: Ref<string[]> = ref([]);

const showHex = ref(false);
const showRGB = ref(false);
const showHSL = ref(false);
const showLCH = ref(false);
const showContrast = ref(false);

const firstStepChroma = ref(29);
const firstStepLightness = ref(101);
const middleStepLightness = ref(52);
const lastStepChroma = ref(21);
const lastStepLightness = ref(0);

onMounted(() => {
  let initialColorCount: number;
  const windowWidth = window.innerWidth;
  if (windowWidth < 512) {
    initialColorCount = 3;
  } else if (windowWidth < 768) {
    initialColorCount = 4;
  } else {
    initialColorCount = 5;
  }

  const initialHue = randomHue();

  for (let i = 0; i < initialColorCount; i++) {
    const hue = initialHue + i * (360 / (initialColorCount + 1));
    const color = randomColor(hue % 360);
    colors.value.push(color.hex());
  }
});

function randomHue() {
  return Math.floor(Math.random() * 360);
}

function randomColor(hue: number) {
  return chroma.hcl(hue, 40 + Math.random() * 10, 50);
}

function lightest(hex: string) {
  const hue = chroma(hex).hcl()[0];
  return chroma.hcl(hue, firstStepChroma.value, firstStepLightness.value).hex();
}

function darkest(hex: string) {
  const hue = chroma(hex).hcl()[0];
  return chroma.hcl(hue, lastStepChroma.value, lastStepLightness.value).hex();
}

function scaled(hex: string) {
  return chroma
    .scale([
      lightest(hex),
      chroma(hex).set("hcl.l", middleStepLightness.value),
      darkest(hex),
    ])
    .colors(colorSteps.value);
}

function getContrastColor(j: number, baseHex: string) {
  if (j < colorSteps.value / 2) {
    return darkest(baseHex);
  } else {
    return lightest(baseHex);
  }
}

function getContrastValue(j: number, baseHex: string, currentHex: string) {
  return (
    Math.round(
      100 * chroma.contrast(currentHex, getContrastColor(j, baseHex))
    ) / 100
  );
}

function toPercentage(value: number) {
  return Math.round(100 * value);
}

function getRGB(hex: string) {
  const [r, g, b] = chroma(hex).rgb();
  return `rgb(${Math.round(r)}, ${Math.round(g)}, ${Math.round(b)})`;
}

function getHSL(hex: string) {
  const [h, s, l] = chroma(hex).hsl();
  return `hsl(${Math.round(h)}deg ${toPercentage(s)}% ${toPercentage(l)}%)`;
}

function getLCH(hex: string) {
  const [l, c, h] = chroma(hex).lch();
  return `lch(${Math.round(l)}% ${Math.round(c)} ${Math.round(h)})`;
}
</script>

<template>
  <div class="page">
    <div class="controls">
      <div class="checkboxes">
        <label>
          <input v-model="showHex" type="checkbox" />
          Hex
        </label>
        <label>
          <input v-model="showRGB" type="checkbox" />
          RGB (Red, Green, Blue)
        </label>
        <label>
          <input v-model="showHSL" type="checkbox" />
          HSL (Hue, Saturation, Lightness)
        </label>
        <label>
          <input v-model="showLCH" type="checkbox" />
          LCH (Lightness, Chroma, Hue)
        </label>
        <label>
          <input v-model="showContrast" type="checkbox" />
          WCAG Contrast Ratio
        </label>
      </div>
      <div class="customization">
        <label>
          Color Steps:
          <input
            v-model="colorSteps"
            type="number"
            @input="
              () => {
                if (colorSteps < 5) {
                  colorSteps = 5;
                }
                if (colorSteps > 15) {
                  colorSteps = 15;
                }
                colorSteps = Math.floor(colorSteps);
              }
            "
          />
        </label>
        <label>
          First Step Chroma:
          <input v-model="firstStepChroma" type="number" />
        </label>
        <label>
          First Step Lightness:
          <input v-model="firstStepLightness" type="number" />
        </label>
        <label>
          Middle Step Lightness:
          <input v-model="middleStepLightness" type="number" />
        </label>
        <label>
          Last Step Chroma:
          <input v-model="lastStepChroma" type="number" />
        </label>
        <label>
          Last Step Lightness:
          <input v-model="lastStepLightness" type="number" />
        </label>
        <button
          v-if="colors.length < 9"
          @click="() => colors.push(randomColor(randomHue()).hex())"
        >
          Add Color
        </button>
      </div>
    </div>
    <div class="color-grid">
      <div v-for="(baseHex, i) in colors" :key="i" class="colors">
        <input
          :value="baseHex"
          type="color"
          @input="
            (event) => {
              const target = event?.target as HTMLInputElement;
              if (target?.value) {
                colors[i] = target.value;
              }
            }
          "
        />
        <template v-for="(currentHex, j) in scaled(baseHex)" :key="j">
          <div
            class="color"
            :style="{
              backgroundColor: currentHex,
              color: getContrastColor(j, baseHex),
            }"
          >
            <div v-if="showHex">
              {{ currentHex }}
            </div>
            <div v-if="showRGB">
              {{ getRGB(currentHex) }}
            </div>
            <div v-if="showHSL">
              {{ getHSL(currentHex) }}
            </div>
            <div v-if="showLCH">
              {{ getLCH(currentHex) }}
            </div>
            <div v-if="showContrast" class="contrast">
              <template v-if="getContrastValue(j, baseHex, currentHex) >= 4.5">
                <Icon name="fa6-solid:check" />
              </template>
              <template v-else>
                <Icon name="fa6-solid:xmark" />
              </template>
              {{ getContrastValue(j, baseHex, currentHex).toFixed(2) }}
            </div>
          </div>
        </template>
        <button
          class="submit"
          :style="{
            backgroundColor: scaled(baseHex).at(-2),
            color: lightest(baseHex),
          }"
          @mouseover="
            (event) => {
              const target = event?.target as HTMLButtonElement;
              if (target?.style) {
                target.style.backgroundColor = scaled(baseHex).at(-3) || '';
              }
            }"
          @mouseout="
              (event) => {
                const target = event?.target as HTMLButtonElement;
                if (target?.style) {
                  target.style.backgroundColor = scaled(baseHex).at(-2) || '';
                }
              }"
        >
          Button
        </button>
        <div
          class="tag"
          :style="{
            backgroundColor: scaled(baseHex).at(1),
            color: scaled(baseHex).at(-2),
          }"
        >
          Tag
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.page {
  display: flex;
  flex-direction: column;
  gap: 20px;
  font-family: "Open Sans", sans-serif;
  font-size: 16px;
}

.controls {
  padding: 10px;
  margin: 0 auto;
  display: flex;
  flex-direction: column;
  gap: 10px;
}
@media (min-width: 768px) {
  .controls {
    flex-direction: row;
    gap: 40px;
  }
}

.checkboxes {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.customization {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

label {
  width: fit-content;
  display: flex;
  align-items: center;
  gap: 5px;
}

input[type="number"] {
  width: 50px;
}

input[type="checkbox"] {
  margin: 0;
}

.color-grid {
  display: flex;
  justify-content: space-evenly;
  padding: 10px;
}
@media (min-width: 1024px) {
  .color-grid {
    width: 100%;
    max-width: 1440px;
    margin: 0 auto;
  }
}

.colors {
  width: 100%;
  display: flex;
  flex-direction: column;
}

input[type="color"] {
  width: 100%;
}

.color {
  width: 100%;
  min-height: 100px;
  height: 10svh;
  max-height: 200px;
  padding: 5px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 5px;
  text-align: center;
}

.contrast {
  display: flex;
  align-items: center;
  gap: 5px;
}

.submit {
  height: 50px;
  border: none;
  border-radius: 20px;
  cursor: pointer;
  margin: 10px;
  font-family: "Open Sans", sans-serif;
  font-size: 16px;
  font-weight: 700;
}

.tag {
  margin: 0 auto;
  padding: 5px 15px;
  font-size: 14px;
  font-weight: 700;
  border-radius: 10px;
}
</style>
