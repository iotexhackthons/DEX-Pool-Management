<template>
  <div id="warning" class="d-flex flex-items-center px-4 py-2 text-left">
    <Icon name="warning" size="22" class="mr-4" />
    <div>
      {{ $t('liquidityFail') }}
      <span v-text="$t('lowerAmounts')" class="link" @click="$emit('lower')" />.
    </div>
  </div>
</template>

<script>
export default {
  props: ['text']
};
</script>

<style scoped lang="scss">
@import '../../vars';

#warning {
  border: 1px solid $warning;
  border-radius: 4px;
  color: $warning;
}

.link {
  color: $blue;
  cursor: pointer;
  text-decoration: underline;
}
</style>
