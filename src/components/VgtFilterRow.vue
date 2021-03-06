<template>
  <tr v-if="hasFilterRow">
    <th v-if="lineNumbers"></th>
    <th v-if="selectable"></th>
    <th class="filter-th"
        v-for="(column, index) in columns" :key="index"
        v-if="!column.hidden">

      <div
              v-if="isFilterable(column)">
        <input v-if="!isDropdown(column) && !isMultiselectDropdown(column)"
               type="text"
               class="vgt-input"
               :placeholder="getPlaceholder(column)"
               :value="columnFilters[column.field]"
               @keyup.enter="updateFiltersOnEnter(column, $event.target.value)"
               @input="updateFiltersOnKeyup(column, $event.target.value)" />

        <!-- options are a list of primitives -->
        <select v-if="isDropdownArray(column)"
                class="vgt-select"
                :value="columnFilters[column.field]"
                @change="updateFilters(column, $event.target.value)">
          <option value="" key="-1">{{ getPlaceholder(column) }}</option>
          <option
                  v-for="(option, i) in column.filterOptions.filterDropdownItems"
                  :key="i"
                  :value="option">
            {{ option }}
          </option>
        </select>

        <!-- options are a list of objects with text and value -->
        <select v-if="isDropdownObjects(column)"
                class="vgt-select"
                :value="columnFilters[column.field]"
                @change="updateFilters(column, $event.target.value, true)">
          <option value="" key="-1">{{ getPlaceholder(column) }}</option>
          <option
                  v-for="(option, i) in column.filterOptions.filterDropdownItems"
                  :key="i"
                  :value="option.value">{{ option.text }}</option>
        </select>

        <!--Manually populated-->
        <v-select v-if="isMultiselectDropdown(column) === 'manual'"
                  :options="column.filterOptions.filterMultiselectDropdownItems"
                  :loading="column.filterOptions.loading"
                  :placeholder="getPlaceholder(column)"
                  multiple
                  @input="(selectedItems) => updateFiltersOnKeyup(column, selectedItems)"
                  :ref="'vgt-multiselect' + column.label + index"
        />
        <!--Auto populated by the rows, does not work in remote table-->
        <v-select
          v-if="isMultiselectDropdown(column) === 'auto'"
          :options="filterDropdownOptions[column.field]"
          :loading="column.filterOptions.loading"
          :placeholder="getPlaceholder(column)"
          multiple
          @input="(selectedItems) => updateFiltersOnKeyup(column, selectedItems)"
          :ref="'vgt-multiselect' + column.label + index"
        />

      </div>
    </th>
  </tr>
</template>

<script>
  import isEqual from 'lodash.isequal';

  export default {
    name: 'VgtFilterRow',
    props: [
      'filterDropdownOptions',
      'lineNumbers',
      'columns',
      'typedColumns',
      'globalSearchEnabled',
      'selectable',
      'mode',
    ],
    watch: {
      columns: {
        handler(newValue, oldValue) {
          this.populateInitialFilters();
        },
        deep: true,
        immediate: true,
      },
    },
    data() {
      return {
        columnFilters: {},
        timer: null,
      };
    },

    computed: {
      // to create a filter row, we need to
      // make sure that there is atleast 1 column
      // that requires filtering
      hasFilterRow() {
        // if (this.mode === 'remote' || !this.globalSearchEnabled) {
        for (let i = 0; i < this.columns.length; i++) {
          const col = this.columns[i];
          if (col.filterOptions && col.filterOptions.enabled) {
            return true;
          }
        }
        // }
        return false;
      },
    },
    methods: {
      reset(emitEvent = false) {
        this.columnFilters = {};

        // Clear the selection in the multiselect
        let multiselectRefs = this.columns
                .map((column, index) => {
                  column.ref = 'vgt-multiselect' + column.label + index;
                  return column;
                })
                .filter(this.isMultiselectDropdown)
                .map(column => column.ref);
        multiselectRefs.forEach((ref) => this.$refs[ref][0].clearSelection());

        if (emitEvent) {
          this.$emit('filter-changed', this.columnFilters);
        }
      },

      isFilterable(column) {
        return column.filterOptions
                && column.filterOptions.enabled;
      },

      isDropdown(column) {
        return this.isFilterable(column)
                && column.filterOptions.filterDropdownItems
                && column.filterOptions.filterDropdownItems.length;
      },

      isDropdownObjects(column) {
        return this.isDropdown(column)
                && typeof column.filterOptions.filterDropdownItems[0] === 'object';
      },

      isDropdownArray(column) {
        return this.isDropdown(column)
                && typeof column.filterOptions.filterDropdownItems[0] !== 'object';
      },

      isMultiselectDropdown(column) {
        if(this.isFilterable(column) && column.filterOptions.filterMultiselectDropdownItems && column.filterOptions.filterMultiselectDropdownItems.length){
          return 'manual';
        }
        if(this.isFilterable(column) && column.filterOptions.filterMultiselectDropdownItems && !column.filterOptions.filterMultiselectDropdownItems.length)
          return 'auto'

        // return this.isFilterable(column)
        //         && column.filterOptions.filterMultiselectDropdownItems;
      },

      // get column's defined placeholder or default one
      getPlaceholder(column) {
        const placeholder = (this.isFilterable(column) && column.filterOptions.placeholder) || `Filter ${column.label}`;
        return placeholder;
      },

      updateFiltersOnEnter(column, value) {
        if (this.timer) clearTimeout(this.timer);
        this.updateFiltersImmediately(column, value);
      },

      updateFiltersOnKeyup(column, value) {
        // if the trigger is enter, we don't filter on keyup
        if (column.filterOptions.trigger === 'enter') return;
        this.updateFilters(column, value);
      },

      // since vue doesn't detect property addition and deletion, we
      // need to create helper function to set property etc
      updateFilters(column, value) {
        if (this.timer) clearTimeout(this.timer);
        this.timer = setTimeout(() => {
          this.updateFiltersImmediately(column, value);
        }, 400);
      },

      updateFiltersImmediately(column, value) {
        this.$set(this.columnFilters, column.field, value);
        this.$emit('filter-changed', this.columnFilters);
      },

      populateInitialFilters() {
        for (let i = 0; i < this.columns.length; i++) {
          const col = this.columns[i];
          // lets see if there are initial
          // filters supplied by user
          if (this.isFilterable(col)
                  && typeof col.filterOptions.filterValue !== 'undefined'
                  && col.filterOptions.filterValue !== null) {
            this.$set(this.columnFilters, col.field, col.filterOptions.filterValue);
            // this.updateFilters(col, col.filterOptions.filterValue);
            // this.$set(col.filterOptions, 'filterValue', undefined);
          }
        }
        //* lets emit event once all filters are set
        this.$emit('filter-changed', this.columnFilters);
      },
    },
  };
</script>

<style scoped>

</style>