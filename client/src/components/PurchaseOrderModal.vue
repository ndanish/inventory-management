<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="isOpen && backlogItem" class="modal-overlay" @click="close">
        <div class="modal-container" @click.stop>
          <div class="modal-header">
            <h3 class="modal-title">{{ mode === 'create' ? 'Create Purchase Order' : 'Purchase Order Details' }}</h3>
            <button class="close-button" @click="close">
              <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
                <path d="M15 5L5 15M5 5L15 15" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
              </svg>
            </button>
          </div>

          <div class="modal-body">
            <!-- Backlog item summary shown in both modes -->
            <div class="item-summary">
              <div class="item-summary-info">
                <h4 class="item-name">{{ backlogItem.item_name }}</h4>
                <div class="item-sku">SKU: {{ backlogItem.item_sku }}</div>
              </div>
              <span class="priority-badge" :class="backlogItem.priority">
                {{ backlogItem.priority }} Priority
              </span>
            </div>

            <!-- Create mode: form -->
            <form v-if="mode === 'create'" @submit.prevent="handleSubmit" novalidate>
              <!-- Shortage summary for context -->
              <div class="shortage-callout">
                <div class="shortage-callout-label">Shortage Amount</div>
                <div class="shortage-callout-value">{{ shortage }} units</div>
              </div>

              <div class="form-group">
                <label class="form-label" for="po-supplier">Supplier Name</label>
                <input
                  id="po-supplier"
                  v-model="form.supplier"
                  type="text"
                  class="form-input"
                  :class="{ 'input-error': errors.supplier }"
                  placeholder="Enter supplier name"
                  autocomplete="organization"
                />
                <span v-if="errors.supplier" class="error-msg">{{ errors.supplier }}</span>
              </div>

              <div class="form-group">
                <label class="form-label" for="po-quantity">Quantity to Order</label>
                <input
                  id="po-quantity"
                  v-model.number="form.quantity"
                  type="number"
                  min="1"
                  class="form-input"
                  :class="{ 'input-error': errors.quantity }"
                />
                <span v-if="errors.quantity" class="error-msg">{{ errors.quantity }}</span>
              </div>

              <div class="form-group">
                <label class="form-label" for="po-delivery">Estimated Delivery Date</label>
                <input
                  id="po-delivery"
                  v-model="form.estimated_delivery"
                  type="date"
                  class="form-input"
                  :class="{ 'input-error': errors.estimated_delivery }"
                />
                <span v-if="errors.estimated_delivery" class="error-msg">{{ errors.estimated_delivery }}</span>
              </div>
            </form>

            <!-- View mode: read-only PO details -->
            <div v-else-if="mode === 'view' && backlogItem.purchase_order" class="po-details">
              <div class="info-grid">
                <div class="info-item">
                  <div class="info-label">PO ID</div>
                  <div class="info-value po-id">{{ backlogItem.purchase_order.id }}</div>
                </div>
                <div class="info-item">
                  <div class="info-label">Supplier</div>
                  <div class="info-value">{{ backlogItem.purchase_order.supplier }}</div>
                </div>
                <div class="info-item">
                  <div class="info-label">Quantity</div>
                  <div class="info-value">{{ backlogItem.purchase_order.quantity }} units</div>
                </div>
                <div class="info-item">
                  <div class="info-label">Status</div>
                  <div class="info-value">
                    <span class="badge" :class="getStatusBadgeClass(backlogItem.purchase_order.status)">
                      {{ backlogItem.purchase_order.status }}
                    </span>
                  </div>
                </div>
                <div class="info-item">
                  <div class="info-label">Estimated Delivery</div>
                  <div class="info-value">{{ formatDate(backlogItem.purchase_order.estimated_delivery) }}</div>
                </div>
                <div class="info-item">
                  <div class="info-label">Created At</div>
                  <div class="info-value">{{ formatDate(backlogItem.purchase_order.created_at) }}</div>
                </div>
              </div>
            </div>
          </div>

          <div class="modal-footer">
            <button class="btn-secondary" @click="close">
              {{ mode === 'create' ? 'Cancel' : 'Close' }}
            </button>
            <button
              v-if="mode === 'create'"
              class="btn-primary"
              @click="handleSubmit"
            >
              Create Purchase Order
            </button>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script>
import { ref, computed, watch } from 'vue'

export default {
  name: 'PurchaseOrderModal',
  props: {
    isOpen: {
      type: Boolean,
      default: false
    },
    backlogItem: {
      type: Object,
      default: null
    },
    // 'create' shows the form; 'view' shows existing PO data read-only
    mode: {
      type: String,
      default: 'create'
    }
  },
  emits: ['close', 'po-created'],
  setup(props, { emit }) {
    // Computed shortage amount used to pre-fill the quantity field
    const shortage = computed(() => {
      if (!props.backlogItem) return 0
      return props.backlogItem.quantity_needed - props.backlogItem.quantity_available
    })

    // Form state — quantity pre-filled with the shortage amount
    const form = ref({
      supplier: '',
      quantity: 0,
      estimated_delivery: ''
    })

    // Field-level error messages
    const errors = ref({
      supplier: '',
      quantity: '',
      estimated_delivery: ''
    })

    // Reset form and errors whenever the modal opens in create mode so
    // stale data from a previous session is not shown
    watch(
      () => props.isOpen,
      (open) => {
        if (open && props.mode === 'create') {
          form.value = {
            supplier: '',
            quantity: shortage.value,
            estimated_delivery: ''
          }
          errors.value = { supplier: '', quantity: '', estimated_delivery: '' }
        }
      }
    )

    // Also reset quantity when backlogItem changes (shortage may differ)
    watch(
      () => props.backlogItem,
      () => {
        if (props.mode === 'create') {
          form.value.quantity = shortage.value
        }
      }
    )

    const validate = () => {
      let valid = true

      errors.value.supplier = ''
      errors.value.quantity = ''
      errors.value.estimated_delivery = ''

      if (!form.value.supplier || !form.value.supplier.trim()) {
        errors.value.supplier = 'Supplier name is required.'
        valid = false
      }

      if (!form.value.quantity || form.value.quantity < 1) {
        errors.value.quantity = 'Quantity must be at least 1.'
        valid = false
      }

      if (!form.value.estimated_delivery) {
        errors.value.estimated_delivery = 'Estimated delivery date is required.'
        valid = false
      }

      return valid
    }

    const handleSubmit = () => {
      if (!validate()) return

      // Generate a simple unique PO ID using a timestamp
      const poId = `PO-${Date.now()}`
      const now = new Date().toISOString()

      const poData = {
        id: poId,
        backlog_item_id: props.backlogItem.id,
        supplier: form.value.supplier.trim(),
        quantity: form.value.quantity,
        estimated_delivery: form.value.estimated_delivery,
        status: 'Pending',
        created_at: now
      }

      emit('po-created', poData)
    }

    const close = () => {
      emit('close')
    }

    const formatDate = (dateString) => {
      if (!dateString) return 'N/A'
      const date = new Date(dateString)
      if (isNaN(date.getTime())) return 'N/A'
      return date.toLocaleDateString('en-US', {
        year: 'numeric',
        month: 'long',
        day: 'numeric'
      })
    }

    // Map PO status values to badge modifier classes
    const getStatusBadgeClass = (status) => {
      if (!status) return ''
      const s = status.toLowerCase()
      if (s === 'pending') return 'warning'
      if (s === 'approved' || s === 'delivered') return 'success'
      if (s === 'cancelled') return 'danger'
      return ''
    }

    return {
      shortage,
      form,
      errors,
      handleSubmit,
      close,
      formatDate,
      getStatusBadgeClass
    }
  }
}
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
  padding: 1rem;
}

.modal-container {
  background: white;
  border-radius: 12px;
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.15);
  max-width: 560px;
  width: 100%;
  max-height: 90vh;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
}

.modal-title {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.close-button {
  background: none;
  border: none;
  color: #64748b;
  cursor: pointer;
  padding: 0.5rem;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 6px;
  transition: all 0.15s ease;
}

.close-button:hover {
  background: #f1f5f9;
  color: #0f172a;
}

.modal-body {
  flex: 1;
  overflow-y: auto;
  padding: 1.5rem 2rem;
}

/* Backlog item summary strip at the top of the modal body */
.item-summary {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding: 1rem 1.25rem;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  margin-bottom: 1.5rem;
}

.item-summary-info {
  flex: 1;
  min-width: 0;
}

.item-name {
  font-size: 1rem;
  font-weight: 700;
  color: #0f172a;
  margin: 0 0 0.25rem 0;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.item-sku {
  font-size: 0.813rem;
  color: #64748b;
  font-family: 'Monaco', 'Courier New', monospace;
}

/* Priority badge — mirrors BacklogDetailModal */
.priority-badge {
  padding: 0.375rem 0.875rem;
  border-radius: 6px;
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
  flex-shrink: 0;
}

.priority-badge.high {
  background: #fecaca;
  color: #991b1b;
}

.priority-badge.medium {
  background: #fed7aa;
  color: #92400e;
}

.priority-badge.low {
  background: #dbeafe;
  color: #1e40af;
}

/* Compact shortage callout above the form */
.shortage-callout {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0.875rem 1.25rem;
  background: #fef2f2;
  border: 2px solid #fecaca;
  border-radius: 10px;
  margin-bottom: 1.5rem;
}

.shortage-callout-label {
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.shortage-callout-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #dc2626;
}

/* Form layout */
.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
  margin-bottom: 1.25rem;
}

.form-group:last-child {
  margin-bottom: 0;
}

.form-label {
  font-size: 0.813rem;
  font-weight: 600;
  color: #475569;
  text-transform: uppercase;
  letter-spacing: 0.04em;
}

.form-input {
  padding: 0.625rem 0.875rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.938rem;
  color: #0f172a;
  font-family: inherit;
  background: white;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
  width: 100%;
  box-sizing: border-box;
}

.form-input:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.form-input.input-error {
  border-color: #ef4444;
}

.form-input.input-error:focus {
  box-shadow: 0 0 0 3px rgba(239, 68, 68, 0.1);
}

.error-msg {
  font-size: 0.813rem;
  color: #ef4444;
  font-weight: 500;
}

/* Read-only PO details grid */
.po-details {
  margin-top: 0.25rem;
}

.info-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1.5rem;
}

.info-item {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.info-label {
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.info-value {
  font-size: 0.938rem;
  color: #0f172a;
  font-weight: 500;
}

.info-value.po-id {
  font-family: 'Monaco', 'Courier New', monospace;
  color: #2563eb;
}

/* Status badges */
.badge {
  display: inline-block;
  padding: 0.25rem 0.625rem;
  border-radius: 4px;
  font-size: 0.813rem;
  font-weight: 600;
}

.badge.warning {
  background: #fffbeb;
  color: #92400e;
  border: 1px solid #fed7aa;
}

.badge.success {
  background: #f0fdf4;
  color: #14532d;
  border: 1px solid #bbf7d0;
}

.badge.danger {
  background: #fef2f2;
  color: #991b1b;
  border: 1px solid #fecaca;
}

/* Footer */
.modal-footer {
  padding: 1.5rem;
  border-top: 1px solid #e2e8f0;
  display: flex;
  justify-content: flex-end;
  gap: 0.75rem;
}

.btn-secondary {
  padding: 0.625rem 1.25rem;
  background: #f1f5f9;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-weight: 500;
  font-size: 0.875rem;
  color: #334155;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-secondary:hover {
  background: #e2e8f0;
  border-color: #cbd5e1;
}

.btn-primary {
  padding: 0.625rem 1.25rem;
  background: #3b82f6;
  border: none;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.875rem;
  color: white;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-primary:hover {
  background: #2563eb;
  box-shadow: 0 2px 4px rgba(59, 130, 246, 0.3);
}

/* Modal transition animations — same as BacklogDetailModal */
.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.2s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}

.modal-enter-active .modal-container,
.modal-leave-active .modal-container {
  transition: transform 0.2s ease;
}

.modal-enter-from .modal-container,
.modal-leave-to .modal-container {
  transform: scale(0.95);
}
</style>
