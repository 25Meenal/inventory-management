<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
    </div>

    <div class="budget-section">
      <h3>{{ t('restocking.budget.title') }}</h3>
      <div class="budget-slider">
        <input
          type="range"
          min="0"
          max="100000"
          step="1000"
          v-model="budget"
          @input="updateRecommendations"
        />
        <div class="budget-display">
          {{ t('restocking.budget.current') }}: {{ formatCurrency(budget, selectedCurrency) }}
        </div>
      </div>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="recommendations-section">
        <h3>{{ t('restocking.recommendations.title') }}</h3>
        <div v-if="recommendations.length === 0" class="no-recommendations">
          {{ t('restocking.recommendations.none') }}
        </div>
        <div v-else class="recommendations-list">
          <div v-for="item in recommendations" :key="item.item_sku" class="recommendation-item">
            <div class="item-details">
              <h4>{{ item.item_name }}</h4>
              <p>{{ t('restocking.recommendations.sku') }}: {{ item.item_sku }}</p>
              <p>{{ t('restocking.recommendations.quantity') }}: {{ item.quantity_to_order }}</p>
              <p>{{ t('restocking.recommendations.unitCost') }}: {{ formatCurrency(item.unit_cost, selectedCurrency) }}</p>
              <p class="total-cost">{{ t('restocking.recommendations.totalCost') }}: {{ formatCurrency(item.total_cost, selectedCurrency) }}</p>
            </div>
          </div>
          <div class="order-summary">
            <h4>{{ t('restocking.order.totalItems') }}: {{ recommendations.length }}</h4>
            <h4>{{ t('restocking.order.totalCost') }}: {{ formatCurrency(totalCost, selectedCurrency) }}</h4>
            <button
              @click="placeOrder"
              :disabled="placingOrder"
              class="place-order-btn"
            >
              {{ placingOrder ? t('common.loading') : t('restocking.order.place') }}
            </button>
          </div>
        </div>
      </div>
    </div>

    <div v-if="successMessage" class="success-message">
      {{ successMessage }}
    </div>
  </div>
</template>

<script>
import { ref, onMounted, computed, watch } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'
import { formatCurrency } from '../utils/currency'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()
    const loading = ref(false)
    const error = ref(null)
    const budget = ref(50000) // Default budget
    const recommendations = ref([])
    const placingOrder = ref(false)
    const successMessage = ref('')

    const selectedCurrency = computed(() => currentCurrency.value)

    const totalCost = computed(() => {
      return recommendations.value.reduce((sum, item) => sum + item.total_cost, 0)
    })

    const updateRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        const response = await api.getRestockingRecommendations(budget.value)
        recommendations.value = response
      } catch (err) {
        error.value = err.message || 'Failed to load recommendations'
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      placingOrder.value = true
      error.value = null
      try {
        const items = recommendations.value.map(item => ({
          sku: item.item_sku,
          name: item.item_name,
          quantity: item.quantity_to_order,
          unit_cost: item.unit_cost,
          total_cost: item.total_cost
        }))
        await api.placeRestockingOrder(items)
        successMessage.value = t('restocking.order.success')
        // Clear recommendations after placing order
        recommendations.value = []
        budget.value = 50000 // Reset budget
      } catch (err) {
        error.value = err.message || 'Failed to place order'
      } finally {
        placingOrder.value = false
      }
    }

    onMounted(() => {
      updateRecommendations()
    })

    return {
      t,
      loading,
      error,
      budget,
      recommendations,
      placingOrder,
      successMessage,
      selectedCurrency,
      totalCost,
      updateRecommendations,
      placeOrder,
      formatCurrency
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 20px;
}

.page-header h2 {
  margin-bottom: 20px;
}

.budget-section {
  margin-bottom: 30px;
}

.budget-slider {
  display: flex;
  align-items: center;
  gap: 20px;
}

.budget-slider input[type="range"] {
  flex: 1;
  max-width: 400px;
}

.budget-display {
  font-weight: bold;
  min-width: 150px;
}

.recommendations-section {
  margin-top: 30px;
}

.recommendations-list {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 20px;
}

.recommendation-item {
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 15px;
  background: #f9f9f9;
}

.item-details h4 {
  margin: 0 0 10px 0;
}

.item-details p {
  margin: 5px 0;
}

.total-cost {
  font-weight: bold;
  color: #0f172a;
}

.order-summary {
  grid-column: 1 / -1;
  border-top: 2px solid #64748b;
  padding-top: 20px;
  text-align: center;
}

.place-order-btn {
  background: #10b981;
  color: white;
  border: none;
  padding: 10px 20px;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  margin-top: 10px;
}

.place-order-btn:disabled {
  background: #ccc;
  cursor: not-allowed;
}

.success-message {
  margin-top: 20px;
  padding: 10px;
  background: #d1fae5;
  color: #065f46;
  border-radius: 4px;
  text-align: center;
}

.loading, .error {
  text-align: center;
  padding: 20px;
}

.error {
  color: #ef4444;
}

.no-recommendations {
  text-align: center;
  padding: 40px;
  color: #64748b;
}
</style>