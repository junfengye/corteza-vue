<template>
  <b-card
    no-body
    class="shadow-sm"
    header-bg-variant="white"
  >
    <template #header>
      <b-container
        class="p-0"
        fluid
        align-v="center"
      >
        <b-row>
          <b-col
            lg="9"
          >
            <slot
              name="header"
              :selected="selected"
            />
          </b-col>
          <b-col
            class="d-flex align-items-center mt-1 mt-lg-0"
          >
            <b-input-group
              class="h-100"
            >
              <b-form-input
                v-model.trim="filter.query"
                :placeholder="$t('search.placeholder')"
                class="border-light border-right-0 pr-0 h-100"
                @keyup="$emit('search', $event)"
              />
              <b-input-group-append>
                <b-input-group-text
                  class="text-primary bg-white border-left-0"
                >
                  <font-awesome-icon
                    :icon="['fas', 'search']"
                  />
                </b-input-group-text>
              </b-input-group-append>
            </b-input-group>
          </b-col>
        </b-row>
      </b-container>
    </template>

    <b-card-body
      class="p-0"
    >
      <b-table
        id="resource-list"
        ref="resourceList"
        hover
        responsive
        head-variant="light"
        class="mb-0"
        :primary-key="primaryKey"
        :sort-by.sync="sorting.sortBy"
        :sort-desc.sync="sorting.sortDesc"
        :items="_items"
        :fields="_fields"
        show-empty
        no-sort-reset
        :tbody-tr-class="{ 'pointer': clickable }"
        no-local-sorting
        @row-clicked="$emit('row-clicked', $event)"
      >
        <template #empty>
          <h5
            class="font-weight-normal text-center"
            style="margin-top: 19vh;"
          >
            {{ $t('noItems') }}
          </h5>
        </template>

        <template #table-busy>
          <div
            class="text-center"
            style="margin-top: 10vh;"
          >
            <b-spinner
              style="width: 5em; height: 5em;"
            />
            <h5 class="mt-2">
              {{ $t('loading') }}
            </h5>
          </div>
        </template>

        <template #head(select)>
          <b-checkbox
            :disabled="disableSelectAll"
            :checked="allRowsSelected && !disableSelectAll"
            class="ml-1"
            @change="selectAllRows"
          />
        </template>

        <template #cell(select)="{ item }">
          <b-form-checkbox
            v-if="isItemSelectable(item)"
            class="ml-1"
            :checked="selected.includes(item[primaryKey])"
            @change="onSelectRow($event, item[primaryKey])"
          />
        </template>

        <!-- Magic; Make slots if parent provides them -->
        <template
          v-for="field in customFieldSlots"
          #[`cell(${field})`]="{ item }"
        >
          <slot
            :name="field"
            :item="item"
          />
        </template>
      </b-table>
    </b-card-body>

    <template #footer>
      <div
        class="d-flex align-items-center justify-content-between"
      >
        <div class="text-truncate">
          <div
            class="text-nowrap font-weight-bold"
          >
            <span v-if="pagination.count > 1">
              {{ $t('pagination.showing', getPagination) }}
            </span>
            <span
              v-else
            >
              {{ $t('pagination.single', getPagination) }}
            </span>
          </div>
        </div>

        <b-button-group>
          <b-button
            :disabled="hasPrevPage"
            variant="link"
            class="text-dark"
            @click="goToPage()"
          >
            <font-awesome-icon :icon="['fas', 'angle-double-left']" />
          </b-button>
          <b-button
            :disabled="hasPrevPage"
            variant="link"
            class="text-dark"
            @click="goToPage('prevPage')"
          >
            <font-awesome-icon :icon="['fas', 'angle-left']" />
            {{ $t('pagination.prev') }}
          </b-button>
          <b-button
            :disabled="hasNextPage"
            variant="link"
            class="text-dark"
            @click="goToPage('nextPage')"
          >
            {{ $t('pagination.next') }}
            <font-awesome-icon :icon="['fas', 'angle-right']" />
          </b-button>
        </b-button-group>
      </div>
    </template>
  </b-card>
</template>
<script>
export default {
  props: {
    primaryKey: {
      type: String,
      required: true,
    },

    filter: {
      type: Object,
      required: true,
    },

    sorting: {
      type: Object,
      required: true,
    },

    pagination: {
      type: Object,
      required: true,
    },

    fields: {
      type: Array,
      required: true,
    },

    // Promise that resolves with an array
    items: {
      type: Function,
      required: true,
    },

    hideSearch: {
      type: Boolean,
    },

    // Are rows clickable
    clickable: {
      type: Boolean,
      default: false,
    },

    selectable: {
      type: Boolean,
      default: false,
    },

    isItemSelectable: {
      type: Function,
      default: () => { true },
    },
  },

  data () {
    return {
      selected: [],
      selectableItemIDs: [],
    }
  },

  computed: {
    _fields () {
      return [
        {
          key: 'select',
          label: '',
          thStyle: 'width: 0; white-space: nowrap;',
        },
        ...this.fields,
      ].map(f => {
        return { ...f, thClass: `${f.thClass || 'border-0'}` }
      })
    },

    customFieldSlots () {
      return [
        ...Object.keys(this.$slots),
        ...Object.keys(this.$scopedSlots),
      ].filter(s => s !== 'header')
    },

    disableSelectAll () {
      return !this.selectableItemIDs.length
    },

    allRowsSelected () {
      return this.selected.length === this.selectableItemIDs.length
    },

    getPagination () {
      const { page = 1, count = 0, limit = 10 } = this.pagination

      return {
        from: ((page - 1) * limit) + 1,
        to: Math.min((page * limit), count),
        count,
      }
    },

    hasPrevPage () {
      return !this.pagination.prevPage
    },

    hasNextPage () {
      return !this.pagination.nextPage
    },
  },

  methods: {
    _items () {
      this.selected = []
      this.selectableItemIDs = []

      return this.items().then((items = []) => {
        this.selectableItemIDs = items.filter(isItemSelectable).map(i => i[this.primaryKey])
        return items
      })
    },

    onSelectRow (selected, itemID) {
      if (selected) {
        if (this.selected.includes(itemID)) {
          return
        }

        this.selected.push(itemID)
      } else {
        const i = this.selected.indexOf(itemID)
        if (i < 0) {
          return
        }

        this.selected.splice(i, 1)
      }
    },

    selectAllRows (allSelected = false) {
      if (allSelected) {
        this.selected = this.selectableItemIDs
      } else {
        this.selected = []
      }
    },

    goToPage (page) {
      const pageCursor = this.pagination[page] || ''
      this.$router.push({ path: this.$route.path, query: { ...this.$route.query, pageCursor } })
    },
  },
}
</script>
