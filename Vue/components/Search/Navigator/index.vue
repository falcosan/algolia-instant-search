<template>
  <div class="highlighted">
    <div
      v-if="
        navigator.paginator && (navigator.items.length || box.search || navigator.loader === true)
      "
      class="py-2.5"
    >
      <div class="flex flex-col xl:flex-row justify-between py-1.5">
        <div class="flex justify-center items-center space-x-1">
          <template v-if="navigator.items.length">
            <span class="pl-2.5 text-sm text-gray-dark" v-text="'Mostrando'" />
            <template v-if="+navigator.limit">
              <span class="font-bold text-sm text-gray-dark" v-text="setResultsIndication" />
              <span class="text-sm text-gray-dark" v-text="'de'" />
            </template>
            <span
              class="font-bold text-sm text-gray-dark"
              v-text="
                `${navigator.items.length} ${`resultado${navigator.items.length > 1 ? 's' : ''}`}`
              "
            />
          </template>
          <span v-else class="pl-2.5 text-sm font-bold" v-text="setResultsIndication" />
        </div>
        <div class="flex justify-center mt-1.5">
          <select
            v-if="+navigator.limit"
            v-model="setterShow.max"
            class="pl-10 py-0 border-0 border-transparent focus:border-transparent focus:ring-0 font-bold text-sm text-gray-dark"
            @change="changeShow"
          >
            <template v-for="i in setterShow.default" :key="i">
              <option
                v-if="setterShow.loop[i - 1] != null"
                :value="setterShow.loop[i - 1]"
                v-text="`${setterShow.loop[i - 1]} resultados por página`"
              />
            </template>
          </select>
        </div>
      </div>
      <hr class="block my-1 text-gray-soft" />
    </div>
    <div v-else-if="navigator.loader" class="flex justify-center mt-10">
      <Icon size="3" name="spinner" class="animate-spin" />
    </div>
    <slot v-if="navigator.items.length" :export-items="setItems" :export-page="setterShow" />
    <div
      v-if="navigator.items.length && +navigator.limit && navigator.paginator"
      class="flex items-center justify-center lg:justify-end pt-5 -m-1"
    >
      <span
        :class="[
          'w-8 h-8 flex flex-none items-center justify-center m-1 rounded-md shadow-md bg-white',
          setterShow.less
            ? 'cursor-pointer hover:bg-opacity-30 hover:bg-gray-very-light text-gray-dark'
            : 'opacity-50 text-gray'
        ]"
        @click="setterShow.less ? changePage('--') : null"
      >
        <Icon class="w-1" name="pt-angle-left" />
        <Icon name="pt-angle-left" />
      </span>
      <span
        :class="[
          'w-8 h-8 flex flex-none items-center justify-center m-1 rounded-md shadow-md bg-white',
          setterShow.less
            ? 'cursor-pointer hover:bg-opacity-30 hover:bg-gray-very-light text-gray-dark'
            : 'opacity-50 text-gray'
        ]"
        @click="setterShow.less ? changePage('-') : null"
      >
        <Icon name="pt-angle-left" size="1.2" />
      </span>
      <div class="flex flex-wrap justify-center">
        <template v-for="page in paginatorPages" :key="page">
          <span
            v-if="page >= maxPage - (paginatorShow - 1) && page <= maxPage"
            :class="[
              'h-8 flex items-center justify-center m-1 rounded-md px-3 shadow-md cursor-pointer font-semibold font-ubuntu text-base hover:underline',
              page === paginatorKey
                ? 'text-white hover:bg-blue-medium bg-blue'
                : 'text-gray hover:bg-opacity-30 hover:bg-gray-very-light bg-white'
            ]"
            @click="changePage(page)"
            v-text="page"
          />
        </template>
      </div>
      <span
        :class="[
          'w-8 h-8 flex flex-none items-center justify-center m-1 rounded-md shadow-md bg-white',
          setterShow.more
            ? 'cursor-pointer hover:bg-opacity-30 hover:bg-gray-very-light text-gray-dark'
            : 'opacity-50 text-gray'
        ]"
        @click="setterShow.more ? changePage('+') : null"
      >
        <Icon name="pt-angle-right" size="1.2" />
      </span>
      <span
        :class="[
          'w-8 h-8 flex flex-none items-center justify-center m-1 rounded-md shadow-md bg-white',
          setterShow.more
            ? 'cursor-pointer hover:bg-opacity-30 hover:bg-gray-very-light text-gray-dark'
            : 'opacity-50 text-gray'
        ]"
        @click="setterShow.more ? changePage('++') : null"
      >
        <Icon class="w-1" name="pt-angle-right" />
        <Icon name="pt-angle-right" />
      </span>
    </div>
  </div>
</template>
<script>
import { reactive, toRefs, computed, inject, watch } from 'vue';
import Icon from '@/components/Icon';
import { useI18n } from 'vue-i18n';
import screen from '@/mixins/screen';
export default {
  name: 'AlSearchNavigator',
  components: { Icon },
  setup() {
    const { t } = useI18n();
    const { sizes, scrollToSmoothly } = screen();
    const navigator = inject('navigator');
    const box = inject('box');
    const state = reactive({
      paginatorMax: +navigator.value.limit ? +navigator.value.limit : navigator.value.items.length,
      paginatorMin:
        +navigator.value.limit === 1
          ? +navigator.value.limit
          : Math.ceil(+navigator.value.limit / 2),
      paginatorIndex: 0,
      paginatorKey: 1,
      paginatorShow: 7,
      paginatorLimit: 3
    });
    const {
      paginatorIndex,
      paginatorMax,
      paginatorKey,
      paginatorShow,
      paginatorLimit,
      paginatorMin
    } = toRefs(state);
    const paginatorPages = computed(() =>
      +navigator.value.limit ? Math.ceil(navigator.value.items.length / paginatorMax.value) : 0
    );
    const setItems = computed(() =>
      paginatorPages.value
        ? navigator.value.items.slice(
            paginatorIndex.value,
            paginatorIndex.value + paginatorMax.value
          )
        : navigator.value.items
    );
    const maxPage = computed(() =>
      paginatorPages.value
        ? Math.min(
            Math.max(paginatorPages.value, 0),
            paginatorKey.value + (paginatorShow.value - 1)
          )
        : 1
    );
    const setResultsIndication = computed(() => {
      if (navigator.value.items.length) {
        return `${paginatorIndex.value === 0 ? '1' : paginatorIndex.value + 1} - ${
          paginatorMax.value * paginatorKey.value > navigator.value.items.length
            ? navigator.value.items.length
            : paginatorMax.value * paginatorKey.value
        }`;
      } else {
        return t('search.noResults');
      }
    });
    const changePage = operation => {
      if (
        operation === '+' &&
        paginatorIndex.value + paginatorMax.value < navigator.value.items.length
      ) {
        paginatorIndex.value += paginatorMax.value;
        paginatorKey.value++;
      } else if (operation === '++') {
        paginatorIndex.value = paginatorMax.value * paginatorPages.value - paginatorMax.value;
        paginatorKey.value = paginatorPages.value;
      } else if (operation === '-' && paginatorIndex.value) {
        paginatorIndex.value -= paginatorMax.value;
        paginatorKey.value--;
      } else if (operation === '--') {
        paginatorIndex.value = 0;
        paginatorKey.value = 1;
      } else if (!isNaN(operation)) {
        paginatorIndex.value = paginatorMax.value * operation - paginatorMax.value;
        paginatorKey.value = operation;
      }
      if (sizes.value.lg) scrollToSmoothly();
    };
    const setterShow = computed(() => {
      const setterLoop = [
        ...new Set(
          Array.from({ length: paginatorLimit.value }, (_, i) =>
            Math.min(box.value.maximum, Math.max(paginatorMin.value, +navigator.value.limit * i))
          )
        )
      ];
      return {
        default: Math.min(Math.max(paginatorLimit.value, paginatorMin.value), setterLoop.length),
        max: paginatorMax.value,
        index: paginatorIndex.value,
        loop: setterLoop,
        more: paginatorKey.value !== paginatorPages.value,
        less: paginatorKey.value > 1
      };
    });
    const changeShow = e => {
      paginatorMax.value = +e.target.value;
      if (paginatorIndex.value && paginatorKey.value !== 1 && paginatorPages.value) {
        paginatorIndex.value = 0;
        paginatorKey.value = 1;
      }
    };
    watch(
      () => [box.value.search, navigator.value.filters],
      val => {
        if (val && paginatorIndex.value && paginatorKey.value !== 1 && paginatorPages.value) {
          paginatorIndex.value = 0;
          paginatorKey.value = 1;
        }
      },
      { deep: true }
    );
    return {
      box,
      maxPage,
      setItems,
      navigator,
      changePage,
      setterShow,
      changeShow,
      paginatorMin,
      paginatorKey,
      paginatorMax,
      paginatorShow,
      paginatorPages,
      paginatorIndex,
      setResultsIndication
    };
  }
};
</script>
