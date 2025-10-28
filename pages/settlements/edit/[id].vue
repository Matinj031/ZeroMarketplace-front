<template>
  <v-card class="spike-card" elevation="10">
    <v-card-item>
      <!-- Header -->
      <div class="d-flex justify-space-between align-center mb-6">
        <h5 class="text-20">ویرایش تسویه حساب</h5>
        <v-btn color="error" variant="flat" rounded="pill" to="/settlements">
          <Icon icon="solar:arrow-right-linear" height="18" class="ml-2" />
          بازگشت به لیست
        </v-btn>
      </div>

      <!-- Loading -->
      <Loading :loading="loading" />

      <!-- Form -->
      <v-form v-if="!loading" ref="formRef" v-model="valid" @submit.prevent="submitForm">
        <div class="bg-bglight pa-6 rounded-md">
          <v-row>
            <!-- Type Display -->
            <v-col cols="12" md="6">
              <v-label class="font-weight-semibold pb-2">نوع تسویه</v-label>
              <v-text-field v-model="typeDisplay" variant="outlined" density="compact" readonly />
            </v-col>

            <!-- Code Display -->
            <v-col cols="12" md="6">
              <v-label class="font-weight-semibold pb-2">کد تسویه</v-label>
              <v-text-field v-model="settlement.code" variant="outlined" density="compact" readonly />
            </v-col>

            <!-- Reference Display -->
            <v-col cols="12">
              <v-label class="font-weight-semibold pb-2">سند مرجع</v-label>
              <v-text-field v-model="referenceDisplay" variant="outlined" density="compact" readonly />
            </v-col>

            <!-- Amount Display -->
            <v-col cols="12" md="6">
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
              <v-text-field v-model="settlement.payment.cash" type="number" variant="outlined" density="compact"
                @input="calculateRemaining" suffix="تومان" />
            </v-col>

            <!-- Bank Payment -->
            <v-col cols="12" md="4">
              <v-label class="font-weight-semibold pb-2">کارتخوان</v-label>
              <v-text-field v-model="settlement.payment.bank" type="number" variant="outlined" density="compact"
                @input="calculateRemaining" suffix="تومان" />
            </v-col>

            <!-- Credit Payment -->
            <v-col cols="12" md="4">
              <v-label class="font-weight-semibold pb-2">نسیه</v-label>
              <v-text-field v-model="settlement.payment.credit" type="number" variant="outlined" density="compact"
                @input="calculateRemaining" suffix="تومان" />
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
          <v-btn flat color="primary" type="submit" :loading="saving" :disabled="remainingAmount !== '0'"
            rounded="pill">
            ذخیره تغییرات
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
import { useRouter, useRoute } from 'vue-router';
import { useNuxtApp } from '#app';
import { Icon } from '@iconify/vue';
import Loading from '~/components/Loading.vue';
import { formatters } from '~/utils/formatters';

definePageMeta({
  layout: 'admin-spike',
  middleware: 'auth',
  requiresAuth: true,
  requiresRole: 'admin'
});

const router = useRouter();
const route = useRoute();
const { $notify } = useNuxtApp();
const formRef = ref(null);
const valid = ref(false);
const loading = ref(true);
const saving = ref(false);
const totalAmount = ref('0');
const referenceDisplay = ref('');

const settlement = ref({
  _id: '',
  code: '',
  type: '',
  _reference: '',
  payment: {
    cash: '0',
    bank: '0',
    credit: '0',
    remaining: '0'
  }
});

const typeDisplay = computed(() => {
  return settlement.value.type === 'receive' ? 'دریافت' : 'پرداخت';
});

const remainingAmount = computed(() => {
  const total = Number(totalAmount.value) || 0;
  const cash = Number(settlement.value.payment.cash) || 0;
  const bank = Number(settlement.value.payment.bank) || 0;
  const credit = Number(settlement.value.payment.credit) || 0;
  const remaining = total - cash - bank - credit;
  return String(remaining);
});

const calculateRemaining = () => {
  settlement.value.payment.remaining = remainingAmount.value;
};

const loadSettlement = async () => {
  loading.value = true;
  try {
    const data = await useApiService.get(`settlements/${route.params.id}`);

    settlement.value = {
      _id: data._id,
      code: data.code,
      type: data.type,
      _reference: data._reference,
      payment: {
        cash: String(data.payment?.cash || 0),
        bank: String(data.payment?.bank || 0),
        credit: String(data.payment?.credit || 0),
        remaining: String(data.payment?.remaining || 0)
      }
    };

    // Load reference details
    if (data._reference) {
      await loadReferenceDetails(data._reference);
    }

    $notify('اطلاعات تسویه بارگذاری شد', 'success');
  } catch (error) {
    $notify('مشکلی در بارگذاری اطلاعات پیش آمد', 'error');
    router.push('/settlements');
  } finally {
    loading.value = false;
  }
};

const loadReferenceDetails = async (referenceId) => {
  try {
    // Try different endpoints to find the reference
    const endpoints = ['sales-invoices', 'purchase-invoices', 'accounting-documents'];

    for (const endpoint of endpoints) {
      try {
        const data = await useApiService.get(`${endpoint}/${referenceId}`);
        if (data) {
          totalAmount.value = String(data.total || 0);
          referenceDisplay.value = `${data.code || referenceId} - ${formatters.price(data.total || 0)} تومان`;
          break;
        }
      } catch (e) {
        // Continue to next endpoint
      }
    }
  } catch (error) {
    console.error('Error loading reference:', error);
  }
};

const submitForm = async () => {
  const { valid: isValid } = await formRef.value.validate();
  if (!isValid) return;

  if (remainingAmount.value !== '0') {
    $notify('مبلغ باقیمانده باید صفر باشد', 'error');
    return;
  }

  saving.value = true;
  try {
    const payload = {
      type: settlement.value.type,
      _reference: settlement.value._reference,
      payment: {
        cash: Number(settlement.value.payment.cash),
        bank: Number(settlement.value.payment.bank),
        credit: Number(settlement.value.payment.credit),
        remaining: Number(remainingAmount.value),
        cashAccounts: [],
        bankAccounts: []
      }
    };

    await useApiService.put(`settlements/${settlement.value._id}`, payload);
    $notify('تسویه با موفقیت ویرایش شد', 'success');
    router.push('/settlements');
  } catch (error) {
    $notify('مشکلی در ویرایش تسویه پیش آمد', 'error');
  } finally {
    saving.value = false;
  }
};

onMounted(() => {
  loadSettlement();
});
</script>
