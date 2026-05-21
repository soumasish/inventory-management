<template>
  <div class="view-container">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Adjust your budget to see recommended items to restock based on demand forecasts.</p>
    </div>

    <div class="card">
      <div class="card-header">
        <span class="card-title">Available Budget</span>
      </div>
      <div class="budget-value">{{ formatCurrency(budget) }}</div>
      <div class="slider-wrap">
        <input
          type="range"
          class="budget-slider"
          :min="10000"
          :max="500000"
          :step="5000"
          v-model.number="budget"
          @input="onBudgetChange"
        />
        <div class="slider-labels">
          <span>$10,000</span>
          <span>$500,000</span>
        </div>
      </div>
    </div>

    <div class="card">
      <div class="card-header">
        <span class="card-title">Recommended Items</span>
        <span class="count-badge">
          {{ recommendations.length }} items &middot; {{ formatCurrency(totalCost) }} total
        </span>
      </div>

      <div v-if="loading" class="loading">Loading recommendations...</div>

      <template v-else>
        <div class="utilization-row">
          <div class="progress-outer">
            <div
              class="progress-inner"
              :style="{
                width: utilizationPct + '%',
                background: parseFloat(utilizationPct) >= 80 ? '#d97706' : '#059669'
              }"
            ></div>
          </div>
          <span class="utilization-label">{{ utilizationPct }}%</span>
        </div>

        <div v-if="error" class="error">{{ error }}</div>

        <div v-if="recommendations.length === 0" class="empty-state">
          No items fit within the current budget. Try increasing the budget.
        </div>

        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>SKU</th>
                <th>Item Name</th>
                <th>Warehouse</th>
                <th>Trend</th>
                <th>Demand Gap</th>
                <th>Qty to Order</th>
                <th>Unit Cost</th>
                <th>Total Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendations" :key="item.sku">
                <td>{{ item.sku }}</td>
                <td>{{ item.name }}</td>
                <td>{{ item.warehouse }}</td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td class="demand-gap">
                  +{{ item.forecasted_demand - item.current_demand }} units
                </td>
                <td>{{ item.quantity_to_order }}</td>
                <td>{{ formatCurrency(item.unit_cost) }}</td>
                <td>{{ formatCurrency(item.total_cost) }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </template>
    </div>

    <div v-if="successOrder" class="success-banner">
      <span>
        Order {{ successOrder.order_number }} placed successfully! Expected delivery in 14 days.
      </span>
      <button class="close-btn" @click="successOrder = null">X</button>
    </div>

    <div class="action-row">
      <button
        class="place-order-btn"
        :disabled="recommendations.length === 0 || loading || placing"
        @click="placeOrder"
      >
        {{ placing ? 'Placing Order...' : 'Place Order' }}
      </button>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const budget = ref(100000)
    const recommendations = ref([])
    const loading = ref(false)
    const error = ref(null)
    const placing = ref(false)
    const successOrder = ref(null)
    let debounceTimer = null

    const formatCurrency = (val) =>
      '$' + val.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 })

    const totalCost = computed(() =>
      recommendations.value.reduce((sum, item) => sum + item.total_cost, 0)
    )

    const utilizationPct = computed(() =>
      Math.min((totalCost.value / budget.value) * 100, 100).toFixed(1)
    )

    const loadRecommendations = async () => {
      try {
        loading.value = true
        error.value = null
        recommendations.value = await api.getRestockingRecommendations(budget.value)
      } catch (err) {
        error.value = 'Failed to load recommendations: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const onBudgetChange = () => {
      clearTimeout(debounceTimer)
      debounceTimer = setTimeout(loadRecommendations, 300)
    }

    const placeOrder = async () => {
      placing.value = true
      try {
        const items = recommendations.value.map(r => ({
          sku: r.sku,
          name: r.name,
          quantity: r.quantity_to_order,
          unit_price: r.unit_cost
        }))
        const order = await api.createRestockingOrder({ budget: budget.value, items })
        successOrder.value = order
        await loadRecommendations()
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        placing.value = false
      }
    }

    onMounted(loadRecommendations)

    return {
      budget,
      recommendations,
      loading,
      error,
      placing,
      successOrder,
      totalCost,
      utilizationPct,
      formatCurrency,
      onBudgetChange,
      placeOrder
    }
  }
}
</script>

<style scoped>
.view-container {
  padding: 2rem;
}

.budget-value {
  font-size: 2rem;
  font-weight: 700;
  color: #2563eb;
  margin-bottom: 1rem;
}

.slider-wrap {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  cursor: pointer;
  height: 6px;
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.813rem;
  color: #64748b;
}

.count-badge {
  font-size: 0.875rem;
  color: #64748b;
  font-weight: 500;
}

.utilization-row {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  margin-bottom: 1rem;
}

.progress-outer {
  flex: 1;
  height: 8px;
  background: #e2e8f0;
  border-radius: 4px;
  overflow: hidden;
}

.progress-inner {
  height: 100%;
  border-radius: 4px;
  transition: width 0.3s ease, background 0.3s ease;
}

.utilization-label {
  font-size: 0.813rem;
  font-weight: 600;
  color: #334155;
  min-width: 3.5rem;
  text-align: right;
}

.demand-gap {
  color: #059669;
  font-weight: 500;
}

.empty-state {
  text-align: center;
  padding: 3rem;
  color: #64748b;
  font-size: 0.938rem;
}

.success-banner {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: #d1fae5;
  border: 1px solid #a7f3d0;
  color: #065f46;
  padding: 12px 16px;
  border-radius: 8px;
  margin-bottom: 1rem;
  font-size: 0.938rem;
  font-weight: 500;
}

.close-btn {
  background: none;
  border: none;
  color: #065f46;
  font-weight: 700;
  cursor: pointer;
  font-size: 0.875rem;
  padding: 0 0.25rem;
  line-height: 1;
}

.close-btn:hover {
  opacity: 0.7;
}

.action-row {
  display: flex;
  justify-content: flex-end;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  padding: 0.75rem 2rem;
  border: none;
  border-radius: 8px;
  font-weight: 600;
  cursor: pointer;
  font-size: 0.938rem;
  transition: opacity 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
</style>
