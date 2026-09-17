<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Review demand forecasts and place restocking orders within your budget.</p>
    </div>

    <div v-if="loading" class="loading">Loading restocking data...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <template v-else>
      <!-- Section 1: Budget Controls -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Budget Controls</h3>
        </div>
        <div class="budget-controls">
          <div class="budget-display">
            <span class="budget-label">Restocking Budget</span>
            <span class="budget-value">{{ formatCurrency(budget) }}</span>
          </div>
          <div class="budget-inputs">
            <input
              type="range"
              class="budget-slider"
              min="1000"
              max="200000"
              step="1000"
              v-model.number="budget"
            />
            <input
              type="number"
              class="budget-number"
              min="1000"
              max="200000"
              step="1000"
              v-model.number="budget"
            />
          </div>
          <div class="budget-range-labels">
            <span>$1,000</span>
            <span>$200,000</span>
          </div>
        </div>
      </div>

      <!-- Section 2: Recommended Items -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items</h3>
          <span class="card-subtitle">Sorted by demand trend priority. Pre-selected within budget.</span>
        </div>
        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th class="col-check"></th>
                <th>Item Name</th>
                <th>SKU</th>
                <th>Trend</th>
                <th>Qty to Restock</th>
                <th>Unit Cost</th>
                <th>Subtotal</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in sortedItems"
                :key="item.item_sku"
                :class="{ 'row-selected': selectedSkus.has(item.item_sku) }"
              >
                <td>
                  <input
                    type="checkbox"
                    :checked="selectedSkus.has(item.item_sku)"
                    @change="toggleSku(item.item_sku)"
                    class="row-checkbox"
                  />
                </td>
                <td>{{ item.item_name }}</td>
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>
                  <span :class="['badge', trendBadgeClass(item.trend)]">{{ item.trend }}</span>
                </td>
                <td>{{ item.forecasted_demand }}</td>
                <td>{{ formatCurrency(item.unit_cost) }}</td>
                <td>{{ formatCurrency(item.subtotal) }}</td>
              </tr>
            </tbody>
          </table>
        </div>

        <!-- Budget Summary Bar -->
        <div class="budget-summary">
          <div class="budget-summary-text">
            <span class="summary-selected">
              Selected: <strong>{{ formatCurrency(selectedTotal) }}</strong>
              of <strong>{{ formatCurrency(budget) }}</strong> budget
            </span>
            <span class="summary-count">{{ selectedSkus.size }} item{{ selectedSkus.size !== 1 ? 's' : '' }} selected</span>
          </div>
          <div class="progress-bar-track">
            <div
              class="progress-bar-fill"
              :class="{ 'over-budget': selectedTotal > budget }"
              :style="{ width: progressPercent + '%' }"
            ></div>
          </div>
          <div v-if="selectedTotal > budget" class="over-budget-warning">
            Over budget by {{ formatCurrency(selectedTotal - budget) }}
          </div>
        </div>
      </div>

      <!-- Section 3: Place Order -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Place Restocking Order</h3>
        </div>

        <div v-if="orderSuccess" class="success-message">
          Restocking order {{ orderSuccess.order_number }} placed successfully.
          Expected delivery: {{ formatDate(orderSuccess.expected_delivery) }}
        </div>

        <div v-if="orderError" class="error">{{ orderError }}</div>

        <button
          class="place-order-btn"
          :disabled="selectedSkus.size === 0 || orderLoading"
          @click="placeOrder"
        >
          <span v-if="orderLoading">Placing Order...</span>
          <span v-else>Place Order ({{ formatCurrency(selectedTotal) }})</span>
        </button>
      </div>

      <!-- Section 4: Submitted Orders -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Submitted Restocking Orders</h3>
        </div>

        <div v-if="ordersLoading" class="loading">Loading orders...</div>
        <div v-else-if="restockOrders.length === 0" class="empty-state">
          No restocking orders placed yet.
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>Order #</th>
                <th>Date Placed</th>
                <th>Items</th>
                <th>Total Value</th>
                <th>Expected Delivery</th>
                <th>Status</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="order in restockOrders" :key="order.id">
                <td><strong>{{ order.order_number }}</strong></td>
                <td>{{ formatDate(order.order_date) }}</td>
                <td>{{ order.items ? order.items.length : '-' }}</td>
                <td>{{ formatCurrency(order.total_value) }}</td>
                <td>{{ order.expected_delivery ? formatDate(order.expected_delivery) : '-' }}</td>
                <td><span class="badge badge-submitted">{{ order.status }}</span></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </template>
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
    const budget = ref(50000)
    const forecastData = ref([])
    const inventoryData = ref([])
    const selectedSkus = ref(new Set())
    const orderLoading = ref(false)
    const orderSuccess = ref(null)
    const orderError = ref(null)
    const restockOrders = ref([])
    const ordersLoading = ref(false)

    const trendOrder = { increasing: 1, stable: 2, decreasing: 3 }

    const sortedItems = computed(() => {
      const inventoryMap = {}
      for (const inv of inventoryData.value) {
        inventoryMap[inv.sku] = inv.unit_cost
      }

      return forecastData.value
        .map(forecast => {
          const unit_cost = inventoryMap[forecast.item_sku] ?? 0
          const subtotal = forecast.forecasted_demand * unit_cost
          return {
            ...forecast,
            unit_cost,
            subtotal
          }
        })
        .sort((a, b) => (trendOrder[a.trend] || 99) - (trendOrder[b.trend] || 99))
    })

    const runGreedyAlgorithm = () => {
      let remaining = budget.value
      const recommended = new Set()
      for (const item of sortedItems.value) {
        if (item.subtotal > 0 && item.subtotal <= remaining) {
          recommended.add(item.item_sku)
          remaining -= item.subtotal
        }
      }
      selectedSkus.value = recommended
    }

    watch(budget, () => {
      runGreedyAlgorithm()
    })

    const selectedItems = computed(() => {
      return sortedItems.value.filter(item => selectedSkus.value.has(item.item_sku))
    })

    const selectedTotal = computed(() => {
      return selectedItems.value.reduce((sum, item) => sum + item.subtotal, 0)
    })

    const progressPercent = computed(() => {
      if (budget.value === 0) return 0
      return Math.min((selectedTotal.value / budget.value) * 100, 100)
    })

    const toggleSku = (sku) => {
      const newSet = new Set(selectedSkus.value)
      if (newSet.has(sku)) {
        newSet.delete(sku)
      } else {
        newSet.add(sku)
      }
      selectedSkus.value = newSet
    }

    const trendBadgeClass = (trend) => {
      if (trend === 'increasing') return 'increasing'
      if (trend === 'decreasing') return 'decreasing'
      return 'stable'
    }

    const formatCurrency = (value) => {
      if (value == null) return '-'
      return value.toLocaleString('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 0 })
    }

    const formatDate = (dateStr) => {
      if (!dateStr) return '-'
      const d = new Date(dateStr)
      if (isNaN(d.getTime())) return dateStr
      return d.toLocaleDateString('en-US', { year: 'numeric', month: 'short', day: 'numeric' })
    }

    const loadRestockOrders = async () => {
      ordersLoading.value = true
      try {
        restockOrders.value = await api.getRestockOrders()
      } catch (err) {
        console.error('Failed to load restock orders:', err)
      } finally {
        ordersLoading.value = false
      }
    }

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({})
        ])
        forecastData.value = forecasts
        inventoryData.value = inventory
        runGreedyAlgorithm()
      } catch (err) {
        error.value = 'Failed to load restocking data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      orderLoading.value = true
      orderError.value = null
      orderSuccess.value = null
      try {
        const items = selectedItems.value.map(item => ({
          sku: item.item_sku,
          name: item.item_name,
          quantity: item.forecasted_demand,
          unit_price: item.unit_cost
        }))
        const result = await api.createRestockOrder({
          items,
          total_value: selectedTotal.value,
          warehouse: null
        })
        orderSuccess.value = result
        await loadRestockOrders()
        runGreedyAlgorithm()
      } catch (err) {
        orderError.value = 'Failed to place order: ' + (err.response?.data?.detail || err.message)
      } finally {
        orderLoading.value = false
      }
    }

    onMounted(async () => {
      await loadData()
      await loadRestockOrders()
    })

    return {
      loading,
      error,
      budget,
      sortedItems,
      selectedSkus,
      selectedTotal,
      progressPercent,
      orderLoading,
      orderSuccess,
      orderError,
      restockOrders,
      ordersLoading,
      toggleSku,
      trendBadgeClass,
      formatCurrency,
      formatDate,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding-bottom: 2rem;
}

/* Budget Controls */
.budget-controls {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.budget-display {
  display: flex;
  align-items: baseline;
  gap: 1rem;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.budget-value {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-inputs {
  display: flex;
  align-items: center;
  gap: 1.25rem;
}

.budget-slider {
  flex: 1;
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
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
  transition: box-shadow 0.15s ease;
}

.budget-slider::-webkit-slider-thumb:hover {
  box-shadow: 0 1px 8px rgba(37, 99, 235, 0.6);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: none;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
}

.budget-number {
  width: 130px;
  padding: 0.5rem 0.75rem;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  font-size: 0.938rem;
  color: #0f172a;
  background: #f8fafc;
  outline: none;
  transition: border-color 0.15s ease;
  text-align: right;
}

.budget-number:focus {
  border-color: #2563eb;
  background: white;
}

.budget-range-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
}

/* Table row selection */
.row-selected {
  background: #eff6ff !important;
}

.row-checkbox {
  width: 16px;
  height: 16px;
  cursor: pointer;
  accent-color: #2563eb;
}

.col-check {
  width: 40px;
}

/* Budget Summary Bar */
.budget-summary {
  margin-top: 1.25rem;
  padding-top: 1.25rem;
  border-top: 1px solid #e2e8f0;
}

.budget-summary-text {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.625rem;
}

.summary-selected {
  font-size: 0.938rem;
  color: #334155;
}

.summary-count {
  font-size: 0.875rem;
  color: #64748b;
  font-weight: 500;
}

.progress-bar-track {
  height: 10px;
  background: #e2e8f0;
  border-radius: 5px;
  overflow: hidden;
}

.progress-bar-fill {
  height: 100%;
  background: #10b981;
  border-radius: 5px;
  transition: width 0.3s ease, background 0.2s ease;
}

.progress-bar-fill.over-budget {
  background: #ef4444;
}

.over-budget-warning {
  margin-top: 0.5rem;
  font-size: 0.813rem;
  color: #dc2626;
  font-weight: 500;
}

/* Place Order */
.place-order-btn {
  display: inline-block;
  padding: 0.75rem 2rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s ease, opacity 0.15s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.success-message {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.875rem 1rem;
  border-radius: 8px;
  margin-bottom: 1rem;
  font-size: 0.938rem;
  font-weight: 500;
}

/* Empty state */
.empty-state {
  padding: 2.5rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

/* Submitted status badge */
.badge-submitted {
  background: #ede9fe;
  color: #5b21b6;
}

.card-subtitle {
  font-size: 0.813rem;
  color: #64748b;
}
</style>
