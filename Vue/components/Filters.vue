<template>
    <div
      v-if="items.length"
      :class="['block overflow-hidden', { 'self-start p-4 shadow-pt bg-white': type === 'facet' }]"
    >
      <input
        v-if="type === 'facet'"
        :class="[
          'w-full py-2.5 px-3 text-center border rounded transition hover:opacity-80 font-bold text-sm bg-white',
          currentItem.length === 0
            ? 'text-gray border-gray border-opacity-60'
            : 'text-blue border-blue cursor-pointer'
        ]"
        type="button"
        :value="t('search.resetFilters')"
        @click="resetFilters"
      />
      <ul
        :class="[
          'flex',
          { 'flex-col items-start': type === 'view' },
          type === 'filter'
            ? 'flex-wrap px-5 py-2 rounded bg-white-dark'
            : { '-mx-1.5': !/facet|view/.test(type) },
          type === 'facet' ? 'flex-col divide-y divide-gray-soft' : 'overflow-auto hide-scroll'
        ]"
      >
        <li
          v-for="(item, index) in setItems"
          :key="item[titlePropName]"
          :class="[
            type === 'facet'
              ? { 'pb-6': index !== setItems.length - 1 }
              : [
                  { 'w-16 h-16 flex-none': item[iconPropName] },
                  type === 'view'
                    ? index !== setItems.length - 1
                      ? 'mb-5'
                      : ''
                    : 'm-1.5 px-3 py-2 text-center',
                  'cursor-pointer'
                ],
            {
              [`rounded-full ${
                currentFilters(item)
                  ? 'border border-blue bg-blue text-white'
                  : 'bg-gray-very-light text-gray'
              }`]: type === 'button'
            },
            {
              [`w-full flex-1 rounded border ${
                currentFilters(item)
                  ? 'border-gray bg-gray text-white'
                  : 'bg-gray-very-light text-gray'
              }`]: type === 'filter'
            },
            {
              [`${
                currentFilters(item)
                  ? type === 'tab'
                    ? 'border-b-2 text-blue border-blue'
                    : ' underline text-blue'
                  : `${type === 'tab' && 'border-b-2 border-transparent'} text-gray`
              }`]: /tab|view/.test(type)
            }
          ]"
          @[ev]="setCurrent(index)"
        >
          <Icon
            v-if="item[iconPropName]"
            size="1.5"
            :name="item[iconPropName]"
            :color="currentFilters(item) ? 'white' : 'gray'"
          />
          <span
            v-if="item[titlePropName]"
            :class="[
              'block font-semibold',
              { 'my-6': type === 'facet' },
              { 'whitespace-nowrap': type !== 'view' }
            ]"
            v-text="item[titlePropName]"
          />
          <template v-if="type === 'facet'">
            <ul class="flex flex-col space-y-3">
              <template v-for="(facet, i) in setterOrder(item)" :key="i">
                <li
                  v-if="i < maxFacets.added[index]"
                  class="flex space-x-2 text-sm text-gray-dark"
                  @click="setCurrent(facet)"
                >
                  <span
                    :class="[
                      'w-4 h-4 mt-0.5 flex items-center flex-none justify-center border rounded-sm cursor-pointer',
                      currentFilters(facet) ? 'border-blue bg-blue' : 'border-gray'
                    ]"
                  >
                    <Icon
                      v-if="currentFilters(facet)"
                      name="pt-check1"
                      class="font-bold transform translate-x-px"
                      color="white"
                    />
                  </span>
                  <div class="cursor-pointer">
                    <span
                      class="inline-block"
                      v-text="
                        facet[facet[displayedPropName] != null ? displayedPropName : facetPropName]
                      "
                    />
                    <span
                      v-if="facet.quantity"
                      class="inline-block ml-1"
                      v-text="`(${facet.quantity})`"
                    />
                  </div>
                </li>
              </template>
            </ul>
            <AlButton
              v-if="type === 'facet' && item[facetPropName].length > maxFacets.default"
              class="font-bold"
              type="empty"
              size-by-padding="pl-1 pt-5"
              :text="maxFacets.total[index] ? 'Ver menos' : 'Ver más'"
              :icon-name="`pt-angle-${maxFacets.total[index] ? 'up' : 'down'}`"
              icon-size="1.2"
              color="blue"
              @click="addFacets(item[facetPropName].length, index)"
            />
          </template>
        </li>
      </ul>
    </div>
  </template>
  <script>
  import { useI18n } from 'vue-i18n';
  import Icon from '@/components/Icon';
  import screen from '@/mixins/screen';
  import AlButton from '@/components/Button';
  import { toRefs, computed, reactive, watch } from 'vue';
  export default {
    name: 'AlFilters',
    components: { Icon, AlButton },
    props: {
      items: {
        type: Array,
        default: () => [],
        required: true
      },
      type: {
        type: String,
        default: 'filter',
        validator: val => /filter|button|tab|facet|view/.test(val)
      },
      multiple: {
        type: Boolean,
        default: false
      },
      init: {
        type: [Number, String],
        default: 0
      },
      titlePropName: {
        type: String,
        default: 'name',
        validator: val => val !== ''
      },
      facetPropName: {
        type: String,
        default: 'facet',
        validator: val => val !== ''
      },
      indexPropName: {
        type: String,
        default: 'index',
        validator: val => val !== ''
      },
      iconPropName: {
        type: String,
        default: 'icon',
        validator: val => val !== ''
      },
      displayedPropName: {
        type: String,
        default: 'displayed',
        validator: val => val !== ''
      },
      orderedPropName: {
        type: String,
        default: 'ordered',
        validator: val => val !== ''
      },
      current: {
        type: String,
        default: ''
      },
      ordered: {
        type: String,
        default: 'quantity',
        validator: val => /alphabetic|quantity/.test(val)
      },
      local: {
        type: Boolean,
        default: false
      }
    },
    emits: ['getFilters'],
    setup(props, { emit }) {
      const { t } = useI18n();
      const { sizes, scrollToSmoothly } = screen();
      const state = reactive({
        currentItem:
          props.type === 'facet' || props.multiple
            ? []
            : (!isNaN(props.init)
                ? props.items[props.init]
                : props.items.find(i => i[props.titlePropName] === props.init) ?? 0) ?? null,
        currentIndex:
          (!isNaN(props.init)
            ? props.init
            : props.items.findIndex(i => i[props.titlePropName] === props.init) ?? 0) ?? null,
        maxFacets: {
          default: 10,
          added: Array.from({ length: props.items.length }, () => 10),
          total: Array.from({ length: props.items.length }, () => false)
        }
      });
      const { currentItem, currentIndex, maxFacets } = toRefs(state);
      const setFacets = computed(() =>
        props.type === 'facet' && props.items.length
          ? props.items.reduce((acc, cur, key) => {
              if (
                cur[props.facetPropName].find(f => f != null && Array.isArray(f[props.facetPropName]))
              ) {
                const count = { [props.facetPropName]: {}, [props.displayedPropName]: {} };
                const setterArray = cur[props.facetPropName]
                  .reduce((prev, target) => {
                    const found = prev.find(i => i[props.facetPropName]);
                    if (!found) {
                      prev.push(target);
                    } else if (target[props.facetPropName] != null) {
                      found[props.facetPropName] = [
                        ...found[props.facetPropName],
                        ...target[props.facetPropName]
                      ];
                    }
                    return prev;
                  }, [])
                  .find(arr => arr[props.facetPropName]);
                Object.keys(setterArray).forEach(el => {
                  if (el === props.facetPropName) {
                    return setterArray[props.facetPropName].filter(Boolean).forEach(f => {
                      if (count[props.facetPropName][f]) count[props.facetPropName][f]++;
                      else count[props.facetPropName][f] = 1;
                      if (
                        setterArray[props.displayedPropName] &&
                        Array.isArray(setterArray[props.displayedPropName])
                      ) {
                        const displayed = setterArray[props.displayedPropName].find(
                          d => d[props.titlePropName] === f
                        );
                        if (displayed) {
                          count[props.displayedPropName][f] = displayed[props.displayedPropName];
                        }
                      }
                    });
                  }
                });
                cur[props.facetPropName] = Object.entries(count[props.facetPropName]).map(
                  ([k, v]) => ({
                    [props.facetPropName]: k,
                    ...(setterArray.quantity != null && { quantity: v }),
                    [props.displayedPropName]: count[props.displayedPropName][k]
                  })
                );
              }
              return [
                ...acc,
                ...cur[props.facetPropName]
                  .filter(({ [props.facetPropName]: item }) => item != null && item !== '')
                  .map(item => ({
                    ...cur,
                    key,
                    [props.titlePropName]: cur[props.titlePropName],
                    [props.facetPropName]: { ...item }
                  }))
              ].map((item, current) => ({
                ...item,
                [props.facetPropName]: {
                  identity: `${item[props.facetPropName][props.facetPropName]}_${key}`
                    .normalize('NFD')
                    .replace(/[\u0300-\u036f]/g, '')
                    .toLowerCase()
                    .replace(/\s/g, '_'),
                  ...item[props.facetPropName],
                  current
                }
              }));
            }, [])
          : []
      );
      const setItems = computed(() => {
        if (props.type === 'facet') {
          return setFacets.value.reduce((acc, cur) => {
            const found = acc.find(f => f.key === cur.key);
            return (
              found
                ? found[props.facetPropName].push(cur[props.facetPropName])
                : acc.push({ ...cur, [props.facetPropName]: [cur[props.facetPropName]] }),
              acc
            );
          }, []);
        } else if (props.multiple) {
          return props.items.map((item, i) => ({ ...item, current: i }));
        } else {
          return props.items.map((item, i) => ({ ...item, current: currentIndex.value === i }));
        }
      });
      const ev = computed(() => (props.type !== 'facet' ? 'click' : null));
      const setCurrent = index => {
        if (props.type === 'facet') {
          const indexFilter = currentItem.value.findIndex(el => el.identity === index.identity);
          if (indexFilter !== -1) {
            currentItem.value.splice(indexFilter, 1);
          } else {
            const setterFilter = {
              identity: setFacets.value[index.current][props.facetPropName].identity,
              [props.indexPropName]: setFacets.value[index.current][props.indexPropName],
              [props.facetPropName]:
                setFacets.value[index.current][props.facetPropName][props.facetPropName]
            };
            currentItem.value.push({
              query: props.local
                ? { [setterFilter[props.indexPropName]]: setterFilter[props.facetPropName] }
                : `${setterFilter[props.indexPropName]}='${setterFilter[props.facetPropName]}'`,
              identity: setterFilter.identity
            });
          }
          if (sizes.value.lg) scrollToSmoothly();
        } else if (props.multiple) {
          if (currentItem.value.find(item => item.current === index)) {
            currentItem.value.splice(
              currentItem.value.findIndex(item => item.current === index),
              1
            );
          } else {
            currentItem.value.push(setItems.value[index]);
          }
        } else {
          currentItem.value = props.items[index];
          currentIndex.value = index;
          if (props.current) emit('getFilters', currentItem.value);
        }
      };
      const currentFilters = item => {
        if (props.type === 'facet') {
          return currentItem.value.find(el => el.identity === item.identity);
        } else if (props.multiple) {
          return currentItem.value.find(el => el.current === item.current);
        } else if (props.current) {
          return item.key === props.current;
        } else {
          return item.current;
        }
      };
      const addFacets = (length, index) => {
        if (maxFacets.value.added[index] === length) {
          maxFacets.value.added[index] = maxFacets.value.default;
          maxFacets.value.total[index] = false;
        } else {
          maxFacets.value.added[index] = length;
          maxFacets.value.total[index] = true;
        }
      };
      const resetFilters = () => {
        if (currentItem.value.length) currentItem.value = [];
      };
      const setterOrder = data => {
        const facets = data[props.facetPropName];
        const ordered = data[props.orderedPropName];
        if (ordered !== false) {
          if (props.ordered === 'alphabetic' || ordered === 'alphabetic') {
            facets.sort((a, b) =>
              new Intl.Collator('es', { sensitivity: 'base' }).compare(
                a[a[props.displayedPropName] ? props.displayedPropName : props.facetPropName],
                b[b[props.displayedPropName] ? props.displayedPropName : props.facetPropName]
              )
            );
          } else if (props.ordered === 'quantity' || ordered === 'quantity') {
            const setter = currentItem.value.map(c => c.identity);
            facets.sort(
              (a, b) =>
                setter.indexOf(b.identity) - setter.indexOf(a.identity) || b.quantity - a.quantity
            );
          }
        }
        return facets.sort((a, b) => {
          const found = el => currentItem.value.some(facet => facet.identity === el.identity);
          return found(a) === found(b) ? 0 : found(a) ? -1 : 1;
        });
      };
      watch(
        currentItem,
        val => {
          if (!props.current)
            emit('getFilters', props.type === 'facet' ? val.map(({ query }) => query) : val);
        },
        {
          deep: true,
          ...(currentItem.value != null && {
            immediate: Array.isArray(currentItem.value)
              ? currentItem.value.length
              : Object.keys(currentItem.value).length
          })
        }
      );
      return {
        t,
        ev,
        setItems,
        addFacets,
        maxFacets,
        setCurrent,
        currentItem,
        setterOrder,
        resetFilters,
        currentFilters
      };
    }
  };
  </script>
  