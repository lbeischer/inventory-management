<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Budget Card -->
      <div class="card budget-card">
        <div class="budget-layout">
          <div class="budget-slider-section">
            <label class="budget-label">{{ t('restocking.budget') }}</label>
            <input
              type="range"
              min="0"
              max="500000"
              step="1000"
              v-model.number="budget"
              class="budget-slider"
            />
            <div class="budget-amount">
              {{ budget.toLocaleString('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 0 }) }}
            </div>
          </div>
          <div class="budget-stats">
            <div class="budget-stat">
              <div class="budget-stat-label">{{ t('restocking.budget') }}</div>
              <div class="budget-stat-value">
                {{ budget.toLocaleString('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 0 }) }}
              </div>
            </div>
            <div class="budget-stat">
              <div class="budget-stat-label">Remaining</div>
              <div class="budget-stat-value" :class="remaining >= 0 ? 'positive' : 'negative'">
                {{ remaining.toLocaleString('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 0 }) }}
              </div>
            </div>
            <div class="budget-stat">
              <div class="budget-stat-label">Items Selected</div>
              <div class="budget-stat-value">{{ selectedItems.size }}</div>
            </div>
          </div>
        </div>
      </div>

      <!-- Success Banner -->
      <div v-if="orderSuccess" class="success-banner">
        {{ t('restocking.orderPlaced') }}
      </div>

      <!-- Recommendations Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendedItems') }} ({{ recommendations.length }})</h3>
        </div>

        <div v-if="recommendations.length === 0" class="empty-state">
          {{ t('restocking.noRecommendations') }}
        </div>
        <div v-else class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th class="col-check"></th>
                <th class="col-sku">SKU</th>
                <th class="col-name">Item Name</th>
                <th class="col-trend">Trend</th>
                <th class="col-qty">Restock Qty</th>
                <th class="col-cost">Unit Cost</th>
                <th class="col-total">Item Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="rec in recommendations"
                :key="rec.item_sku"
                :class="{ 'row-dim': !selectedItems.has(rec.item_sku) && !canAfford(rec) }"
              >
                <td class="col-check">
                  <input
                    type="checkbox"
                    :checked="selectedItems.has(rec.item_sku)"
                    @change="toggleItem(rec.item_sku)"
                  />
                </td>
                <td class="col-sku"><strong>{{ rec.item_sku }}</strong></td>
                <td class="col-name">{{ rec.item_name }}</td>
                <td class="col-trend">
                  <span :class="['badge', rec.trend]">{{ rec.trend }}</span>
                </td>
                <td class="col-qty">{{ rec.gap.toLocaleString() }}</td>
                <td class="col-cost">{{ rec.unit_cost.toLocaleString('en-US', { style: 'currency', currency: 'USD' }) }}</td>
                <td class="col-total">
                  <strong>{{ rec.item_cost.toLocaleString('en-US', { style: 'currency', currency: 'USD' }) }}</strong>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Place Order Button -->
      <button
        class="place-order-btn"
        :disabled="selectedItems.size === 0 || placing"
        @click="placeOrder"
      >
        {{ placing ? 'Placing...' : t('restocking.placeOrder') }}
      </button>

    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const router = useRouter()
    const { t } = useI18n()

    const loading = ref(true)
    const error = ref(null)
    const placing = ref(false)
    const orderSuccess = ref(false)

    const budget = ref(50000)
    const allForecasts = ref([])
    const inventoryMap = ref({})
    const selectedItems = ref(new Set())

    // Build recommendations: cross-ref forecasts with inventory, keep gap > 0, sort by gap desc
    const recommendations = computed(() => {
      return allForecasts.value
        .map(forecast => {
          const gap = forecast.forecasted_demand - forecast.current_demand
          const unit_cost = inventoryMap.value[forecast.item_sku] ?? 0
          const item_cost = gap * unit_cost
          return { ...forecast, unit_cost, gap, item_cost }
        })
        .filter(r => r.gap > 0)
        .sort((a, b) => b.gap - a.gap)
    })

    // Greedy auto-selection within budget
    const autoSelected = computed(() => {
      const selected = new Set()
      let running = 0
      for (const rec of recommendations.value) {
        if (running + rec.item_cost <= budget.value) {
          selected.add(rec.item_sku)
          running += rec.item_cost
        }
      }
      return selected
    })

    const totalCost = computed(() => {
      return recommendations.value
        .filter(r => selectedItems.value.has(r.item_sku))
        .reduce((sum, r) => sum + r.item_cost, 0)
    })

    const remaining = computed(() => budget.value - totalCost.value)

    // Whether adding an unselected item would exceed budget
    const canAfford = (rec) => {
      if (selectedItems.value.has(rec.item_sku)) return true
      return totalCost.value + rec.item_cost <= budget.value
    }

    const toggleItem = (sku) => {
      const next = new Set(selectedItems.value)
      if (next.has(sku)) {
        next.delete(sku)
      } else {
        next.add(sku)
      }
      selectedItems.value = next
    }

    // When budget changes, re-run greedy fill
    watch(budget, () => {
      selectedItems.value = new Set(autoSelected.value)
    })

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [forecastsData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({})
        ])

        allForecasts.value = forecastsData

        // Build sku -> unit_cost map
        const map = {}
        for (const item of inventoryData) {
          map[item.sku] = item.unit_cost
        }
        inventoryMap.value = map

        // Initialize selection from autoSelected
        selectedItems.value = new Set(autoSelected.value)
      } catch (err) {
        error.value = 'Failed to load restocking data: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (selectedItems.value.size === 0 || placing.value) return
      placing.value = true
      try {
        const selectedRecs = recommendations.value.filter(r => selectedItems.value.has(r.item_sku))
        const items = selectedRecs.map(r => ({
          sku: r.item_sku,
          name: r.item_name,
          quantity: r.gap,
          unit_price: r.unit_cost,
          trend: r.trend
        }))
        await api.submitRestockingOrder({ items, total_value: totalCost.value })
        orderSuccess.value = true
        setTimeout(() => {
          router.push('/orders')
        }, 2000)
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
        console.error(err)
      } finally {
        placing.value = false
      }
    }

    onMounted(loadData)

    return {
      t,
      loading,
      error,
      placing,
      orderSuccess,
      budget,
      recommendations,
      selectedItems,
      totalCost,
      remaining,
      canAfford,
      toggleItem,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding-bottom: 2rem;
}

.budget-card {
  margin-bottom: 1.25rem;
}

.budget-layout {
  display: flex;
  align-items: flex-start;
  gap: 2rem;
}

.budget-slider-section {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #475569;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.budget-slider {
  width: 100%;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
}

.budget-amount {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-stats {
  display: flex;
  gap: 1.5rem;
  flex-shrink: 0;
}

.budget-stat {
  text-align: center;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 0.875rem 1.25rem;
  min-width: 130px;
}

.budget-stat-label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: 0.375rem;
}

.budget-stat-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-stat-value.positive {
  color: #059669;
}

.budget-stat-value.negative {
  color: #dc2626;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  border-radius: 8px;
  padding: 1rem 1.25rem;
  font-weight: 600;
  font-size: 0.938rem;
  margin-bottom: 1.25rem;
}

.restock-table {
  table-layout: fixed;
  width: 100%;
}

.col-check {
  width: 40px;
}

.col-sku {
  width: 110px;
}

.col-name {
  width: auto;
}

.col-trend {
  width: 110px;
}

.col-qty {
  width: 110px;
}

.col-cost {
  width: 120px;
}

.col-total {
  width: 130px;
}

.row-dim {
  opacity: 0.4;
}

.empty-state {
  text-align: center;
  padding: 3rem;
  color: #64748b;
  font-size: 0.938rem;
}

.place-order-btn {
  display: block;
  width: 100%;
  padding: 0.875rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
  margin-top: 1.25rem;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}
</style>
