<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Review demand signals and place restocking orders</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Budget Card -->
      <div class="card budget-card">
        <div class="budget-stats">
          <div class="budget-stat">
            <span class="budget-label">Budget</span>
            <span class="budget-value">${{ budget.toLocaleString() }}</span>
          </div>
          <div class="budget-divider"></div>
          <div class="budget-stat">
            <span class="budget-label">Allocated</span>
            <span class="budget-value" :class="{ 'over-budget': allocated > budget }">
              ${{ allocated.toLocaleString() }}
            </span>
          </div>
          <div class="budget-divider"></div>
          <div class="budget-stat">
            <span class="budget-label">Remaining</span>
            <span class="budget-value" :class="{ 'over-budget': remaining < 0 }">
              ${{ remaining.toLocaleString() }}
            </span>
          </div>
        </div>
        <div class="slider-container">
          <input
            type="range"
            v-model.number="budget"
            :min="0"
            :max="budgetMax"
            :step="1000"
            class="budget-slider"
          />
          <div class="slider-labels">
            <span>$0</span>
            <span>${{ budgetMax.toLocaleString() }}</span>
          </div>
        </div>
      </div>

      <!-- Success Message -->
      <div v-if="submitted && orderResult" class="card success-card">
        <div class="success-content">
          <div class="success-title">Order placed successfully</div>
          <div class="success-details">
            <span>Order <strong>{{ orderResult.order_number }}</strong></span>
            <span>Expected delivery: <strong>{{ formatDate(orderResult.expected_delivery) }}</strong></span>
            <span>Total: <strong>${{ orderResult.total_value.toLocaleString() }}</strong></span>
          </div>
          <button class="btn btn-secondary" @click="resetOrder">Place Another Order</button>
        </div>
      </div>

      <!-- Error message for order submission -->
      <div v-if="submitError" class="error">{{ submitError }}</div>

      <!-- Recommended Items Table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Eligible Items ({{ eligibleItems.length }})</h3>
          <button
            class="btn btn-primary"
            :disabled="selectedSkus.size === 0 || budget === 0 || submitting || submitted"
            @click="placeOrder"
          >
            {{ submitting ? 'Placing...' : 'Place Order' }}
          </button>
        </div>
        <div v-if="eligibleItems.length === 0" class="empty-state">
          No items currently eligible for restocking.
        </div>
        <div v-else class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th class="col-check"></th>
                <th>Item Name</th>
                <th>SKU</th>
                <th>Warehouse</th>
                <th>Current Stock</th>
                <th>Reorder Point</th>
                <th>Qty to Order</th>
                <th>Unit Cost</th>
                <th>Total Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in eligibleItems"
                :key="item.item_sku"
                :class="{ 'row-recommended': isRecommended(item.item_sku) }"
              >
                <td class="col-check">
                  <input
                    type="checkbox"
                    :checked="selectedSkus.has(item.item_sku)"
                    @change="toggleSku(item.item_sku)"
                  />
                </td>
                <td>
                  <div class="item-name-cell">
                    {{ item.item_name }}
                    <span v-if="isRecommended(item.item_sku)" class="badge info recommended-badge">Recommended</span>
                  </div>
                </td>
                <td class="sku-cell">{{ item.item_sku }}</td>
                <td>{{ item.warehouse }}</td>
                <td>{{ item.quantity_on_hand }}</td>
                <td>{{ item.reorder_point }}</td>
                <td><strong>{{ item.restock_qty }}</strong></td>
                <td>${{ item.unit_cost.toLocaleString() }}</td>
                <td><strong>${{ item.line_total.toLocaleString() }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const submitted = ref(false)
    const submitError = ref(null)
    const orderResult = ref(null)

    const demandForecasts = ref([])
    const inventory = ref([])
    const budget = ref(50000)
    // Use a ref wrapping a Set — reassign to trigger reactivity
    const selectedSkus = ref(new Set())

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [forecasts, inv] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({})
        ])
        demandForecasts.value = forecasts
        inventory.value = inv
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    // All items eligible for restocking (increasing trend + below reorder point)
    const eligibleItems = computed(() => {
      return demandForecasts.value
        .filter(f => f.trend === 'increasing')
        .map(f => {
          const inv = inventory.value.find(i => i.sku === f.item_sku)
          if (!inv || inv.quantity_on_hand > inv.reorder_point) return null
          const qty = inv.reorder_point - inv.quantity_on_hand
          return {
            ...f,
            unit_cost: inv.unit_cost,
            restock_qty: qty,
            line_total: qty * inv.unit_cost,
            warehouse: inv.warehouse,
            category: inv.category,
            quantity_on_hand: inv.quantity_on_hand,
            reorder_point: inv.reorder_point
          }
        })
        .filter(Boolean)
        .sort((a, b) => b.line_total - a.line_total)
    })

    // Greedy budget allocation: add items in order until budget exceeded
    const recommendedItems = computed(() => {
      let runningTotal = 0
      const result = []
      for (const item of eligibleItems.value) {
        if (runningTotal + item.line_total <= budget.value) {
          result.push(item)
          runningTotal += item.line_total
        }
      }
      return result
    })

    // Max budget slider value based on total of all eligible items
    const budgetMax = computed(() => {
      const totalEligible = eligibleItems.value.reduce((sum, i) => sum + i.line_total, 0)
      return Math.max(Math.ceil(totalEligible / 10000) * 10000, 10000)
    })

    // Allocated = sum of line_total for items in selectedSkus
    const allocated = computed(() => {
      return eligibleItems.value
        .filter(i => selectedSkus.value.has(i.item_sku))
        .reduce((sum, i) => sum + i.line_total, 0)
    })

    const remaining = computed(() => budget.value - allocated.value)

    // When recommended items change, reset selectedSkus to recommended set
    watch(recommendedItems, (newItems) => {
      selectedSkus.value = new Set(newItems.map(i => i.item_sku))
    })

    const isRecommended = (sku) => {
      return recommendedItems.value.some(i => i.item_sku === sku)
    }

    const toggleSku = (sku) => {
      const next = new Set(selectedSkus.value)
      if (next.has(sku)) {
        next.delete(sku)
      } else {
        next.add(sku)
      }
      selectedSkus.value = next
    }

    const formatDate = (dateString) => {
      const date = new Date(dateString)
      if (isNaN(date.getTime())) return dateString
      return date.toLocaleDateString('en-US', {
        year: 'numeric',
        month: 'short',
        day: 'numeric'
      })
    }

    const placeOrder = async () => {
      if (selectedSkus.value.size === 0 || budget.value === 0 || submitting.value) return

      submitting.value = true
      submitError.value = null

      try {
        const selectedItems = eligibleItems.value
          .filter(i => selectedSkus.value.has(i.item_sku))
          .map(i => ({
            sku: i.item_sku,
            name: i.item_name,
            quantity: i.restock_qty,
            unit_price: i.unit_cost
          }))

        const response = await api.createRestockOrder({
          items: selectedItems,
          warehouse: null,
          category: null
        })

        orderResult.value = response
        submitted.value = true
      } catch (err) {
        submitError.value = 'Failed to place order: ' + err.message
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    const resetOrder = () => {
      submitted.value = false
      orderResult.value = null
      submitError.value = null
      // Re-populate selectedSkus from recommended
      selectedSkus.value = new Set(recommendedItems.value.map(i => i.item_sku))
    }

    onMounted(loadData)

    return {
      loading,
      error,
      submitting,
      submitted,
      submitError,
      orderResult,
      budget,
      selectedSkus,
      eligibleItems,
      recommendedItems,
      budgetMax,
      allocated,
      remaining,
      isRecommended,
      toggleSku,
      formatDate,
      placeOrder,
      resetOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  /* inherits main-content padding */
}

/* Budget Card */
.budget-card {
  margin-bottom: 1.25rem;
}

.budget-stats {
  display: flex;
  align-items: center;
  gap: 0;
  margin-bottom: 1.25rem;
}

.budget-stat {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  padding: 0 2rem;
}

.budget-stat:first-child {
  padding-left: 0;
}

.budget-divider {
  width: 1px;
  height: 40px;
  background: #e2e8f0;
}

.budget-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.budget-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-value.over-budget {
  color: #dc2626;
}

.slider-container {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.budget-slider {
  width: 100%;
  height: 6px;
  -webkit-appearance: none;
  appearance: none;
  background: #e2e8f0;
  border-radius: 3px;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
}

.budget-slider::-moz-range-thumb {
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
}

/* Success Card */
.success-card {
  border-color: #86efac;
  background: #f0fdf4;
  margin-bottom: 1.25rem;
}

.success-content {
  display: flex;
  align-items: center;
  gap: 1.5rem;
  flex-wrap: wrap;
}

.success-title {
  font-weight: 700;
  color: #065f46;
  font-size: 1rem;
}

.success-details {
  display: flex;
  gap: 1.5rem;
  flex-wrap: wrap;
  color: #047857;
  font-size: 0.875rem;
}

/* Table */
.restock-table {
  width: 100%;
  border-collapse: collapse;
}

.col-check {
  width: 40px;
}

.row-recommended {
  background: #eff6ff;
}

.row-recommended:hover {
  background: #dbeafe !important;
}

.item-name-cell {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  flex-wrap: wrap;
}

.recommended-badge {
  font-size: 0.688rem;
  padding: 0.188rem 0.5rem;
}

.sku-cell {
  font-family: monospace;
  font-size: 0.813rem;
  color: #475569;
}

/* Buttons */
.btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 0.5rem 1.25rem;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  border: none;
  transition: all 0.2s ease;
  text-decoration: none;
}

.btn-primary {
  background: #2563eb;
  color: white;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  background: #93c5fd;
  cursor: not-allowed;
}

.btn-secondary {
  background: white;
  color: #374151;
  border: 1px solid #d1d5db;
}

.btn-secondary:hover {
  background: #f9fafb;
}

.empty-state {
  text-align: center;
  padding: 3rem;
  color: #64748b;
  font-size: 0.938rem;
}
</style>
