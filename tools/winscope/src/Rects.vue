<!-- Copyright (C) 2017 The Android Open Source Project

     Licensed under the Apache License, Version 2.0 (the "License");
     you may not use this file except in compliance with the License.
     You may obtain a copy of the License at

          http://www.apache.org/licenses/LICENSE-2.0

     Unless required by applicable law or agreed to in writing, software
     distributed under the License is distributed on an "AS IS" BASIS,
     WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
     See the License for the specific language governing permissions and
     limitations under the License.
-->
<template>
  <div class="bounds" :style="boundsStyle">
    <div
      class="rect" v-for="r in filteredRects"
      :style="rectToStyle(r)"
      @click="onClick(r)"
      v-bind:key="`${r.left}-${r.right}-${r.top}-${r.bottom}-${r.ref.name}`"
    >
      <span class="label">{{r.label}}</span>
    </div>
    <div class="highlight" v-if="highlight" :style="rectToStyle(highlight)" />
  </div>
</template>

<script>

// eslint-disable-next-line camelcase
import {multiply_rect} from './matrix_utils.js';

export default {
  name: 'rects',
  props: ['bounds', 'rects', 'highlight'],
  data() {
    return {
      desiredHeight: 800,
      desiredWidth: 400,
    };
  },
  computed: {
    boundsC() {
      if (this.bounds) {
        return this.bounds;
      }
      const width = Math.max(
          ...this.rects.map((r) => multiply_rect(r.transform, r).right));
      const height = Math.max(
          ...this.rects.map((r) => multiply_rect(r.transform, r).bottom));
      return {width, height};
    },
    boundsStyle() {
      return this.rectToStyle({top: 0, left: 0, right: this.boundsC.width,
        bottom: this.boundsC.height});
    },
    filteredRects() {
      return this.rects.filter((rect) => {
        const isVisible = rect.ref.visible ?? rect.ref.isVisible;
        console.warn(`Name: ${rect.ref.name} Kind: ${rect.ref.kind} isVisible=${isVisible}`);
        return isVisible;
      });
    },
  },
  methods: {
    s(sourceCoordinate) { // translate source into target coordinates
      let scale;
      if (this.boundsC.width < this.boundsC.height) {
        scale = this.desiredHeight / this.boundsC.height;
      } else {
        scale = this.desiredWidth / this.boundsC.width;
      }
      return sourceCoordinate * scale;
    },
    rectToStyle(r) {
      const x = this.s(r.left);
      const y = this.s(r.top);
      const w = this.s(r.right) - this.s(r.left);
      const h = this.s(r.bottom) - this.s(r.top);
      const t = r.transform;
      const tr = t ? `matrix(${t.dsdx}, ${t.dtdx}, ${t.dsdy}, ${t.dtdy}, ` +
          `${this.s(t.tx)}, ${this.s(t.ty)})` : '';
      return `top: ${y}px; left: ${x}px; height: ${h}px; width: ${w}px;` +
             `transform: ${tr}; transform-origin: 0 0;`;
    },
    onClick(r) {
      this.$emit('rect-click', r.ref);
    },
  },
};
</script>

<style scoped>
.bounds {
  position: relative;
  overflow: hidden;
}
.highlight, .rect {
  position: absolute;
  box-sizing: border-box;
  display: flex;
  justify-content: flex-end;
}
.rect {
  border: 1px solid black;
  background-color: rgba(110, 114, 116, 0.8);
}
.highlight {
  border: 2px solid rgb(235, 52, 52);
  pointer-events: none;
}
.label {
  align-self: center;
}
</style>
