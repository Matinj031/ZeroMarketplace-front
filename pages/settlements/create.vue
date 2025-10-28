<template>
  <v-card class="spike-card" elevation="10">
    <v-card-item>
      <!-- Header -->
      <div class="d-flex justify-space-between align-center mb-6">
        <h5 class="text-20">افزودن تسویه حساب جدید</h5>
        <v-btn color="error" variant="flat" rounded="pill" to="/settlements">
          <Icon icon="solar:arrow-right-linear" height="18" class="ml-2" />
          بازگشت به لیست
        </v-btn>
      </div>

      <!-- Form -->
      <v-form ref="formRef" v-model="valid" @submit.prevent="submitForm">
        <div class="bg-bglight pa-6 rounded-md">
          <v-row>
            <!-- Reference Type -->
            <v-col cols="12" md="6">
              <v-label class="font-weight-semibold pb-2">نوع فاکتور</v-label>
              <v-select v-model="settlement.type" :items="referenceTypeOptions" item-title="text" item-value="value"
                variant="outlined" density="compact" :rules="[rules.required]"
                @update:model-value="onReferenceTypeChange" />
            </v-col>

            <!-- Reference Selection (Invoice) -->
            <v-col cols="12" md="6">
              <v-label class="font-weight-semibold pb-2">{{ referenceTypeLabel }}</v-label>
              <v-autocomplete v-model="settlement._reference" :items="referenceItems" item-title="displayText"
                item-value="_id" variant="outlined" density="compact" :loading="loadingReferences"
                :rules="[rules.required]" @update:model-value="onReferenceSelect" />
            </v-col>

            <!-- Amount Display -->
            <v-col cols="12" md="6" v-if="settlement._reference">
              <v-label class="font-weight-semibold pb-2">مبلغ کل</v-label>
              <v-text-field v-model="totalAmount" variant="outlined" density="compact" readonly suffix="تومان" />
            </v-col>
          </v-row>

          <!-- Payment Section -->
          <v-divider class="my-6" />
          <h6 class="text-h6 mb-4">جزئیات پرداخت</h6>

          <v-row>
            <!-- Cash Payment -->
            <v-col cols="12" md="4">
              <v-label class="font-weight-semibold pb-2">نقد</v-label>
              <v-text-field v-model.number="settlement.payment.cash" type="number" variant="outlined" density="compact"
                @input="calculateRemaining" suffix="تومان" min="0" />
            </v-col>

            <!-- Bank Payment -->
            <v-col cols="12" md="4">
              <v-label class="font-weight-semibold pb-2">کارتخوان</v-label>
              <v-text-field v-model.number="settlement.payment.bank" type="number" variant="outlined" density="compact"
                @input="calculateRemaining" suffix="تومان" min="0" />
            </v-col>

            <!-- Credit Payment -->
            <v-col cols="12" md="4">
              <v-label class="font-weight-semibold pb-2">نسیه</v-label>
              <v-text-field v-model.number="settlement.payment.credit" type="number" variant="outlined"
                density="compact" @input="calculateRemaining" suffix="تومان" min="0" />
            </v-col>

            <!-- Remaining Amount -->
            <v-col cols="12">
              <v-label class="font-weight-semibold pb-2">باقیمانده</v-label>
              <v-text-field v-model="remainingAmount" variant="outlined" density="compact" readonly
                :color="remainingAmount === '0' ? 'success' : 'error'" suffix="تومان" />
              <p v-if="remainingAmount !== '0'" class="text-error text-caption mt-1">
                مبلغ باقیمانده باید صفر باشد
              </p>
            </v-col>
          </v-row>
        </div>

        <!-- Actions -->
        <div class="d-flex align-center justify-end ga-3 mt-6">
          <v-btn flat color="primary" type="submit" :loading="loading" :disabled="remainingAmount !== '0'"
            rounded="pill">
            ذخیره تسویه
          </v-btn>
          <v-btn flat color="error" to="/settlements" rounded="pill">
            انصراف
          </v-btn>
        </div>
      </v-form>
    </v-card-item>
  </v-card>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue';
import { useRouter } from 'vue-router';
import { useNuxtApp } from '#app';
import { Icon } from '@iconify/vue';
import { formatters } from '~/utils/formatters';

definePageMeta({
  layout: 'admin-spike',
  middleware: 'auth',
  requiresAuth: true,
  requiresRole: 'admin'
});

const router = useRouter();
const { $notify } = useNuxtApp();
const formRef = ref(null);
const valid = ref(false);
const loading = ref(false);
const loadingReferences = ref(false);
const referenceItems = ref([]);
const totalAmount = ref('0');

const accounts = ref([]);
const defaultCashAccount = ref(null);
const defaultBankAccount = ref(null);

const settlement = ref({
  type: 'sales-invoice',
  _reference: '',
  payment: {
    cash: 0,
    bank: 0,
    credit: 0,
    remaining: 0,
    cashAccounts: [],
    bankAccounts: [],
    distributedCash: false,
    distributedBank: false
  }
});

const referenceTypeOptions = [
  { text: 'فاکتور فروش', value: 'sales-invoice' },
  { text: 'فاکتور خرید', value: 'purchase-invoice' }
];

const referenceTypeLabel = computed(() => {
  const option = referenceTypeOptions.find(opt => opt.value === settlement.value.type);
  return option ? option.text : 'سند مرجع';
});

const remainingAmount = computed(() => {
  const total = Number(totalAmount.value) || 0;
  const cash = Number(settlement.value.payment.cash) || 0;
  const bank = Number(settlement.value.payment.bank) || 0;
  const credit = Number(settlement.value.payment.credit) || 0;
  const remaining = total - cash - bank - credit;
  return String(remaining);
});

const rules = {
  required: (value) => !!value || 'این فیلد الزامی است'
};

const loadAccounts = async () => {
  try {
    const search = new URLSearchParams();
    search.set('types', ['bank', 'cash']);
    search.set('perPage', 100);

    const data = await useApiService.get(`accounts?${search}`);
    accounts.value = data.list || [];

    // Find default accounts
    defaultCashAccount.value = accounts.value.find(acc => acc.type === '1' && acc.defaultFor);
    defaultBankAccount.value = accounts.value.find(acc => acc.type === '2' && acc.defaultFor);

    // Initialize account arrays
    settlement.value.payment.cashAccounts = accounts.value
      .filter(acc => acc.type === '1')
      .map(acc => ({
        _account: acc._id,
        amount: 0,
        default: acc.defaultFor || false,
        title: acc.title
      }));

    settlement.value.payment.bankAccounts = accounts.value
      .filter(acc => acc.type === '2')
      .map(acc => ({
        _account: acc._id,
        amount: 0,
        default: acc.defaultFor || false,
        title: acc.title
      }));
  } catch (error) {
    $notify('مشکلی در بارگذاری حساب‌ها پیش آمد', 'error');
  }
};

const onReferenceTypeChange = async () => {
  settlement.value._reference = '';
  totalAmount.value = '0';
  await loadReferences();
};

const loadReferences = async () => {
  if (!settlement.value.type) return;

  loadingReferences.value = true;
  try {
    const endpoint = settlement.value.type === 'sales-invoice' ? 'sales-invoices' : 'purchase-invoices';

    const search = new URLSearchParams();
    search.set('perPage', 100);
    search.set('statuses', [1, 2]);

    const data = await useApiService.get(`${endpoint}?${search}`);
    referenceItems.value = (data.list || []).map(item => ({
      ...item,
      displayText: `${item.code || item._id} - ${formatters.price(item.total || 0)} تومان`
    }));
  } catch (error) {
    $notify('مشکلی در بارگذاری لیست پیش آمد', 'error');
  } finally {
    loadingReferences.value = false;
  }
};

const onReferenceSelect = () => {
  const selected = referenceItems.value.find(item => item._id === settlement.value._reference);
  if (selected) {
    totalAmount.value = String(selected.total || 0);
    calculateRemaining();
  }
};

const calculateRemaining = () => {
  settlement.value.payment.remaining = remainingAmount.value;
};

const submitForm = async () => {
  const { valid: isValid } = await formRef.value.validate();
  if (!isValid) return;

  if (remainingAmount.value !== '0') {
    $notify('مبلغ باقیمانده باید صفر باشد', 'error');
    return;
  }

  if (!defaultCashAccount.value && Number(settlement.value.payment.cash) > 0) {
    $notify('حساب صندوق پیش‌فرض تنظیم نشده است', 'error');
    return;
  }

  if (!defaultBankAccount.value && Number(settlement.value.payment.bank) > 0) {
    $notify('حساب بانکی پیش‌فرض تنظیم نشده است', 'error');
    return;
  }

  loading.value = true;
  try {
    // Update account amounts based on payment
    const cashAmount = Number(settlement.value.payment.cash);
    const bankAmount = Number(settlement.value.payment.bank);

    // Set cash account amounts
    const cashAccounts = settlement.value.payment.cashAccounts.map(acc => {
      if (acc.default && cashAmount > 0) {
        return { ...acc, amount: cashAmount };
      }
      return { ...acc, amount: 0 };
    });

    // Set bank account amounts
    const bankAccounts = settlement.value.payment.bankAccounts.map(acc => {
      if (acc.default && bankAmount > 0) {
        return { ...acc, amount: bankAmount };
      }
      return { ...acc, amount: 0 };
    });

    const payload = {
      type: settlement.value.type,
      _reference: settlement.value._reference,
      payment: {
        cash: cashAmount,
        bank: bankAmount,
        credit: Number(settlement.value.payment.credit),
        remaining: Number(remainingAmount.value),
        cashAccounts: cashAccounts,
        bankAccounts: bankAccounts,
        distributedCash: false,
        distributedBank: false
      }
    };

    await useApiService.post('settlements', payload);
    $notify('تسویه با موفقیت ایجاد شد', 'success');
    router.push('/settlements');
  } catch (error) {
    console.error('Settlement creation error:', error);
    const message = error?.response?._data?.message || error?.message;
    $notify(message || 'مشکلی در ایجاد تسویه پیش آمد', 'error');
  } finally {
    loading.value = false;
  }
};

onMounted(async () => {
  await loadAccounts();
  await loadReferences();
});
</script>
