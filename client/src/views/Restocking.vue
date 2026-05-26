<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Success banner (shown after placing order) -->
      <div v-if="placedOrder" class="success-banner">
        <strong>{{ t('restocking.orderPlaced') }}</strong>
        — {{ t('restocking.orderNumber') }}: {{ placedOrder.order_number }}
        | {{ t('restocking.expectedDelivery') }}: {{ formatDate(placedOrder.expected_delivery) }}
      </div>

      <!-- Budget section (card) -->
      <div class="card budget-card">
        <div class="budget-header">
          <label class="budget-label">{{ t('restocking.budget') }}</label>
          <span class="budget-display">
            {{ currencySymbol }}{{ Math.round(budget).toLocaleString() }}
            {{ t('restocking.budgetOf') }}
            {{ currencySymbol }}{{ Math.round(maxBudget).toLocaleString() }}
          </span>
          <span class="items-selected-count">
            {{ t('restocking.itemsSelected', { count: selectedItems.length }) }}
          </span>
        </div>
        <input
          type="range"
          class="budget-slider"
          :min="0"
          :max="Math.round(maxBudget)"
          :step="100"
          v-model.number="budget"
        />
      </div>

      <!-- Empty state -->
      <div v-if="recommendations.length === 0" class="card">
        <p class="empty-state">{{ t('restocking.noItems') }}</p>
      </div>

      <!-- Recommendations table (card) -->
      <div v-else class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendations') }} ({{ selectedItems.length }} / {{ recommendations.length }})</h3>
        </div>
        <div class="table-container">
          <table class="restocking-table">
            <thead>
              <tr>
                <th>{{ t('restocking.table.priority') }}</th>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.inStock') }}</th>
                <th>{{ t('restocking.table.reorderPoint') }}</th>
                <th>{{ t('restocking.table.trend') }}</th>
                <th>{{ t('restocking.table.qtyToOrder') }}</th>
                <th>{{ t('restocking.table.estimatedCost') }}</th>
              </tr>
            </thead>
            <tbody>
              <!-- Selected items (within budget) -->
              <tr v-for="item in selectedItems" :key="item.sku" class="row-selected">
                <td><span :class="getPriorityClass(item)">{{ getPriorityLabel(item) }}</span></td>
                <td>{{ item.name }}</td>
                <td class="sku-cell">{{ item.sku }}</td>
                <td>{{ item.quantity_on_hand.toLocaleString() }}</td>
                <td>{{ item.reorder_point.toLocaleString() }}</td>
                <td><span :class="['badge', item.trend === 'increasing' ? 'success' : item.trend === 'decreasing' ? 'danger' : 'info']">{{ item.trend }}</span></td>
                <td><strong>{{ item.recommendedQty.toLocaleString() }}</strong></td>
                <td><strong>{{ currencySymbol }}{{ Math.round(item.estimatedCost).toLocaleString() }}</strong></td>
              </tr>
              <!-- Items outside budget (greyed out) -->
              <tr
                v-for="item in recommendations.filter(r => !selectedItems.includes(r))"
                :key="'unselected-' + item.sku"
                class="row-unselected"
              >
                <td><span :class="getPriorityClass(item)">{{ getPriorityLabel(item) }}</span></td>
                <td>{{ item.name }}</td>
                <td class="sku-cell">{{ item.sku }}</td>
                <td>{{ item.quantity_on_hand.toLocaleString() }}</td>
                <td>{{ item.reorder_point.toLocaleString() }}</td>
                <td><span :class="['badge', item.trend === 'increasing' ? 'success' : item.trend === 'decreasing' ? 'danger' : 'info']">{{ item.trend }}</span></td>
                <td>{{ item.recommendedQty.toLocaleString() }}</td>
                <td>{{ currencySymbol }}{{ Math.round(item.estimatedCost).toLocaleString() }}</td>
              </tr>
            </tbody>
          </table>
        </div>

        <!-- Footer: total + place order button -->
        <div class="order-footer">
          <div class="total-cost">
            {{ t('restocking.totalCost') }}:
            <strong>{{ currencySymbol }}{{ Math.round(totalSelectedCost).toLocaleString() }}</strong>
          </div>
          <button
            class="btn-primary place-order-btn"
            :disabled="selectedItems.length === 0 || orderPlacing"
            @click="placeOrder"
          >
            {{ orderPlacing ? t('common.loading') : t('restocking.placeOrder') }}
          </button>
        </div>
      </div>

    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()
    const { selectedLocation, selectedCategory, getCurrentFilters } = useFilters()

    const loading = ref(true)
    const error = ref(null)
    const allInventory = ref([])
    const allForecasts = ref([])
    const budget = ref(0)
    const orderPlacing = ref(false)
    const placedOrder = ref(null)

    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    const recommendations = computed(() => {
      // Build a lookup map: SKU -> inventory item
      const invMap = {}
      for (const item of allInventory.value) {
        invMap[item.sku] = item
      }

      const result = []
      for (const forecast of allForecasts.value) {
        const inv = invMap[forecast.item_sku]
        if (!inv) continue  // no matching inventory item, skip

        const isLowStock = inv.quantity_on_hand <= inv.reorder_point
        const isIncreasing = forecast.trend === 'increasing'

        // Include only if low stock OR increasing demand
        if (!isLowStock && !isIncreasing) continue

        // Recommended quantity: enough to meet forecasted demand and bring stock above 2x reorder point
        const qtyToMeetDemand = Math.max(0, forecast.forecasted_demand - inv.quantity_on_hand)
        const qtyToMeetReorder = Math.max(0, inv.reorder_point * 2 - inv.quantity_on_hand)
        const recommendedQty = Math.max(qtyToMeetDemand, qtyToMeetReorder, 1)
        const estimatedCost = recommendedQty * inv.unit_cost

        // Priority for sorting: 0 = urgent (both), 1 = low stock only, 2 = high demand only
        const priority = isLowStock && isIncreasing ? 0 : isLowStock ? 1 : 2

        result.push({
          sku: inv.sku,
          name: inv.name,
          category: inv.category,
          quantity_on_hand: inv.quantity_on_hand,
          reorder_point: inv.reorder_point,
          unit_cost: inv.unit_cost,
          trend: forecast.trend,
          recommendedQty,
          estimatedCost,
          priority,
          isLowStock,
          isIncreasing
        })
      }

      // Sort by priority ascending (urgent first)
      result.sort((a, b) => a.priority - b.priority)
      return result
    })

    // Total cost if all recommended items are ordered
    const maxBudget = computed(() => {
      return recommendations.value.reduce((sum, item) => sum + item.estimatedCost, 0)
    })

    // Items selected within budget: greedy inclusion by priority order
    const selectedItems = computed(() => {
      let remaining = budget.value
      const selected = []
      for (const item of recommendations.value) {
        if (item.estimatedCost <= remaining) {
          selected.push(item)
          remaining -= item.estimatedCost
        }
      }
      return selected
    })

    const totalSelectedCost = computed(() => {
      return selectedItems.value.reduce((sum, item) => sum + item.estimatedCost, 0)
    })

    const loadData = async () => {
      try {
        loading.value = true
        error.value = null
        placedOrder.value = null
        const filters = getCurrentFilters()
        // fetch inventory with warehouse/category filters; demand has no filters
        const [inv, forecasts] = await Promise.all([
          api.getInventory({ warehouse: filters.warehouse, category: filters.category }),
          api.getDemandForecasts()
        ])
        allInventory.value = inv
        allForecasts.value = forecasts
        // initialize budget to the max (all items) after data loads
        budget.value = maxBudget.value
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    watch([selectedLocation, selectedCategory], () => {
      loadData()
    })

    const placeOrder = async () => {
      if (selectedItems.value.length === 0) return
      try {
        orderPlacing.value = true
        const filters = getCurrentFilters()
        const payload = {
          items: selectedItems.value.map(item => ({
            sku: item.sku,
            name: item.name,
            quantity: item.recommendedQty,
            unit_price: item.unit_cost
          })),
          warehouse: filters.warehouse !== 'all' ? filters.warehouse : null
        }
        placedOrder.value = await api.createRestockingOrder(payload)
        // Reset budget to max so the page reflects the new state
        budget.value = maxBudget.value
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        orderPlacing.value = false
      }
    }

    const getPriorityLabel = (item) => {
      if (item.isLowStock && item.isIncreasing) return t('restocking.urgentBadge')
      if (item.isLowStock) return t('restocking.lowStockBadge')
      return t('restocking.highDemandBadge')
    }

    const getPriorityClass = (item) => {
      if (item.isLowStock && item.isIncreasing) return 'badge danger'
      if (item.isLowStock) return 'badge warning'
      return 'badge info'
    }

    const formatDate = (dateString) => {
      const { currentLocale } = useI18n()
      const locale = currentLocale.value === 'ja' ? 'ja-JP' : 'en-US'
      return new Date(dateString).toLocaleDateString(locale, {
        year: 'numeric', month: 'short', day: 'numeric'
      })
    }

    onMounted(loadData)

    return {
      t,
      loading,
      error,
      allInventory,
      allForecasts,
      budget,
      orderPlacing,
      placedOrder,
      currencySymbol,
      recommendations,
      maxBudget,
      selectedItems,
      totalSelectedCost,
      loadData,
      placeOrder,
      getPriorityLabel,
      getPriorityClass,
      formatDate
    }
  }
}
</script>

<style scoped>
.restocking {
  color: #e2e8f0;
}

.success-banner {
  background: #14532d;
  color: #86efac;
  padding: 1rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
}

.budget-card {
  margin-bottom: 1.25rem;
}

.budget-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.875rem;
  flex-wrap: wrap;
  gap: 0.5rem;
}

.budget-label {
  font-weight: 600;
  font-size: 0.938rem;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.budget-display {
  font-size: 1rem;
  font-weight: 600;
  color: #e2e8f0;
}

.items-selected-count {
  font-size: 0.875rem;
  color: #64748b;
}

.budget-slider {
  width: 100%;
  height: 6px;
  border-radius: 3px;
  outline: none;
  cursor: pointer;
  accent-color: #2563eb;
}

.restocking-table {
  table-layout: fixed;
  width: 100%;
}

.restocking-table th:nth-child(1) { width: 110px; }
.restocking-table th:nth-child(2) { width: auto; }
.restocking-table th:nth-child(3) { width: 110px; }
.restocking-table th:nth-child(4) { width: 90px; }
.restocking-table th:nth-child(5) { width: 110px; }
.restocking-table th:nth-child(6) { width: 100px; }
.restocking-table th:nth-child(7) { width: 110px; }
.restocking-table th:nth-child(8) { width: 130px; }

.sku-cell {
  font-family: monospace;
  color: #64748b;
}

.row-selected {
  opacity: 1;
}

.row-unselected {
  opacity: 0.4;
}

.empty-state {
  text-align: center;
  color: #64748b;
  padding: 2rem;
}

.order-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-top: 1rem;
  border-top: 1px solid #334155;
}

.total-cost {
  font-size: 1rem;
  color: #e2e8f0;
}

.btn-primary.place-order-btn {
  background: #2563eb;
  color: #ffffff;
  padding: 0.625rem 1.5rem;
  border: none;
  border-radius: 6px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.btn-primary.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary.place-order-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
</style>
