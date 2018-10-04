<template>
<div>
    <div :class="sizeClasses">
        <div class="input-group-text" ref="prependDiv" v-if="$slots.prepend || prepend" >
            <div  class="input-group-prepend px-1">
                <slot name="prepend">
                    <span class="input-group-text">{{ prepend }}</span>
                </slot>
            </div>
        </div>
        <input
        ref="input"
        type="search"
        :class="`form-control ${inputClass}`"
        :placeholder="placeholder"
        :aria-label="placeholder"
        :value="value"
        @focus="isFocused = true"
        @blur="handleBlur"
        @input="$emit('input', $event.target.value)"
        autocomplete="off"
      />
        <div class="input-group-text"  v-if="$slots.append || append" >
            <div class="input-group-append px-1">
                <slot name="append">
                    <span class="input-group-text">{{ append }}</span>
                </slot>
            </div>
        </div>
    </div>
    <vue-bootstrap-typeahead-list v-if="cols === false" class="vbt-autcomplete-list" ref="list" v-show="isFocused && data.length > 0" :query="value" :data="formattedData" :background-variant="backgroundVariant" :text-variant="textVariant" :minMatchingChars="minMatchingChars" @hit="handleHit">
        <!-- pass down all scoped slots -->
        <template v-for="(slot, slotName) in $scopedSlots" :slot="slotName" slot-scope="{ data, htmlText }">
            <slot :name="slotName" v-bind="{ data, htmlText }"></slot>
        </template>
        <!-- below is the right solution, however if the user does not provide a scoped slot, vue will still set $scopedSlots.suggestion to a blank scope
      <template v-if="$scopedSlots.suggestion" slot="suggestion" slot-scope="{ data, htmlText }">
        <slot name="suggestion" v-bind="{ data, htmlText }" />
      </template>-->
    </vue-bootstrap-typeahead-list>
    <b-row v-else>
        <b-col class="p-0 z-top" :cols="Math.round(12 / formattedData.filter(list => list.length > 0).length)" v-for="(col, index) in formattedData" :key="index" v-show="col.length > 0">
            <vue-bootstrap-typeahead-list class="w-100 p-absolute" :title="cols[index].title" :id="`typeahead-input-${index}`" :query="value" :data="col" :background-variant="backgroundVariant" :text-variant="textVariant" v-show="isFocused && col.length > 0" :minMatchingChars="minMatchingChars" @hit="handleHit">
                <!-- pass down all scoped slots -->
                <template v-for="(slot, slotName) in $scopedSlots" :slot="slotName" slot-scope="{ data, htmlText }">
                    <slot :name="slotName" v-bind="{ data, htmlText }"></slot>
                </template>
                <!-- below is the right solution, however if the user does not provide a scoped slot, vue will still set $scopedSlots.suggestion to a blank scope
          <template v-if="$scopedSlots.suggestion" slot="suggestion" slot-scope="{ data, htmlText }">
            <slot name="suggestion" v-bind="{ data, htmlText }" />
          </template>-->
            </vue-bootstrap-typeahead-list>
        </b-col>

    </b-row>
</div>
</template>

<script>
import VueBootstrapTypeaheadList from "./VueBootstrapTypeaheadList.vue";
import ResizeObserver from "resize-observer-polyfill";

export default {
  name: "VueBootstrapTypehead",

  components: {
    VueBootstrapTypeaheadList
  },

  props: {
    size: {
      type: String,
      default: null,
      validator: size => ["lg", "sm"].indexOf(size) > -1
    },
    value: String,
    data: {
      type: Array,
      required: true,
      validator: d => d instanceof Array
    },
    serializer: {
      type: Function,
      default: d => d,
      validator: d => d instanceof Function
    },
    backgroundVariant: String,
    textVariant: String,
    inputClass: {
      type: String,
      default: ""
    },
    maxMatches: {
      type: Number,
      default: 10
    },
    minMatchingChars: {
      type: Number,
      default: 2
    },
    placeholder: String,
    prepend: String,
    append: String,
    cols: {
      type: Array | Boolean,
      default: false
    }
  },

  computed: {
    sizeClasses() {
      return this.size ? `input-group input-group-${this.size}` : "input-group";
    },

    formattedData() {
      if (!(this.data instanceof Array)) {
        return [];
      }
      let retVal = this.data.map((d, i) => {
        return {
          id: i,
          data: d,
          text: this.serializer(d)
        };
      });

      if (this.cols !== false && this.cols.length > 0) {
        let retList = new Array(this.cols.length);
        this.cols.forEach((col, index) => {
          retList[index] = retVal.filter(entry => col.filter(entry.data));
        });
        console.log({
          retList,
          retVal
        });
        return retList;
      } else return retVal;
    }
  },

  methods: {
    resizeList(el) {
      if (!this.cols || this.cols.length === 0) {
        const rect = el.getBoundingClientRect();
        const listStyle = this.$refs.list.$el.style;

        // Set the width of the list on resize
        listStyle.width = rect.width + "px";

        // Set the margin when the prepend prop or slot is populated
        // (setting the "left" CSS property doesn't work)
        if (this.$refs.prependDiv) {
          const prependRect = this.$refs.prependDiv.getBoundingClientRect();
          listStyle.marginLeft = prependRect.width + "px";
        }
      } else if (false) {
        const rect = el.getBoundingClientRect();
        const lists = this.cols.map((col, index) => {
          return this.$el.querySelector(`#typeahead-input-${index}`);
        });
        lists.forEach(list => {
          list &&
            (() => {
              const listStyle = list.style;
              listStyle.width = Math.round(rect.width / 2) + "px";
              listStyle.minWidth = "196px";
            })();
        });
      }
    },

    handleHit(evt) {
      this.$emit("input", evt.text);
      this.$emit("hit", evt.data);
      this.$refs.input.blur();
      this.isFocused = false;
    },

    handleBlur(evt) {
      const tgt = evt.relatedTarget;
      if (tgt && tgt.classList.contains("vbst-item")) {
        return;
      }
      this.isFocused = false;
    }
  },

  data() {
    return {
      isFocused: false
    };
  },

  mounted() {
    this.$_ro = new ResizeObserver(e => {
      this.$refs.input && this.resizeList(this.$refs.input);
    });
    this.$refs.input && this.$_ro.observe(this.$refs.input);
    this.$refs.list && this.$_ro.observe(this.$refs.list.$el);
  },

  beforeDestroy() {
    this.$_ro.disconnect();
  }
};
</script>

<style scoped>
.vbt-autcomplete-list {
  padding-top: 5px;
  position: absolute;
  max-height: 350px;
  overflow-y: auto;
  z-index: 999;
}
.z-top {
  z-index: 9999;
}
.p-absolute {
  position: absolute;
}
</style>
