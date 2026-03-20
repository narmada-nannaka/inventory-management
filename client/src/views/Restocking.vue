<script setup>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

const LEAD_TIME_DAYS = {
  'Sensors': 7,
  'Circuit Boards': 14,
  'Controllers': 14,
  'Actuators': 21,
  'Power Supplies': 21
}
const DEFAULT_LEAD_TIME = 14

const loading = ref(true)
const error = ref(null)
const budget = ref(100000)
const allForecasts = ref([])
const allInventory = ref([])
const submitting = ref(false)
const orderSuccess = ref(false)
const lastOrderNumber = ref('')

const recommendations = computed(() => {
  const inventoryMap = new Map()
  for (const item of allInventory.value) {
    inventoryMap.set(item.sku, item)
  }

  const scored = []
  for (const forecast of allForecasts.value) {
    const inventoryItem = inventoryMap.get(forecast.item_sku)
    if (!inventoryItem) continue

    const qty_to_order = forecast.forecasted_demand
    const unit_cost = inventoryItem.unit_cost
    const total_cost = qty_to_order * unit_cost
    const current_stock = inventoryItem.quantity_on_hand
    const reorder_point = inventoryItem.reorder_point
    const lead_time = LEAD_TIME_DAYS[inventoryItem.category] ?? DEFAULT_LEAD_TIME
    const is_increasing = forecast.trend === 'increasing'
    const is_below_reorder = inventoryItem.quantity_on_hand <= inventoryItem.reorder_point

    let score
    if (is_increasing && is_below_reorder) score = 3
    else if (is_increasing && !is_below_reorder) score = 2
    else if (!is_increasing && is_below_reorder) score = 1
    else score = 0

    scored.push({
      sku: forecast.item_sku,
      name: inventoryItem.name,
      category: inventoryItem.category,
      current_stock,
      reorder_point,
      forecasted_demand: forecast.forecasted_demand,
      qty_to_order,
      unit_cost,
      total_cost,
      lead_time,
      score
    })
  }

  scored.sort((a, b) => b.score - a.score)

  const selected = []
  let running_total = 0
  for (const item of scored) {
    if (running_total + item.total_cost <= budget.value) {
      selected.push(item)
      running_total += item.total_cost
    }
  }

  return selected
})

const totalOrderCost = computed(() =>
  recommendations.value.reduce((sum, item) => sum + item.total_cost, 0)
)

const remainingBudget = computed(() => budget.value - totalOrderCost.value)

const getPriorityClass = (score) => {
  if (score === 3) return 'danger'
  if (score === 2) return 'warning'
  if (score === 1) return 'info'
  return 'stable'
}

const getPriorityLabel = (score) => {
  if (score === 3) return 'Critical'
  if (score === 2) return 'High'
  if (score === 1) return 'Medium'
  return 'Low'
}

const placeOrder = async () => {
  submitting.value = true
  orderSuccess.value = false
  try {
    const order = await api.createRestockingOrder({
      items: recommendations.value.map(r => ({
        sku: r.sku,
        name: r.name,
        category: r.category,
        quantity: r.qty_to_order,
        unit_cost: r.unit_cost
      })),
      budget: budget.value
    })
    lastOrderNumber.value = order.order_number
    orderSuccess.value = true
  } catch (err) {
    error.value = 'Failed to place order: ' + err.message
  } finally {
    submitting.value = false
  }
}

const loadData = async () => {
  try {
    loading.value = true
    error.value = null
    const [forecastsData, inventoryData] = await Promise.all([
      api.getDemandForecasts(),
      api.getInventory()
    ])
    allForecasts.value = forecastsData
    allInventory.value = inventoryData
  } catch (err) {
    error.value = 'Failed to load data: ' + err.message
  } finally {
    loading.value = false
  }
}

onMounted(() => loadData())
</script>

<template>
  <div class="view-container">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Plan and submit restocking orders based on demand forecasts and budget.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div v-if="orderSuccess" class="success-banner">
        Order {{ lastOrderNumber }} submitted successfully. View it in the Orders tab.
      </div>

      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">Budget</h3>
        </div>
        <div class="budget-controls">
          <div class="budget-display">
            <span class="budget-label">Budget</span>
            <span class="budget-value">${{ budget.toLocaleString() }}</span>
          </div>
          <input
            type="range"
            min="10000"
            max="500000"
            step="5000"
            v-model.number="budget"
            class="budget-slider"
          />
          <div class="budget-range-labels">
            <span>$10K</span>
            <span>$500K</span>
          </div>
        </div>
        <div class="budget-summary">
          <div class="budget-stat">
            <span class="budget-stat-label">Order Cost</span>
            <span class="budget-stat-value">${{ totalOrderCost.toLocaleString() }}</span>
          </div>
          <div class="budget-stat">
            <span class="budget-stat-label">Remaining</span>
            <span class="budget-stat-value" :class="{ 'text-danger': remainingBudget < 0 }">
              ${{ remainingBudget.toLocaleString() }}
            </span>
          </div>
          <div class="budget-stat">
            <span class="budget-stat-label">Items</span>
            <span class="budget-stat-value">{{ recommendations.length }}</span>
          </div>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Restocking ({{ recommendations.length }} items)</h3>
          <button
            class="btn-primary"
            @click="placeOrder"
            :disabled="submitting || recommendations.length === 0"
          >
            {{ submitting ? 'Placing Order...' : 'Place Order' }}
          </button>
        </div>

        <div v-if="recommendations.length === 0" class="empty-state">
          No items fit within the current budget. Try increasing the budget.
        </div>

        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>Priority</th>
                <th>SKU</th>
                <th>Name</th>
                <th>Category</th>
                <th>Current Stock</th>
                <th>Reorder Point</th>
                <th>Forecasted Demand</th>
                <th>Qty to Order</th>
                <th>Unit Cost</th>
                <th>Total Cost</th>
                <th>Lead Time</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendations" :key="item.sku">
                <td>
                  <span :class="['badge', getPriorityClass(item.score)]">
                    {{ getPriorityLabel(item.score) }}
                  </span>
                </td>
                <td><strong>{{ item.sku }}</strong></td>
                <td>{{ item.name }}</td>
                <td>{{ item.category }}</td>
                <td>{{ item.current_stock }}</td>
                <td>{{ item.reorder_point }}</td>
                <td>{{ item.forecasted_demand }}</td>
                <td><strong>{{ item.qty_to_order }}</strong></td>
                <td>${{ item.unit_cost.toFixed(2) }}</td>
                <td><strong>${{ item.total_cost.toLocaleString() }}</strong></td>
                <td>{{ item.lead_time }} days</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.view-container {
  padding: 2rem;
}

.page-header {
  margin-bottom: 1.5rem;
}

.page-header h2 {
  margin: 0 0 0.25rem;
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
}

.page-header p {
  margin: 0;
  color: #64748b;
  font-size: 0.938rem;
}

.budget-card {
  margin-bottom: 1.25rem;
}

.budget-controls {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  margin-bottom: 1.25rem;
}

.budget-display {
  display: flex;
  align-items: baseline;
  gap: 1rem;
}

.budget-label {
  font-size: 0.875rem;
  color: #64748b;
  font-weight: 600;
}

.budget-value {
  font-size: 1.75rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-slider {
  width: 100%;
  max-width: 500px;
  accent-color: #2563eb;
  cursor: pointer;
}

.budget-range-labels {
  display: flex;
  justify-content: space-between;
  max-width: 500px;
  font-size: 0.75rem;
  color: #94a3b8;
}

.budget-summary {
  display: flex;
  gap: 2rem;
  padding-top: 1rem;
  border-top: 1px solid #e2e8f0;
}

.budget-stat {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.budget-stat-label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.budget-stat-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
}

.text-danger {
  color: #dc2626;
}

.btn-primary {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.5rem 1.25rem;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.empty-state {
  text-align: center;
  padding: 2rem;
  color: #64748b;
  font-size: 0.938rem;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.875rem 1rem;
  border-radius: 8px;
  font-weight: 500;
  margin-bottom: 1.25rem;
}

.table-container {
  overflow-x: auto;
}
</style>
