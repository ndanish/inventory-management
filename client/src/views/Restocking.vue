<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Recommendations</h2>
      <p>Budget-based recommendations from demand forecasts</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Budget Card -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">Available Budget</h3>
        </div>
        <div class="budget-body">
          <div class="budget-labels">
            <span class="budget-amount">Budget: ${{ budget.toLocaleString() }}</span>
            <span class="budget-cost">
              Est. Cost: ${{ totalCost.toLocaleString() }} / ${{ maxBudget.toLocaleString() }}
            </span>
          </div>
          <input
            type="range"
            class="budget-slider"
            :min="0"
            :max="maxBudget"
            :step="1000"
            v-model.number="budget"
          />
          <div class="budget-progress-track">
            <div
              class="budget-progress-fill"
              :style="{ width: budgetUsageWidth }"
              :class="{ 'over-budget': totalCost > budget }"
            ></div>
          </div>
        </div>
      </div>

      <!-- Recommendations Table Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">
            Recommended Items ({{ recommendedItems.length }} of {{ enrichedItems.length }})
          </h3>
        </div>

        <div v-if="enrichedItems.length === 0" class="empty-state">
          No demand gaps found
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>SKU</th>
                <th>Item Name</th>
                <th>Trend</th>
                <th>Demand Gap</th>
                <th>Unit Cost</th>
                <th>Est. Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in enrichedItems"
                :key="item.item_sku"
                :class="{ 'row-excluded': !isRecommended(item.item_sku) }"
              >
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ item.item_name }}</td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td>{{ item.demandGap }}</td>
                <td>${{ item.unit_cost.toLocaleString() }}</td>
                <td><strong>${{ item.restockCost.toLocaleString() }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Place Order Section -->
      <div class="card order-card">
        <div class="card-header">
          <h3 class="card-title">Place Order</h3>
        </div>

        <div v-if="orderSuccess" class="success-message">
          Restocking order submitted. View it in the Orders tab.
        </div>
        <div v-if="orderError" class="error">{{ orderError }}</div>

        <div class="order-actions">
          <button
            class="btn btn-primary"
            :disabled="recommendedItems.length === 0 || isSubmitting"
            @click="showLeadTimeInput = true"
            v-if="!showLeadTimeInput && !orderSuccess"
          >
            Place Order
          </button>
        </div>

        <div v-if="showLeadTimeInput" class="lead-time-section">
          <label class="lead-time-label" for="lead-time-input">
            Delivery lead time (days)
          </label>
          <div class="lead-time-controls">
            <input
              id="lead-time-input"
              type="number"
              class="lead-time-input"
              :min="1"
              :max="365"
              v-model.number="leadTimeDays"
            />
            <button
              class="btn btn-primary"
              :disabled="isSubmitting"
              @click="submitOrder"
            >
              {{ isSubmitting ? 'Submitting...' : 'Confirm Order' }}
            </button>
            <button
              class="btn btn-secondary"
              :disabled="isSubmitting"
              @click="showLeadTimeInput = false"
            >
              Cancel
            </button>
          </div>
        </div>
      </div>

    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const loading = ref(true)
    const error = ref(null)

    const forecasts = ref([])
    const inventoryItems = ref([])

    const budget = ref(0)
    const leadTimeDays = ref(14)
    const showLeadTimeInput = ref(false)
    const isSubmitting = ref(false)
    const orderSuccess = ref(false)
    const orderError = ref(null)

    // Join forecast items with inventory, keep only positive gap items
    const enrichedItems = computed(() => {
      const inventoryMap = {}
      inventoryItems.value.forEach(inv => {
        inventoryMap[inv.sku] = inv
      })

      const items = forecasts.value
        .filter(f => f.forecasted_demand > f.current_demand)
        .map(f => {
          const inv = inventoryMap[f.item_sku]
          const unitCost = inv ? (inv.unit_cost || 0) : 0
          const demandGap = f.forecasted_demand - f.current_demand
          return {
            ...f,
            unit_cost: unitCost,
            demandGap,
            restockCost: demandGap * unitCost
          }
        })

      // Sort: increasing first, stable second, decreasing third
      const trendOrder = { increasing: 0, stable: 1, decreasing: 2 }
      items.sort((a, b) => {
        const aOrder = trendOrder[a.trend] !== undefined ? trendOrder[a.trend] : 3
        const bOrder = trendOrder[b.trend] !== undefined ? trendOrder[b.trend] : 3
        return aOrder - bOrder
      })

      return items
    })

    const maxBudget = computed(() => {
      const total = enrichedItems.value.reduce((sum, i) => sum + i.restockCost, 0)
      return Math.ceil(total / 1000) * 1000
    })

    // Greedy selection within budget
    const recommendedItems = computed(() => {
      let running = 0
      const result = []
      for (const item of enrichedItems.value) {
        if (running + item.restockCost <= budget.value) {
          result.push(item)
          running += item.restockCost
        }
      }
      return result
    })

    const totalCost = computed(() => {
      return recommendedItems.value.reduce((sum, i) => sum + i.restockCost, 0)
    })

    const recommendedSkus = computed(() => {
      return new Set(recommendedItems.value.map(i => i.item_sku))
    })

    const isRecommended = (sku) => {
      return recommendedSkus.value.has(sku)
    }

    const budgetUsageWidth = computed(() => {
      if (budget.value === 0) return '0%'
      const pct = Math.min((totalCost.value / budget.value) * 100, 100)
      return pct.toFixed(1) + '%'
    })

    const submitOrder = async () => {
      isSubmitting.value = true
      orderError.value = null
      try {
        const now = new Date()
        const delivery = new Date(now.getTime() + leadTimeDays.value * 86400000)
        await api.createOrder({
          order_number: `RST-${now.getTime()}`,
          customer: 'Internal Restocking',
          items: recommendedItems.value.map(i => ({
            sku: i.item_sku,
            name: i.item_name,
            quantity: i.demandGap,
            unit_price: i.unit_cost
          })),
          status: 'Submitted',
          order_date: now.toISOString().slice(0, 19),
          expected_delivery: delivery.toISOString().slice(0, 19),
          total_value: totalCost.value,
          warehouse: null,
          category: null
        })
        orderSuccess.value = true
        showLeadTimeInput.value = false
      } catch (e) {
        orderError.value = 'Failed to submit order. Please try again.'
      } finally {
        isSubmitting.value = false
      }
    }

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [forecastsData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        forecasts.value = forecastsData
        inventoryItems.value = inventoryData
        // Set initial budget to half of maxBudget after data loads
        budget.value = Math.round(maxBudget.value / 2)
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    onMounted(loadData)

    return {
      loading,
      error,
      enrichedItems,
      maxBudget,
      budget,
      recommendedItems,
      totalCost,
      budgetUsageWidth,
      isRecommended,
      leadTimeDays,
      showLeadTimeInput,
      isSubmitting,
      orderSuccess,
      orderError,
      submitOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  /* inherits main-content padding from App.vue */
}

/* Budget card body */
.budget-body {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.budget-labels {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.budget-amount {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-cost {
  font-size: 0.875rem;
  color: #64748b;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  cursor: pointer;
  height: 6px;
}

.budget-progress-track {
  width: 100%;
  height: 8px;
  background: #e2e8f0;
  border-radius: 4px;
  overflow: hidden;
}

.budget-progress-fill {
  height: 100%;
  background: #2563eb;
  border-radius: 4px;
  transition: width 0.2s ease;
}

.budget-progress-fill.over-budget {
  background: #dc2626;
}

/* Row excluded (not in budget) */
.row-excluded {
  opacity: 0.4;
}

/* Empty state */
.empty-state {
  padding: 3rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

/* Order card */
.order-card {
  margin-top: 0;
}

.order-actions {
  padding: 0.5rem 0;
}

/* Lead time section */
.lead-time-section {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  padding-top: 0.5rem;
}

.lead-time-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #475569;
}

.lead-time-controls {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  flex-wrap: wrap;
}

.lead-time-input {
  width: 100px;
  padding: 0.5rem 0.75rem;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  font-size: 0.875rem;
  color: #0f172a;
  background: #fff;
  outline: none;
  transition: border-color 0.2s;
}

.lead-time-input:focus {
  border-color: #2563eb;
  box-shadow: 0 0 0 2px rgba(37, 99, 235, 0.15);
}

/* Buttons */
.btn {
  padding: 0.5rem 1.25rem;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  border: none;
  transition: all 0.2s ease;
  white-space: nowrap;
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-primary {
  background: #2563eb;
  color: white;
}

.btn-primary:not(:disabled):hover {
  background: #1d4ed8;
}

.btn-secondary {
  background: #f1f5f9;
  color: #475569;
  border: 1px solid #e2e8f0;
}

.btn-secondary:not(:disabled):hover {
  background: #e2e8f0;
  color: #0f172a;
}

/* Success message */
.success-message {
  background: #d1fae5;
  border: 1px solid #a7f3d0;
  color: #065f46;
  padding: 0.875rem 1rem;
  border-radius: 8px;
  font-size: 0.938rem;
  margin-bottom: 1rem;
}
</style>
