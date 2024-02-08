<template>
  <component :is="type" @[ev]="setSearch">
    <template
      v-if="type === 'navigator' && hasSlot('content')"
      #default="{ exportItems, exportPage }"
    >
      <slot name="content" :get-items="exportItems" :get-page="exportPage" />
    </template>
  </component>
</template>
<script>
import Box from './Box';
import slot from '@/mixins/slot';
import Navigator from './Navigator';
import { MeiliSearch } from 'meilisearch';
import { reactive, toRefs, computed, watch } from 'vue';
import { mapMeilisearchQuery } from '@/utils/helpers/object';
export default {
  name: 'AlSearch',
  components: { Box, Navigator },
  provide() {
    return {
      box: computed(() => ({
        local: this.local,
        focus: this.focus,
        search: this.search,
        category: this.category,
        placeholder: this.placeholder,
        maximum: this.setterMaximum(this)
      })),
      navigator: computed(() => ({
        limit: this.limit,
        filters: this.filters,
        paginator: this.paginator,
        processed: this.mixedWaiters,
        placeholder: this.placeholder,
        loader: this.loading ?? this.loader,
        items: this.items ?? this.storageItems
      }))
    };
  },
  props: {
    type: {
      type: String,
      default: 'box',
      validator: val => /box|navigator/.test(val)
    },
    source: {
      type: String,
      default: 'prisma',
      validator: val => /prisma|strapi/.test(val)
    },
    search: {
      type: String,
      default: ''
    },
    local: {
      type: Boolean,
      default: false
    },
    placeholder: {
      type: String,
      default: 'Buscador'
    },
    focus: {
      type: Boolean,
      default: false
    },
    formatted: {
      type: Boolean,
      default: false
    },
    processed: {
      type: Boolean,
      default: false
    },
    category: {
      type: String,
      default: ''
    },
    items: {
      type: Array,
      default: undefined
    },
    paginator: {
      type: Boolean,
      default: false
    },
    maximum: {
      type: [Boolean, Number, String],
      default: 999,
      validator: val => !!val === val || !isNaN(+val)
    },
    limit: {
      type: [Number, String],
      default: 6,
      validator: val => !isNaN(+val)
    },
    filters: {
      type: [Array, String, Boolean, Object],
      default: ''
    },
    highlighted: {
      type: [Boolean, Array, String],
      default: false
    },
    retrieved: {
      type: [Boolean, Array, String],
      default: false
    },
    loader: {
      type: [Boolean, typeof undefined],
      default: undefined
    }
  },
  emits: ['update:search', 'getItems'],
  setup(props, { emit }) {
    const { hasSlot } = slot();
    const state = reactive({
      loading: false,
      storageItems: [],
      processedItems: false,
      processedSearch: false
    });
    const { storageItems, processedItems, processedSearch, loading } = toRefs(state);
    const ev = computed(() => (props.type === 'box' ? 'setSearch' : null));
    const checkFilterType = computed(
      () =>
        typeof props.filters === 'object' && !Array.isArray(props.filters) && props.filters != null
    );
    const setterCategory = computed(() => {
      if (props.category) {
        return props.category.includes(':') ? props.category.split(':')[0] : props.category;
      } else {
        return false;
      }
    });
    const setterMaximum = (source = props) => {
      if (source.maximum === true) return 9999999;
      else if (source.maximum === false) return 20;
      else if (!isNaN(+source.maximum)) return +source.maximum;
      else return 999;
    };
    const getItems = async () => {
      if (setterCategory.value && !props.local) {
        processedSearch.value = false;
        loading.value = true;
        const client = new MeiliSearch({
          host: import.meta.env[`APP_MEILISEARCH_HOST_${props.source.toUpperCase()}`],
          apiKey: import.meta.env[`APP_MEILISEARCH_API_KEY_${props.source.toUpperCase()}`]
        }).index(setterCategory.value);
        return await client
          .search(props.search, {
            ...(!props.search && { limit: setterMaximum() }),
            ...(props.category.includes(':') && {
              sort: [props.category.substring(props.category.indexOf(':') + 1)]
            }),
            ...(checkFilterType.value &&
              processedItems.value &&
              /filter/.test(Object.keys(props.filters)) &&
              props.filters.filter.length && {
                filter: Array.isArray(props.filters.filter)
                  ? props.filters.filter
                  : [props.filters.filter]
              }),
            ...(checkFilterType.value &&
              processedItems.value &&
              /facets/.test(Object.keys(props.filters)) &&
              props.filters.facets.length && {
                facets: Array.isArray(props.filters.facets)
                  ? props.filters.facets
                  : [props.filters.facets]
              }),
            ...(props.highlighted &&
              props.search && {
                attributesToHighlight:
                  !!props.highlighted === props.highlighted
                    ? ['*']
                    : Array.isArray(props.highlighted)
                      ? props.highlighted
                      : [props.highlighted]
              }),
            ...(props.retrieved && {
              attributesToRetrieve:
                !!props.retrieved === props.retrieved
                  ? ['*']
                  : Array.isArray(props.retrieved)
                    ? props.retrieved.length
                      ? props.retrieved
                      : ['*']
                    : [props.retrieved]
            })
          })
          .then(({ hits }) => {
            const result = mapMeilisearchQuery(hits);
            storageItems.value = result;
            emit(
              'getItems',
              props.formatted
                ? props.highlighted && props.search
                  ? result.map(hit => hit._formatted)
                  : result
                : result
            );
            processedItems.value = true;
          })
          .finally(() => {
            processedSearch.value = true;
            loading.value = false;
          });
      }
      if (props.local) {
        storageItems.value = props.items.filter(hits =>
          hits.source.toLowerCase().includes(props.search.toLowerCase())
        );
        emit('getItems', storageItems.value);
      }
    };
    const mixedWaiters = computed(() => processedSearch.value && !props.processed);
    const setSearch = search => emit('update:search', search);
    const watcherFilter = computed(
      () => /filter/.test(Object.keys(props.filters)) && props.filters.filter
    );
    watch(() => [watcherFilter.value, props.search], getItems, { deep: true, immediate: true });
    return {
      ev,
      loading,
      hasSlot,
      getItems,
      setSearch,
      storageItems,
      mixedWaiters,
      setterMaximum
    };
  }
};
</script>
