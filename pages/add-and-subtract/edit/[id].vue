<template>
  <v-card class="spike-card" elevation="10">
    <v-card-item>
      <div class="d-flex justify-space-between align-center mb-6">
        <h5 class="text-20">ویرایش تراکنش</h5>
        <v-btn color="error" variant="flat" rounded="pill" to="/add-and-subtract">
          <Icon icon="solar:arrow-right-linear" height="18" class="ml-2" />
          بازگشت به لیست
        </v-btn>
      </div>

      <Loading :loading="loading" />

      <div v-if="!loading" class="bg-bglight pa-6 rounded-md">
        <AddAndSubtractAdd ref="formRef" @exit="handleExit" @refresh="handleRefresh" />
      </div>
    </v-card-item>
  </v-card>
</template>

<script setup>
import { ref, onMounted, watch } from "vue";
import { useRouter, useRoute } from "vue-router";
import { useNuxtApp } from "#app";
import { Icon } from "@iconify/vue";
import AddAndSubtractAdd from "~/components/add-and-subtract/AddAndSubtract-Add.vue";
import Loading from "~/components/Loading.vue";

definePageMeta({
  layout: "admin-spike",
  middleware: "auth",
  requiresAuth: true,
  requiresRole: "admin",
});

const router = useRouter();
const route = useRoute();
const { $notify } = useNuxtApp();
const formRef = ref(null);
const loading = ref(true);
const operationData = ref(null);

const loadData = async () => {
  loading.value = true;
  try {
    const response = await useApiService.get(`add-and-subtract/${route.params.id}`);
    console.log("API Response:", response);
    
    // Handle different response structures
    const data = response?.data || response;
    
    if (data) {
      console.log("Operation data to set:", data);
      operationData.value = data;
    }
  } catch (error) {
    console.error("Error loading data:", error);
    router.push("/add-and-subtract");
  } finally {
    loading.value = false;
  }
};

// Watch for both formRef and operationData to be ready
watch([formRef, operationData], ([form, data]) => {
  if (form && data && typeof form.setEdit === 'function') {
    console.log("Setting edit data:", data);
    console.log("Form ref methods:", Object.keys(form));
    try {
      form.setEdit(data);
      console.log("setEdit called successfully");
    } catch (error) {
      console.error("Error calling setEdit:", error);
    }
  } else {
    console.log("Waiting for form or data:", { hasForm: !!form, hasData: !!data, hasSetEdit: form && typeof form.setEdit === 'function' });
  }
}, { immediate: true });

const handleExit = () => {
  router.push("/add-and-subtract");
};

const handleRefresh = () => {
  // Optionally refresh data or perform other actions
};

onMounted(() => {
  loadData();
});
</script>
