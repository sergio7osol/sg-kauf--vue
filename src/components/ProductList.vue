<template>
  <div class="col-11 offset-1 product-list pt-3 pl-3">
      <table class="table table-striped product-list__items mt-3">
        <thead class="product product--default">
          <product :product="emptyProduct" :isDefault="true" @retrigger-save-product="retriggerSaveProduct" :key="emptyProduct.measure + Date.now()" /> <!-- :key added, so that the product component always rerenders -->
        </thead>
        <tbody class="product">
          <product 
            v-for="(product, i) in buyProducts" 
            :isDefault="false" 
            :product="product"
            :index="i"
            :key="product.name + Date.now() + i" 
            @remove-product="retriggerSendProductToRemove"
          /> 
        </tbody>
      </table>
  </div>
</template>

<script>
import Product from './Product.vue';
 
export default {
  name: 'product-list',
  data() {
    return {
      emptyProduct: {
        name: null,
        description: "",
        weightAmount: null,
        measure: "piece",
        price: null,
        discount: null
      }
    }
  },
  props: {
    buyProducts: Array
  },
  emits: ['retrigger-save-product', 'remove-product'],
  methods: {
    retriggerSaveProduct(product) {
        this.$emit('retrigger-save-product', product);
    },
    retriggerSendProductToRemove(product) {
        console.log('product-list product: ', product);
        this.$emit('remove-product', product);
    }
  },
  components: {
    Product
  }
};
</script>

<style scoped lang="scss">    
</style>
