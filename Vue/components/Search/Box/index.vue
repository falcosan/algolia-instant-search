<template>
  <div>
    <div class="relative w-full flex items-center rounded border overflow-hidden border-gray-soft">
      <Icon class="py-3 pl-3" size="1.2" name="pt-search" />
      <input
        v-appear="box.focus"
        class="appearance-none block w-full border-none focus:outline-none focus:ring-0 focus:shadow-none bg-white text-gray-dark"
        :placeholder="box.placeholder"
        type="text"
        :value="box.search"
        @input="toSearch"
      />
    </div>
  </div>
</template>
<script>
import { inject } from 'vue';
import { useI18n } from 'vue-i18n';
import Icon from '@/components/Icon';
export default {
  name: 'AlSearchBox',
  directives: {
    appear: {
      mounted(el, binding) {
        if (binding.value) el.focus();
      }
    }
  },
  components: { Icon },
  emits: ['setSearch'],
  setup(_, { emit }) {
    const box = inject('box');
    const { t } = useI18n();
    const toSearch = e => {
      emit('setSearch', e.target.value);
    };
    return {
      t,
      box,
      toSearch
    };
  }
};
</script>
<style scoped>
::placeholder {
  @apply text-gray-light;
}
</style>
