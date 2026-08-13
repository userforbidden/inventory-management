<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <!-- Budget Slider Section -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">{{ t('restocking.budgetSection') }}</h3>
      </div>
      <div class="card-content">
        <div class="budget-container">
          <div class="budget-input-group">
            <label for="budget-slider" class="budget-label">
              {{ t('restocking.budgetAllocation') }}
            </label>
            <div class="slider-wrapper">
              <input
                id="budget-slider"
                v-model.number="budget"
                type="range"
                :min="0"
                :max="100000"
                :step="1000"
                class="budget-slider"
              />
              <div class="slider-track-value">
                {{ currencySymbol }}{{ (budget / 1000).toFixed(0) }}k
              </div>
            </div>
            <div class="budget-display">
              <span class="budget-label-text">{{ t('restocking.selectedBudget') }}:</span>
              <span class="budget-amount">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
            </div>
          </div>

          <button
            @click="fetchRecommendations"
            :disabled="budget <= 0 || loadingRecommendations"
            class="btn btn-primary"
          >
            <span v-if="!loadingRecommendations">{{ t('restocking.getRecommendations') }}</span>
            <span v-else>{{ t('common.loading') }}...</span>
          </button>
        </div>

        <!-- Error message for recommendations -->
        <div v-if="recommendationError" class="error-message">
          {{ recommendationError }}
        </div>
      </div>
    </div>

    <!-- Recommendations Table Section -->
    <div v-if="recommendations.length > 0" class="card">
      <div class="card-header">
        <h3 class="card-title">{{ t('restocking.recommendations') }}</h3>
        <div class="recommendation-stats">
          <div class="stat-item">
            <span class="stat-label">{{ t('restocking.totalCost') }}:</span>
            <span class="stat-value">{{ currencySymbol }}{{ totalRecommendationCost.toLocaleString() }}</span>
          </div>
          <div class="stat-item">
            <span class="stat-label">{{ t('restocking.budgetRemaining') }}:</span>
            <span class="stat-value" :class="budgetRemaining < 0 ? 'over-budget' : ''">
              {{ currencySymbol }}{{ budgetRemaining.toLocaleString() }}
            </span>
          </div>
        </div>
      </div>

      <div class="table-container">
        <table>
          <thead>
            <tr>
              <th class="col-checkbox">
                <input
                  v-model="selectAllRecommendations"
                  type="checkbox"
                  class="checkbox"
                />
              </th>
              <th>{{ t('restocking.table.productName') }}</th>
              <th>{{ t('restocking.table.sku') }}</th>
              <th class="col-numeric">{{ t('restocking.table.quantity') }}</th>
              <th class="col-numeric">{{ t('restocking.table.unitCost') }}</th>
              <th class="col-numeric">{{ t('restocking.table.totalCost') }}</th>
              <th class="col-numeric">{{ t('restocking.table.demandForecast') }}</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(item, index) in recommendations" :key="index">
              <td class="col-checkbox">
                <input
                  v-model="selectedRecommendations"
                  :value="index"
                  type="checkbox"
                  class="checkbox"
                />
              </td>
              <td><strong>{{ translateProductName(item.product_name) }}</strong></td>
              <td>{{ item.sku }}</td>
              <td class="col-numeric">{{ item.quantity }}</td>
              <td class="col-numeric">{{ currencySymbol }}{{ item.unit_cost.toFixed(2) }}</td>
              <td class="col-numeric"><strong>{{ currencySymbol }}{{ (item.quantity * item.unit_cost).toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</strong></td>
              <td class="col-numeric">{{ item.demand_forecast || 'N/A' }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <!-- Place Order Section -->
    <div v-if="recommendations.length > 0" class="card">
      <div class="card-header">
        <h3 class="card-title">{{ t('restocking.orderSummary') }}</h3>
      </div>
      <div class="card-content">
        <div class="order-summary">
          <div class="summary-row">
            <span class="summary-label">{{ t('restocking.itemsToOrder') }}:</span>
            <span class="summary-value">{{ selectedRecommendations.length }}</span>
          </div>
          <div class="summary-row">
            <span class="summary-label">{{ t('restocking.orderTotal') }}:</span>
            <span class="summary-value">{{ currencySymbol }}{{ selectedOrderCost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</span>
          </div>
          <div class="summary-row">
            <span class="summary-label">{{ t('restocking.expectedDelivery') }}:</span>
            <span class="summary-value">{{ expectedDeliveryDate }}</span>
          </div>
        </div>

        <button
          @click="submitOrder"
          :disabled="selectedRecommendations.length === 0 || submittingOrder || selectedOrderCost > budget"
          class="btn btn-success"
        >
          <span v-if="!submittingOrder">{{ t('restocking.placeOrder') }}</span>
          <span v-else>{{ t('common.loading') }}...</span>
        </button>

        <!-- Error message for order submission -->
        <div v-if="orderError" class="error-message">
          {{ orderError }}
        </div>

        <!-- Budget exceeded warning -->
        <div v-if="selectedOrderCost > budget" class="warning-message">
          {{ t('restocking.budgetExceeded') }}
        </div>
      </div>
    </div>

    <!-- Toast Notification -->
    <transition name="slide-up">
      <div v-if="showNotification" :class="['toast', notificationType]">
        <div class="toast-content">
          <span class="toast-message">{{ notificationMessage }}</span>
          <button @click="showNotification = false" class="toast-close">
            <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor">
              <path fill-rule="evenodd" d="M4.293 4.293a1 1 0 011.414 0L10 8.586l4.293-4.293a1 1 0 111.414 1.414L11.414 10l4.293 4.293a1 1 0 01-1.414 1.414L10 11.414l-4.293 4.293a1 1 0 01-1.414-1.414L8.586 10 4.293 5.707a1 1 0 010-1.414z" clip-rule="evenodd" />
            </svg>
          </button>
        </div>
      </div>
    </transition>
  </div>
</template>

<script>
import { ref, onMounted, watch, computed } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency, translateProductName } = useI18n()

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    // Budget state
    const budget = ref(10000)

    // Recommendations state
    const recommendations = ref([])
    const selectedRecommendations = ref([])
    const loadingRecommendations = ref(false)
    const recommendationError = ref(null)

    // Order submission state
    const submittingOrder = ref(false)
    const orderError = ref(null)

    // Toast notification state
    const showNotification = ref(false)
    const notificationMessage = ref('')
    const notificationType = ref('success')

    // Use shared filters
    const { getCurrentFilters } = useFilters()

    // Computed: currency display
    const currencyDisplay = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    // Computed: total cost of all recommendations
    const totalRecommendationCost = computed(() => {
      return recommendations.value.reduce((total, item) => {
        return total + (item.quantity * item.unit_cost)
      }, 0)
    })

    // Computed: budget remaining
    const budgetRemaining = computed(() => {
      return budget.value - totalRecommendationCost.value
    })

    // Computed: total cost of selected items
    const selectedOrderCost = computed(() => {
      return selectedRecommendations.value.reduce((total, index) => {
        const item = recommendations.value[index]
        return total + (item.quantity * item.unit_cost)
      }, 0)
    })

    // Computed: expected delivery date (7 days from today)
    const expectedDeliveryDate = computed(() => {
      const today = new Date()
      const deliveryDate = new Date(today.getTime() + 7 * 24 * 60 * 60 * 1000)
      const locale = currentLocale.value === 'ja' ? 'ja-JP' : 'en-US'
      return deliveryDate.toLocaleDateString(locale, {
        year: 'numeric',
        month: 'short',
        day: 'numeric'
      })
    })

    // Get current locale for date formatting
    const { currentLocale } = useI18n()

    // Computed: select all handler
    const selectAllRecommendations = computed({
      get() {
        return (
          recommendations.value.length > 0 &&
          selectedRecommendations.value.length === recommendations.value.length
        )
      },
      set(value) {
        if (value) {
          selectedRecommendations.value = recommendations.value.map((_, index) => index)
        } else {
          selectedRecommendations.value = []
        }
      }
    })

    /**
     * Fetch restocking recommendations based on budget
     */
    const fetchRecommendations = async () => {
      if (budget.value <= 0) {
        showToast(t('restocking.budgetError'), 'error')
        return
      }

      try {
        loadingRecommendations.value = true
        recommendationError.value = null
        recommendations.value = await api.getRestockingRecommendations(budget.value)
        selectedRecommendations.value = []

        if (recommendations.value.length === 0) {
          showToast(t('restocking.noRecommendations'), 'info')
        } else {
          showToast(
            t('restocking.recommendationsLoaded', { count: recommendations.value.length }),
            'success'
          )
        }
      } catch (err) {
        recommendationError.value = t('restocking.fetchError') + ': ' + err.message
        showToast(t('restocking.fetchError'), 'error')
      } finally {
        loadingRecommendations.value = false
      }
    }

    /**
     * Submit the restocking order with selected items
     */
    const submitOrder = async () => {
      if (selectedRecommendations.value.length === 0) {
        showToast(t('restocking.selectItems'), 'error')
        return
      }

      if (selectedOrderCost.value > budget.value) {
        showToast(t('restocking.budgetExceeded'), 'error')
        return
      }

      try {
        submittingOrder.value = true
        orderError.value = null

        // Prepare items for submission
        const itemsToOrder = selectedRecommendations.value.map(index => {
          const item = recommendations.value[index]
          return {
            product_name: item.product_name,
            sku: item.sku,
            quantity: item.quantity,
            unit_cost: item.unit_cost
          }
        })

        const response = await api.submitRestockingOrder(itemsToOrder)

        // Show success notification
        showToast(
          t('restocking.orderPlaced', { orderId: response.id || 'N/A' }),
          'success'
        )

        // Clear selections and reset
        selectedRecommendations.value = []
        recommendations.value = []
        budget.value = 10000
      } catch (err) {
        orderError.value = t('restocking.submitError') + ': ' + err.message
        showToast(t('restocking.submitError'), 'error')
      } finally {
        submittingOrder.value = false
      }
    }

    /**
     * Show toast notification with auto-hide
     */
    const showToast = (message, type = 'success') => {
      notificationMessage.value = message
      notificationType.value = type
      showNotification.value = true

      // Auto-hide after 4 seconds
      setTimeout(() => {
        showNotification.value = false
      }, 4000)
    }

    // Initialize component
    onMounted(() => {
      // Component is ready
    })

    return {
      t,
      budget,
      currencySymbol,
      currencyDisplay,
      recommendations,
      selectedRecommendations,
      selectAllRecommendations,
      loadingRecommendations,
      recommendationError,
      submittingOrder,
      orderError,
      showNotification,
      notificationMessage,
      notificationType,
      totalRecommendationCost,
      budgetRemaining,
      selectedOrderCost,
      expectedDeliveryDate,
      fetchRecommendations,
      submitOrder,
      showToast,
      translateProductName
    }
  }
}
</script>

<style scoped>
/* Page Layout */
.restocking {
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 1.5rem;
}

.page-header {
  margin-bottom: 1.5rem;
}

.page-header h2 {
  margin-bottom: 0.25rem;
  color: #0f172a;
  font-size: 1.875rem;
  font-weight: 700;
}

.page-header p {
  color: #64748b;
  font-size: 0.875rem;
}

/* Card Styling */
.card {
  background: #ffffff;
  border: 1px solid #e2e8f0;
  border-radius: 12px;
  margin-bottom: 1.5rem;
  box-shadow: 0 1px 2px 0 rgba(0, 0, 0, 0.02);
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 1.5rem;
  padding: 1.25rem 1.5rem;
  border-bottom: 1px solid #e2e8f0;
}

.card-title {
  font-size: 1rem;
  font-weight: 600;
  color: #0f172a;
  margin: 0;
}

.card-content {
  padding: 1.5rem;
}

/* Budget Section */
.budget-container {
  display: flex;
  align-items: flex-end;
  gap: 2rem;
}

.budget-input-group {
  flex: 1;
}

.budget-label {
  display: block;
  font-size: 0.875rem;
  font-weight: 600;
  color: #0f172a;
  margin-bottom: 0.75rem;
}

.slider-wrapper {
  position: relative;
  margin-bottom: 0.75rem;
}

.budget-slider {
  width: 100%;
  height: 8px;
  border-radius: 4px;
  background: #e2e8f0;
  outline: none;
  -webkit-appearance: none;
  appearance: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #3b82f6;
  cursor: pointer;
  box-shadow: 0 2px 4px rgba(59, 130, 246, 0.4);
  transition: all 0.2s;
}

.budget-slider::-webkit-slider-thumb:hover {
  background: #2563eb;
  box-shadow: 0 4px 8px rgba(59, 130, 246, 0.5);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #3b82f6;
  cursor: pointer;
  border: none;
  box-shadow: 0 2px 4px rgba(59, 130, 246, 0.4);
  transition: all 0.2s;
}

.budget-slider::-moz-range-thumb:hover {
  background: #2563eb;
  box-shadow: 0 4px 8px rgba(59, 130, 246, 0.5);
}

.slider-track-value {
  position: absolute;
  right: 0;
  top: -1.75rem;
  font-size: 0.875rem;
  font-weight: 600;
  color: #3b82f6;
  background: #eff6ff;
  padding: 0.25rem 0.75rem;
  border-radius: 6px;
  border: 1px solid #bfdbfe;
}

.budget-display {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.budget-label-text {
  font-size: 0.875rem;
  color: #64748b;
  font-weight: 500;
}

.budget-amount {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
}

/* Buttons */
.btn {
  padding: 0.75rem 1.5rem;
  font-size: 0.875rem;
  font-weight: 600;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-primary {
  background: #3b82f6;
  color: white;
}

.btn-primary:hover:not(:disabled) {
  background: #2563eb;
  box-shadow: 0 4px 12px rgba(59, 130, 246, 0.3);
}

.btn-success {
  background: #10b981;
  color: white;
  width: 100%;
  margin-top: 1rem;
}

.btn-success:hover:not(:disabled) {
  background: #059669;
  box-shadow: 0 4px 12px rgba(16, 185, 129, 0.3);
}

/* Recommendation Stats */
.recommendation-stats {
  display: flex;
  gap: 2rem;
}

.stat-item {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.stat-label {
  font-size: 0.875rem;
  color: #64748b;
  font-weight: 500;
}

.stat-value {
  font-size: 0.875rem;
  font-weight: 700;
  color: #0f172a;
}

.stat-value.over-budget {
  color: #ef4444;
}

/* Table Styling */
.table-container {
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
}

thead {
  background: #f8fafc;
  border-bottom: 2px solid #e2e8f0;
}

th {
  padding: 0.75rem 1rem;
  text-align: left;
  font-size: 0.75rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #475569;
}

th.col-checkbox {
  width: 3rem;
  text-align: center;
}

th.col-numeric {
  text-align: right;
}

tbody tr {
  border-bottom: 1px solid #e2e8f0;
  transition: background-color 0.15s ease;
}

tbody tr:hover {
  background: #f8fafc;
}

td {
  padding: 0.75rem 1rem;
  font-size: 0.875rem;
  color: #1e293b;
}

td.col-checkbox {
  text-align: center;
}

td.col-numeric {
  text-align: right;
  font-family: 'Monaco', 'Courier New', monospace;
}

/* Checkbox Styling */
.checkbox {
  width: 18px;
  height: 18px;
  cursor: pointer;
  accent-color: #3b82f6;
}

/* Order Summary */
.order-summary {
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 1.5rem;
  margin-bottom: 1.5rem;
}

.summary-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.75rem 0;
}

.summary-row:not(:last-child) {
  border-bottom: 1px solid #cbd5e1;
}

.summary-label {
  font-size: 0.875rem;
  font-weight: 500;
  color: #64748b;
}

.summary-value {
  font-size: 0.875rem;
  font-weight: 700;
  color: #0f172a;
}

/* Error and Warning Messages */
.error-message {
  background: #fee2e2;
  border: 1px solid #fecaca;
  border-radius: 8px;
  padding: 0.75rem 1rem;
  margin-top: 1rem;
  color: #991b1b;
  font-size: 0.875rem;
}

.warning-message {
  background: #fef3c7;
  border: 1px solid #fcd34d;
  border-radius: 8px;
  padding: 0.75rem 1rem;
  margin-top: 1rem;
  color: #92400e;
  font-size: 0.875rem;
}

/* Toast Notification */
.toast {
  position: fixed;
  bottom: 1.5rem;
  right: 1.5rem;
  max-width: 400px;
  padding: 1rem 1.5rem;
  border-radius: 8px;
  box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1);
  display: flex;
  align-items: center;
  gap: 1rem;
  z-index: 1000;
  animation: slideUp 0.3s ease-out;
}

.toast.success {
  background: #d1fae5;
  border: 1px solid #a7f3d0;
  color: #065f46;
}

.toast.error {
  background: #fee2e2;
  border: 1px solid #fecaca;
  color: #991b1b;
}

.toast.info {
  background: #dbeafe;
  border: 1px solid #bfdbfe;
  color: #0c2340;
}

.toast-content {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  flex: 1;
}

.toast-message {
  font-size: 0.875rem;
  font-weight: 500;
}

.toast-close {
  background: transparent;
  border: none;
  cursor: pointer;
  padding: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 20px;
  height: 20px;
  opacity: 0.7;
  transition: opacity 0.2s;
}

.toast-close:hover {
  opacity: 1;
}

.toast-close svg {
  width: 16px;
  height: 16px;
}

/* Slide-up Animation */
@keyframes slideUp {
  from {
    transform: translateY(100%);
    opacity: 0;
  }
  to {
    transform: translateY(0);
    opacity: 1;
  }
}

.slide-up-enter-active,
.slide-up-leave-active {
  transition: all 0.3s ease;
}

.slide-up-enter-from,
.slide-up-leave-to {
  transform: translateY(100%);
  opacity: 0;
}

/* Responsive Design */
@media (max-width: 768px) {
  .restocking {
    padding: 0 1rem;
  }

  .page-header h2 {
    font-size: 1.5rem;
  }

  .budget-container {
    flex-direction: column;
    gap: 1rem;
  }

  .btn {
    width: 100%;
  }

  .card-header {
    flex-direction: column;
    align-items: flex-start;
  }

  .recommendation-stats {
    flex-direction: column;
    gap: 1rem;
  }

  .table-container {
    overflow-x: auto;
  }

  table {
    font-size: 0.8rem;
  }

  td,
  th {
    padding: 0.5rem;
  }

  .toast {
    left: 1rem;
    right: 1rem;
    max-width: none;
  }
}
</style>
