<template>
  <div
    ref="container"
    :class="'wheel keen-slider wheel--perspective-' + perspective"
  >
    <div
      class="wheel__shadow-top"
      :style="{
        transform: `translateZ(${radius}px)`,
        '-webkit-transform': `translateZ(${radius}px)`,
      }"
    ></div>
    <div class="wheel__inner">
      <div class="wheel__slides" :style="{ width: `${width}px` }">
        <div
          v-for="(slide, index) in slideValues"
          :key="index"
          class="wheel__slide"
          :style="slide.style"
        >
          <span>{{ slide.value }}</span>
        </div>
      </div>
      <div
        v-if="label"
        class="wheel__label"
        :style="{
          transform: `translateZ(${radius}px)`,
          '-webkit-transform': `translateZ(${radius}px)`,
        }"
      >
        {{ label }}
      </div>
    </div>
    <div
      class="wheel__shadow-bottom"
      :style="{
        transform: `translateZ(${radius}px)`,
        '-webkit-transform': `translateZ(${radius}px)`,
      }"
    ></div>
  </div>
</template>

<script lang="ts" setup>
import { ref } from "vue";

import { useKeenSlider } from "keen-slider/vue.es";
import "keen-slider/keen-slider.min.css";

import type {
  KeenSliderInstance,
  KeenSliderOptions,
  TrackDetails,
} from "keen-slider/vue.es";

const props = withDefaults(
  defineProps<{
    initIdx?: number;
    loop?: boolean;
    length: number;
    perspective?: string;
    label?: string;
    width?: number;
    setValue?: Function;
  }>(),
  {
    initIdx: 0,
    loop: false,
    perspective: "center",
    label: "",
    width: 100,
  },
);

const wheelSize = 20;
const slideDegree = 360 / 20;
const slidesPerView = props.loop ? 9 : 1;

const height = ref(0);
const radius = ref(0);
const slideValues = ref<
  {
    style: {
      transform: string;
      WebkitTransform: string;
    };
    value: number;
  }[]
>([]);

function setSlideValues(details: TrackDetails) {
  const offset = props.loop ? 1 / 2 - 1 / slidesPerView / 2 : 0;
  const values = [];

  for (let i = 0; i < props.length; i++) {
    const distance = (details.slides[i].distance - offset) * slidesPerView;
    const rotate =
      Math.abs(distance) > wheelSize / 2
        ? 180
        : distance * (360 / wheelSize) * -1;
    const style = {
      transform: `rotateX(${rotate}deg) translateZ(${radius.value}px)`,
      WebkitTransform: `rotateX(${rotate}deg) translateZ(${radius.value}px)`,
    };

    values.push({ style, value: i });
  }

  slideValues.value = values;
}

const options: KeenSliderOptions = {
  slides: {
    number: props.length,
    origin: props.loop ? "center" : "auto",
    perView: slidesPerView,
  },
  vertical: true,
  initial: props.initIdx || 0,
  loop: props.loop,
  renderMode: "performance",
  created: (s) => {
    height.value = s.size;
    radius.value = height.value / 2;
    setSlideValues(s.track.details);
  },
  updated: (slider) => {
    height.value = slider.size;
  },
  dragSpeed: (val) => {
    return (
      val *
      (height.value /
        ((height.value / 2) * Math.tan(slideDegree * (Math.PI / 180))) /
        slidesPerView)
    );
  },
  detailsChanged: (slider) => {
    setSlideValues(slider.track.details);
  },
  rubberband: true,
  mode: "free-snap",
};

function dragOutsideContainer(slider: KeenSliderInstance) {
  const breakFactorValue = 2;
  const speed = slider.options.dragSpeed || 1;
  const dragSpeed =
    typeof speed === "function"
      ? speed
      : (val: number) => val * (speed as number);

  let dragJustStarted = false;
  let isDragging = false;
  let lastValue = 0;
  let min: number;
  let max: number;
  let direction = 0;
  let size = 0;
  let windowSize = 0;
  let defaultDirection = 1;
  let sumDistance = 0;

  function setSizes() {
    size = slider.size;
    windowSize = window.innerHeight;

    if (slider.track.details) {
      min = slider.track.details.min;
      max = slider.track.details.max;
    }
  }

  function sign(x: number): number {
    return (x > 0 ? 1 : 0) - (x < 0 ? 1 : 0) || +x;
  }

  function clamp(value: number, min: number, max: number): number {
    return Math.min(Math.max(value, min), max);
  }

  function rubberband(distance: number) {
    const length = slider.track.details.length;
    const position = slider.track.details.position;
    const clampedDistance = clamp(distance, min - position, max - position);

    if (length === 0) return 0;

    if (!slider.options.rubberband) {
      return clampedDistance;
    }

    if (position <= max && position >= min) return distance;
    if (
      (position < min && direction > 0) ||
      (position > max && direction < 0)
    ) {
      return distance;
    }

    const overflow =
      (position < min ? position - min : position - max) / length;
    const trackSize = size * length;
    const overflowedSize = Math.abs(overflow * trackSize);
    const p = Math.max(0, 1 - (overflowedSize / windowSize) * breakFactorValue);

    return p * p * distance;
  }

  function onMouseDown(e: MouseEvent) {
    isDragging = true;

    sumDistance = 0;
    dragJustStarted = true;
    lastValue = e.y;

    // Добавляем обработчики для движения и отпускания мыши
    window.addEventListener("mousemove", onMouseMove);
    window.addEventListener("mouseup", onMouseUp);
    slider.container.addEventListener("mouseleave", () =>
      slider.animator.stop(),
    );
  }

  function onMouseMove(e: MouseEvent) {
    if (!isDragging) return;

    const value = e.y;

    if (dragJustStarted) {
      lastValue = value;
      dragJustStarted = false;
      slider.emit("dragChecked");
    }

    const distance = rubberband(
      (dragSpeed(lastValue - value) / size) * defaultDirection,
    );
    direction = sign(distance);

    sumDistance += distance;
    slider.track.add(distance);
    lastValue = value;

    slider.emit("dragged");
  }

  function onMouseUp() {
    if (!isDragging) return;

    isDragging = false;
    slider.emit("dragEnded");

    // Удаляем обработчики
    window.removeEventListener("mousemove", onMouseMove);
    window.removeEventListener("mouseup", onMouseUp);
  }

  slider.on("created", () => {
    setSizes();

    // Добавляем обработчик нажатия мыши
    slider.container.addEventListener("mousedown", onMouseDown);
  });

  // Очистка при уничтожении слайдера
  slider.on("destroyed", () => {
    slider.container.removeEventListener("mousedown", onMouseDown);
    window.removeEventListener("mousemove", onMouseMove);
    window.removeEventListener("mouseup", onMouseUp);
  });
}

const [container] = useKeenSlider(options, [dragOutsideContainer]);
</script>

<style>
body {
  margin: 0;
  font-family: "Inter", sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
.wheel {
  color: #fff;
  display: block;
  height: 100%;
  overflow: visible;
  width: 100%;
}
.wheel--perspective-right .wheel__inner {
  perspective-origin: calc(50% + 100px) 50%;
  transform: translateX(10px);
  -webkit-transform: translateX(10px);
}
.wheel--perspective-left .wheel__inner {
  perspective-origin: calc(50% - 100px) 50%;
  transform: translateX(-10px);
  -webkit-transform: translateX(-10px);
}

.wheel__inner {
  display: flex;
  align-items: center;
  justify-content: center;
  perspective: 1000px;
  transform-style: preserve-3d;
  height: 16%;
  width: 100%;
}

.wheel__slides {
  height: 100%;
  position: relative;
  width: 100%;
}

.wheel__shadow-top,
.wheel__shadow-bottom {
  background: linear-gradient(
    to bottom,
    rgba(0, 0, 0, 0.9) 0%,
    rgba(0, 0, 0, 0.5) 100%
  );
  left: 0;
  height: calc(42% + 2px);
  width: 100%;
  border-bottom: 0.5px solid rgba(255, 255, 255, 0.3);
  position: relative;
  margin-top: -2px;
  z-index: 5;
}

.wheel__shadow-bottom {
  background: linear-gradient(
    to bottom,
    rgba(0, 0, 0, 0.5) 0%,
    rgba(0, 0, 0, 0.9) 100%
  );
  margin-top: 2px;
  border-bottom: none;
  border-top: 0.5px solid rgba(255, 255, 255, 0.3);
}

.wheel__label {
  font-weight: 500;
  font-size: 15px;
  line-height: 1;
  margin-top: 1px;
  margin-left: 5px;
}

.wheel__slide {
  align-items: center;
  backface-visibility: hidden;
  -webkit-backface-visibility: hidden;
  display: flex;
  font-size: 20px;
  font-weight: 400;
  height: 100%;
  width: 100%;
  position: absolute;
  justify-content: flex-end;
}
</style>
