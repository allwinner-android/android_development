<!-- Copyright (C) 2019 The Android Open Source Project

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
  <md-card-content class="container">
    <div class="rects" v-if="hasScreenView">
      <rects
        :bounds="bounds"
        :rects="rects"
        :highlight="highlight"
        @rect-click="onRectClick"
      />
    </div>

    <div class="hierarchy">
      <flat-card>
        <md-content
          md-tag="md-toolbar"
          md-elevation="0"
          class="card-toolbar md-transparent md-dense"
        >
          <h2 class="md-title" style="flex: 1;">Hierarchy</h2>
          <md-checkbox
            v-model="showHierachyDiff"
            v-if="diffVisualizationAvailable"
          >
            Show Diff
          </md-checkbox>
          <md-checkbox v-model="store.simplifyNames">
            Simplify names
          </md-checkbox>
          <md-checkbox v-model="store.onlyVisible">Only visible</md-checkbox>
          <md-checkbox v-model="store.flattened">Flat</md-checkbox>
          <md-field md-inline class="filter">
            <label>Filter...</label>
            <md-input v-model="hierarchyPropertyFilterString"></md-input>
          </md-field>
        </md-content>
        <div class="tree-view-wrapper">
          <tree-view
            class="treeview"
            :item="tree"
            @item-selected="itemSelected"
            :selected="hierarchySelected"
            :filter="hierarchyFilter"
            :flattened="store.flattened"
            :items-clickable="true"
            :useGlobalCollapsedState="true"
            :simplify-names="store.simplifyNames"
            ref="hierarchy"
          />
        </div>
      </flat-card>
    </div>

    <div class="properties">
      <flat-card>
        <md-content
          md-tag="md-toolbar"
          md-elevation="0"
          class="card-toolbar md-transparent md-dense"
        >
          <h2 class="md-title" style="flex: 1">Properties</h2>
          <md-checkbox
            v-model="showPropertiesDiff"
            v-if="diffVisualizationAvailable"
          >
            Show Diff
          </md-checkbox>
          <md-field md-inline class="filter">
            <label>Filter...</label>
            <md-input v-model="propertyFilterString"></md-input>
          </md-field>
        </md-content>
        <div class="properties-content">
          <div v-if="elementSummary" class="element-summary">
            <div v-for="elem in elementSummary" v-bind:key="elem.key">
              <!-- eslint-disable-next-line max-len -->
              <span class="key">{{ elem.key }}:</span> <span class="value">{{ elem.value }}</span>
            </div>
          </div>
          <div v-if="selectedTree" class="tree-view-wrapper">
            <tree-view
              class="treeview"
              :item="selectedTree"
              :filter="propertyFilter"
              :collapseChildren="true"
              :useGlobalCollapsedState="true"
              :elementView="PropertiesTreeElement"
            />
          </div>
          <div class="no-properties" v-else>
            <i class="material-icons none-icon">
              filter_none
            </i>
            <span>No element selected in the hierachy.</span>
          </div>
        </div>
      </flat-card>
    </div>

  </md-card-content>
</template>
<script>
import TreeView from './TreeView.vue';
import Rects from './Rects.vue';
import FlatCard from './components/FlatCard.vue';
import PropertiesTreeElement from './PropertiesTreeElement.vue';

import {ObjectTransformer} from './transform.js';
import {DiffGenerator, defaultModifiedCheck} from './utils/diff.js';
import {TRACE_TYPES, DUMP_TYPES} from './decode.js';
import {stableIdCompatibilityFixup} from './utils/utils.js';
import {CompatibleFeatures} from './utils/compatibility.js';

function formatProto(obj) {
  if (obj?.prettyPrint) {
    return obj.prettyPrint();
  }
}

function findEntryInTree(tree, id) {
  if (tree.stableId === id) {
    return tree;
  }

  if (!tree.children) {
    return null;
  }

  for (const child of tree.children) {
    const foundEntry = findEntryInTree(child, id);
    if (foundEntry) {
      return foundEntry;
    }
  }

  return null;
}

export default {
  name: 'traceview',
  props: ['store', 'file', 'summarizer'],
  data() {
    return {
      propertyFilterString: '',
      hierarchyPropertyFilterString: '',
      selectedTree: null,
      hierarchySelected: null,
      lastSelectedStableId: null,
      bounds: {},
      rects: [],
      item: null,
      tree: null,
      highlight: null,
      showHierachyDiff: false,
      showPropertiesDiff: false,
      PropertiesTreeElement,
    };
  },
  methods: {
    itemSelected(item) {
      this.hierarchySelected = item;
      this.selectedTree = this.getTransformedProperties(item);
      this.highlight = item.rect;
      this.lastSelectedStableId = item.stableId;
      this.$emit('focus');
    },
    getTransformedProperties(item) {
      const transformer = new ObjectTransformer(
          item.obj,
          item.name,
          stableIdCompatibilityFixup(item),
      ).setOptions({
        skip: item.skip,
        formatter: formatProto,
      });

      if (this.showPropertiesDiff && this.diffVisualizationAvailable) {
        const prevItem = this.getItemFromPrevTree(item);
        transformer.withDiff(prevItem?.obj);
      }

      return transformer.transform();
    },
    onRectClick(item) {
      if (item) {
        this.itemSelected(item);
      }
    },
    generateTreeFromItem(item) {
      if (!this.showHierachyDiff || !this.diffVisualizationAvailable) {
        return item;
      }

      return new DiffGenerator(this.item)
          .compareWith(this.getDataWithOffset(-1))
          .withUniqueNodeId((node) => {
            return node.stableId;
          })
          .withModifiedCheck(defaultModifiedCheck)
          .generateDiffTree();
    },
    setData(item) {
      this.item = item;
      this.tree = this.generateTreeFromItem(item);

      const rects = item.rects //.toArray()
      this.rects = [...rects].reverse();
      this.bounds = item.bounds;

      this.hierarchySelected = null;
      this.selectedTree = null;
      this.highlight = null;

      function findItem(item, stableId) {
        if (item.stableId === stableId) {
          return item;
        }
        if (Array.isArray(item.children)) {
          for (const child of item.children) {
            const found = findItem(child, stableId);
            if (found) {
              return found;
            }
          }
        }
        return null;
      }

      if (this.lastSelectedStableId) {
        const found = findItem(item, this.lastSelectedStableId);
        if (found) {
          this.itemSelected(found);
        }
      }
    },
    arrowUp() {
      return this.$refs.hierarchy.selectPrev();
    },
    arrowDown() {
      return this.$refs.hierarchy.selectNext();
    },
    getDataWithOffset(offset) {
      const index = this.file.selectedIndex + offset;

      if (index < 0 || index >= this.file.data.length) {
        return null;
      }

      return this.file.data[index];
    },
    getItemFromPrevTree(entry) {
      if (!this.showPropertiesDiff || !this.hierarchySelected) {
        return null;
      }

      const id = entry.stableId;
      if (!id) {
        throw new Error('Entry has no stableId...');
      }

      const prevTree = this.getDataWithOffset(-1);
      if (!prevTree) {
        console.warn('No previous entry');
        return null;
      }

      const prevEntry = findEntryInTree(prevTree, id);
      if (!prevEntry) {
        console.warn('Didn\'t exist in last entry');
        // TODO: Maybe handle this in some way.
      }

      return prevEntry;
    },
  },
  created() {
    this.setData(this.file.data[this.file.selectedIndex ?? 0]);
  },
  watch: {
    selectedIndex() {
      this.setData(this.file.data[this.file.selectedIndex ?? 0]);
    },
    showHierachyDiff() {
      this.tree = this.generateTreeFromItem(this.item);
    },
    showPropertiesDiff() {
      if (this.hierarchySelected) {
        this.selectedTree =
            this.getTransformedProperties(this.hierarchySelected);
      }
    },
  },
  computed: {
    diffVisualizationAvailable() {
      return CompatibleFeatures.DiffVisualization && (
        this.file.type == TRACE_TYPES.WINDOW_MANAGER ||
          this.file.type == TRACE_TYPES.SURFACE_FLINGER
      );
    },
    selectedIndex() {
      return this.file.selectedIndex;
    },
    hierarchyFilter() {
      const hierarchyPropertyFilter =
          getFilter(this.hierarchyPropertyFilterString);
      return this.store.onlyVisible ? (c) => {
        return c.visible && hierarchyPropertyFilter(c);
      } : hierarchyPropertyFilter;
    },
    propertyFilter() {
      return getFilter(this.propertyFilterString);
    },
    hasScreenView() {
      return this.file.type == TRACE_TYPES.WINDOW_MANAGER ||
          this.file.type == TRACE_TYPES.SURFACE_FLINGER ||
          this.file.type == DUMP_TYPES.WINDOW_MANAGER ||
          this.file.type == DUMP_TYPES.SURFACE_FLINGER;
    },
    elementSummary() {
      if (!this.hierarchySelected || !this.summarizer) {
        return null;
      }

      const summary = this.summarizer(this.hierarchySelected);

      if (summary?.length === 0) {
        return null;
      }

      return summary;
    },
  },
  components: {
    'tree-view': TreeView,
    'rects': Rects,
    'flat-card': FlatCard,
  },
};

function getFilter(filterString) {
  const filterStrings = filterString.split(',');
  const positive = [];
  const negative = [];
  filterStrings.forEach((f) => {
    if (f.startsWith('!')) {
      const str = f.substring(1);
      negative.push((s) => s.indexOf(str) === -1);
    } else {
      const str = f;
      positive.push((s) => s.indexOf(str) !== -1);
    }
  });
  const filter = (item) => {
    const apply = (f) => f(String(item.name));
    return (positive.length === 0 || positive.some(apply)) &&
          (negative.length === 0 || negative.every(apply));
  };
  return filter;
}

</script>
<style scoped>
.container {
  display: flex;
  flex-wrap: wrap;
}

.rects {
  flex: none;
  margin: 8px;
}

.hierarchy,
.properties {
  flex: 1;
  margin: 8px;
  min-width: 400px;
  min-height: 50rem;
}

.rects,
.hierarchy,
.properties {
  padding: 5px;
}

.flat-card {
  display: flex;
  flex-direction: column;
  height: 100%;
}

.hierarchy>.tree-view,
.properties>.tree-view {
  margin: 16px;
}

.treeview {
  overflow: auto;
  white-space: pre-line;
}

.no-properties {
  display: flex;
  flex: 1;
  flex-direction: column;
  align-self: center;
  align-items: center;
  justify-content: center;
  padding: 50px 25px;
}

.no-properties .none-icon {
  font-size: 35px;
  margin-bottom: 10px;
}

.no-properties span {
  font-weight: 100;
}

.filter {
  width: auto;
}

.element-summary {
  padding: 1rem;
  border-bottom: thin solid rgba(0,0,0,.12);
}

.element-summary .key {
  font-weight: 500;
}

.element-summary .value {
  color: rgba(0, 0, 0, 0.75);
}

.properties-content {
  display: flex;
  flex-direction: column;
  flex: 1;
}

.tree-view-wrapper {
  display: flex;
  flex-direction: column;
  flex: 1;
}

.treeview {
  flex: 1 0 0;
}
</style>
