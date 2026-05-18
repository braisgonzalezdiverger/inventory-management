<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Automatically select items to restock based on demand forecasts and your available budget.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Available Budget</h3>
        </div>
        <div class="budget-display">{{ formatCurrency(budget, currentCurrency) }}</div>
        <input
          type="range"
          class="budget-slider"
          :min="0"
          :max="maxBudget"
          :step="sliderStep"
          v-model.number="budget"
        />
        <div class="slider-labels">
          <span>Min $0</span>
          <span>Max {{ formatCurrency(maxBudget, currentCurrency) }}</span>
        </div>
        <div class="budget-progress-bar">
          <div
            class="budget-progress-fill"
            :class="{
              'fill-danger': budgetUsedPercent >= 90,
              'fill-warning': budgetUsedPercent >= 70 && budgetUsedPercent < 90
            }"
            :style="{ width: budgetUsedPercent + '%' }"
          ></div>
        </div>
        <div class="budget-meta">
          {{ recommendedItems.length }} of {{ enrichedItems.length }} items fit in budget &middot; {{ formatCurrency(budgetRemaining, currentCurrency) }} remaining
        </div>
      </div>

      <!-- Recommendations Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items</h3>
          <span class="badge info">{{ recommendedItems.length }} item{{ recommendedItems.length !== 1 ? 's' : '' }}</span>
        </div>
        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th>SKU</th>
                <th>Item Name</th>
                <th>Trend</th>
                <th>Forecasted Qty</th>
                <th>Unit Cost</th>
                <th>Total Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in enrichedItems"
                :key="item.sku"
                :class="{
                  'row-selected': isSelected(item.sku),
                  'row-dimmed': !isSelected(item.sku)
                }"
              >
                <td>{{ item.sku }}</td>
                <td>{{ item.name }}</td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td>{{ item.forecasted_demand }}</td>
                <td>{{ formatCurrency(item.unit_cost, currentCurrency) }}</td>
                <td>{{ formatCurrency(item.totalCost, currentCurrency) }}</td>
              </tr>
            </tbody>
            <tfoot>
              <tr class="totals-row">
                <td colspan="5">Selected Total</td>
                <td>{{ formatCurrency(totalSelectedCost, currentCurrency) }}</td>
              </tr>
            </tfoot>
          </table>
        </div>
      </div>

      <!-- Place Order Section -->
      <div class="place-order-section">
        <div v-if="orderSuccess" class="success-card">
          <p>Order submitted successfully! Expected delivery in 14 days.</p>
          <router-link to="/orders">View in Orders tab &rarr;</router-link>
        </div>
        <template v-else>
          <button
            class="btn-primary"
            :disabled="recommendedItems.length === 0 || submitting"
            @click="placeOrder"
          >
            Place Order ({{ recommendedItems.length }} item{{ recommendedItems.length !== 1 ? 's' : '' }} &middot; {{ formatCurrency(totalSelectedCost, currentCurrency) }})
          </button>
          <div v-if="orderError" class="order-error">{{ orderError }}</div>
        </template>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'
import { formatCurrency } from '../utils/currency'

export default {
  name: 'Restocking',
  setup() {
    const { currentCurrency } = useI18n()

    const loading = ref(true)
    const error = ref(null)
    const demandForecasts = ref([])
    const inventoryItems = ref([])
    const budget = ref(0)
    const submitting = ref(false)
    const orderSuccess = ref(false)
    const orderError = ref(null)

    const enrichedItems = computed(() => {
      return demandForecasts.value.map(forecast => {
        const inventoryItem = inventoryItems.value.find(i => i.sku === forecast.item_sku)
        return {
          id: forecast.id,
          sku: forecast.item_sku,
          name: forecast.item_name,
          trend: forecast.trend,
          current_demand: forecast.current_demand,
          forecasted_demand: forecast.forecasted_demand,
          unit_cost: inventoryItem?.unit_cost ?? 0,
          category: inventoryItem?.category ?? '',
          warehouse: inventoryItem?.warehouse ?? '',
          totalCost: forecast.forecasted_demand * (inventoryItem?.unit_cost ?? 0)
        }
      })
    })

    const maxBudget = computed(() => {
      const total = enrichedItems.value.reduce((sum, item) => sum + item.totalCost, 0)
      return Math.ceil(total / 1000) * 1000
    })

    const sliderStep = computed(() => {
      // target ~200 draggable positions; round to nearest 100
      return Math.max(100, Math.round(maxBudget.value / 200 / 100) * 100)
    })

    const trendPriority = { increasing: 1, stable: 2, decreasing: 3 }

    const recommendedItems = computed(() => {
      const sorted = [...enrichedItems.value].sort((a, b) => {
        const tp = trendPriority[a.trend] - trendPriority[b.trend]
        if (tp !== 0) return tp
        return a.totalCost - b.totalCost
      })
      let remaining = budget.value
      const selected = []
      for (const item of sorted) {
        if (item.totalCost <= remaining) {
          selected.push(item)
          remaining -= item.totalCost
        }
      }
      return selected
    })

    const selectedSkus = computed(() => new Set(recommendedItems.value.map(i => i.sku)))

    const isSelected = (sku) => selectedSkus.value.has(sku)

    const totalSelectedCost = computed(() =>
      recommendedItems.value.reduce((sum, i) => sum + i.totalCost, 0)
    )

    const budgetRemaining = computed(() => budget.value - totalSelectedCost.value)

    const budgetUsedPercent = computed(() => {
      if (budget.value === 0) return 0
      return Math.min(100, Math.round((totalSelectedCost.value / budget.value) * 100))
    })

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        demandForecasts.value = forecasts
        inventoryItems.value = inventory
        // Initialize budget to 50% of maxBudget after data loads
        budget.value = Math.round(maxBudget.value / 2 / sliderStep.value) * sliderStep.value
      } catch (err) {
        error.value = 'Failed to load data'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      submitting.value = true
      orderError.value = null
      try {
        const payload = {
          order_number: `RST-${Date.now()}`,
          customer: 'Internal Restocking',
          items: recommendedItems.value.map(i => ({
            sku: i.sku,
            name: i.name,
            quantity: i.forecasted_demand,
            unit_price: i.unit_cost
          })),
          status: 'Processing',
          order_date: new Date().toISOString(),
          expected_delivery: new Date(Date.now() + 14 * 86400000).toISOString(),
          warehouse: null,
          category: null,
          source: 'restocking'
        }
        await api.createOrder(payload)
        orderSuccess.value = true
      } catch (err) {
        orderError.value = 'Failed to submit order. Please try again.'
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadData)

    return {
      loading,
      error,
      enrichedItems,
      budget,
      maxBudget,
      sliderStep,
      recommendedItems,
      isSelected,
      totalSelectedCost,
      budgetRemaining,
      budgetUsedPercent,
      submitting,
      orderSuccess,
      orderError,
      placeOrder,
      currentCurrency,
      formatCurrency
    }
  }
}
</script>

<style scoped>
.budget-display {
  font-size: 2.5rem;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 1rem;
}

.budget-slider {
  width: 100%;
  height: 6px;
  accent-color: #3b82f6;
  cursor: pointer;
  margin: 0.75rem 0;
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #64748b;
}

.budget-progress-bar {
  height: 8px;
  background: #e2e8f0;
  border-radius: 4px;
  overflow: hidden;
  margin: 0.75rem 0;
}

.budget-progress-fill {
  height: 100%;
  background: #3b82f6;
  border-radius: 4px;
  transition: width 0.3s ease;
}

.budget-progress-fill.fill-warning {
  background: #f59e0b;
}

.budget-progress-fill.fill-danger {
  background: #ef4444;
}

.budget-meta {
  font-size: 0.875rem;
  color: #64748b;
  margin-top: 0.5rem;
}

.row-selected {
  background: #f0fdf4;
}

.row-dimmed {
  opacity: 0.4;
}

.totals-row td {
  font-weight: 600;
  background: #f8fafc;
  border-top: 2px solid #e2e8f0;
}

.place-order-section {
  margin-top: 1.5rem;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 0.75rem;
}

.btn-primary {
  padding: 0.75rem 2rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
  transform: translateY(-1px);
}

.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
  transform: none;
}

.success-card {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 8px;
  padding: 1rem 1.25rem;
  color: #065f46;
}

.success-card a {
  color: #065f46;
  font-weight: 600;
}

.order-error {
  color: #dc2626;
  font-size: 0.875rem;
}
</style>
