<template>
  <div id="warning" class="d-flex flex-items-center p-4 text-left">
    <Icon name="warning" size="22" class="mr-4" />
    {{ text }}
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
</style>
