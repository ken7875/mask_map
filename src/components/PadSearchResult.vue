<template>
  <div class="result d-flex">
    <template v-for="(item, key) in result">
      <a
        href="#"
        v-if="
          item.properties.county === choose.county &&
          item.properties.town === choose.town
        "
        :key="key"
        class="rounded-lg d-block p-4 mr-3"
        :class="{
          green: item.properties.mask_adult || item.properties.mask_child,
          gray:
            item.properties.mask_adult === false ||
            item.properties.mask_child === false,
        }"
        @click="updatePanTo(item)"
      >
        <h5 class="text-dark">{{ item.properties.name }}</h5>
        <p class="text-dark text-sm text-nowrap">電話: {{ item.properties.phone }}</p>
        <p class="bg-danger text-nowrap px-2 py-1 text-dark">
          成人口罩: {{ item.properties.mask_adult }}
        </p>
        <p class="bg-warning text-nowrap px-2 py-1 text-dark">
          兒童口罩: {{ item.properties.mask_child }}
        </p>
      </a>
    </template>
  </div>
</template>
<style lang="scss">
  .result{
    overflow-x: auto;
  }
</style>
<script>

export default {
  props: ['result', 'choose'],
  methods: {
    updatePanTo(item) {
      this.$emit('updatePlace', item);
    },
  },
};
</script>
