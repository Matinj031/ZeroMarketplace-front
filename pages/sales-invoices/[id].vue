<template>
    <v-card class="spike-card" elevation="10">
        <v-card-item>
            <!-- Header -->
            <div class="d-flex justify-space-between align-center mb-6">
                <h5 class="text-20">جزئیات فاکتور فروش</h5>
                <v-btn color="error" variant="flat" rounded="pill" to="/sales-invoices">
                    <Icon icon="solar:arrow-right-linear" height="18" class="ml-2" />
                    بازگشت به لیست
                </v-btn>
            </div>

            <!-- Loading State -->
            <Loading :loading="loading" />

            <!-- Invoice Details -->
            <div v-if="!loading && invoice" class="bg-bglight pa-6 rounded-md">
                <!-- Basic Info -->
                <v-row class="mb-4">
                    <v-col cols="12" md="6">
                        <div class="mb-4">
                            <v-label class="font-weight-semibold pb-2">کد فاکتور</v-label>
                            <div class="text-16">{{ invoice.code }}</div>
                        </div>
                    </v-col>
                    <v-col cols="12" md="6">
                        <div class="mb-4">
                            <v-label class="font-weight-semibold pb-2">تاریخ</v-label>
                            <div class="text-16">{{ invoice.dateTimeJalali }}</div>
                        </div>
                    </v-col>
                    <v-col cols="12" md="6">
                        <div class="mb-4">
                            <v-label class="font-weight-semibold pb-2">مشتری</v-label>
                            <div class="text-16">{{ invoice._customer?.fullName || "-" }}</div>
                        </div>
                    </v-col>
                    <v-col cols="12" md="6">
                        <div class="mb-4">
                            <v-label class="font-weight-semibold pb-2">وضعیت</v-label>
                            <div>
                                <v-chip class="spike-chip" rounded="pill" :color="getStatusColor(invoice.status)"
                                    variant="tonal" size="small">
                                    {{ getStatusLabel(invoice.status) }}
                                </v-chip>
                            </div>
                        </div>
                    </v-col>
                </v-row>

                <!-- Line Items -->
                <v-divider class="my-6" />
                <h6 class="text-18 font-weight-semibold mb-4">اقلام فاکتور</h6>
                <v-table class="spike-table month-table" v-if="invoice.items && invoice.items.length">
                    <thead>
                        <tr>
                            <th class="text-subtitle-1 font-weight-semibold">ردیف</th>
                            <th class="text-subtitle-1 font-weight-semibold">محصول</th>
                            <th class="text-subtitle-1 font-weight-semibold">تعداد</th>
                            <th class="text-subtitle-1 font-weight-semibold">قیمت واحد</th>
                            <th class="text-subtitle-1 font-weight-semibold">جمع</th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr v-for="(item, index) in invoice.items" :key="index" class="month-item">
                            <td class="text-14">{{ index + 1 }}</td>
                            <td class="text-14">{{ item._product?.name || "-" }}</td>
                            <td class="text-14">{{ item.quantity }}</td>
                            <td class="text-14">{{ formatters.price(item.price) }}</td>
                            <td class="text-14">{{ formatters.price(item.quantity * item.price) }}</td>
                        </tr>
                    </tbody>
                </v-table>

                <!-- Totals -->
                <v-divider class="my-6" />
                <v-row>
                    <v-col cols="12" md="6" offset-md="6">
                        <div class="d-flex justify-space-between mb-2">
                            <span class="font-weight-semibold">جمع کل:</span>
                            <span class="text-16 font-weight-bold">{{ formatters.price(invoice.total) }} تومان</span>
                        </div>
                    </v-col>
                </v-row>

                <!-- Description -->
                <v-divider class="my-6" v-if="invoice.description" />
                <div v-if="invoice.description">
                    <v-label class="font-weight-semibold pb-2">توضیحات</v-label>
                    <div class="text-14">{{ invoice.description }}</div>
                </div>
            </div>

            <!-- Action Buttons -->
            <div v-if="!loading && invoice" class="d-flex align-center justify-end ga-3 mt-6">
                <v-btn color="primary" variant="flat" rounded="pill" :to="`/sales-invoices/edit/${invoice._id}`">
                    <Icon icon="solar:pen-linear" height="18" class="ml-2" />
                    ویرایش
                </v-btn>
                <v-btn color="error" variant="flat" rounded="pill" @click="handleDelete">
                    <Icon icon="solar:trash-bin-minimalistic-linear" height="18" class="ml-2" />
                    حذف
                </v-btn>
            </div>
        </v-card-item>
    </v-card>

    <!-- Delete Confirmation Dialog -->
    <v-dialog v-model="showConfirmation" max-width="500px">
        <v-card>
            <v-card-title class="pa-4 bg-primary">حذف فاکتور فروش</v-card-title>
            <v-card-text class="pt-4">
                <h5 class="text-16">آیا از حذف این فاکتور اطمینان دارید؟</h5>
            </v-card-text>
            <v-card-actions>
                <v-btn color="primary" class="px-4" variant="flat" rounded="pill" @click="confirmDelete">
                    بله، حذف شود
                </v-btn>
                <v-btn color="error" variant="flat" rounded="pill" class="px-4 text-white"
                    @click="showConfirmation = false">
                    انصراف
                </v-btn>
            </v-card-actions>
        </v-card>
    </v-dialog>
</template>

<script setup>
import { ref, onMounted } from "vue";
import { useRouter, useRoute } from "vue-router";
import { useNuxtApp } from "#app";
import { Icon } from "@iconify/vue";
import Loading from "~/components/Loading.vue";
import { formatters } from "~/utils/formatters";

definePageMeta({
    layout: "admin-spike",
    middleware: "auth",
    requiresAuth: true,
    requiresRole: "admin",
});

const router = useRouter();
const route = useRoute();
const { $notify } = useNuxtApp();

const invoice = ref(null);
const loading = ref(true);
const showConfirmation = ref(false);

const getStatusColor = (status) => {
    const colors = {
        Draft: "grey",
        Pending: "warning",
        Completed: "success",
        Paid: "info",
        Cancelled: "error",
    };
    return colors[status] || "grey";
};

const getStatusLabel = (status) => {
    const labels = {
        Draft: "پیش نویس",
        Pending: "در انتظار",
        Completed: "تکمیل شده",
        Paid: "پرداخت شده",
        Cancelled: "لغو شده",
    };
    return labels[status] || status;
};

const loadInvoice = async () => {
    loading.value = true;
    try {
        const data = await useApiService.get(`sales-invoices/${route.params.id}`);
        if (data) {
            invoice.value = data;
            $notify("اطلاعات فاکتور بارگذاری شد", "success");
        }
    } catch (error) {
        $notify("مشکلی در بارگذاری اطلاعات پیش آمد", "error");
        router.push("/sales-invoices");
    } finally {
        loading.value = false;
    }
};

const handleDelete = () => {
    showConfirmation.value = true;
};

const confirmDelete = async () => {
    try {
        await useApiService.remove(`sales-invoices/${route.params.id}`);
        $notify("فاکتور با موفقیت حذف شد", "success");
        router.push("/sales-invoices");
    } catch (error) {
        $notify("مشکلی در حذف فاکتور پیش آمد", "error");
    } finally {
        showConfirmation.value = false;
    }
};

onMounted(() => {
    loadInvoice();
});
</script>
