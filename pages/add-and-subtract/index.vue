<template>
  <v-card class="spike-card" elevation="10">
    <v-card-item>
      <!-- Search and Actions -->
      <div class="d-sm-flex justify-space-between align-center my-4">
        <v-sheet width="255" class="mb-lg-0 mb-4">
          <v-text-field
            v-model="searchValue"
            label="جستجوی عملیات"
            variant="outlined"
            hide-details
            class="w-100"
            density="compact"
          >
            <template v-slot:prepend-inner>
              <Icon icon="solar:magnifer-linear" height="18" width="25" />
            </template>
          </v-text-field>
        </v-sheet>
        <v-btn color="primary" rounded="pill" flat to="/add-and-subtract/create">
          عملیات جدید
        </v-btn>
      </div>

      <Loading :loading="loading" />

      <!-- Simple List View -->
      <div v-if="!loading && filteredList.length" class="mt-4">
        <v-list class="pa-0">
          <v-list-item
            v-for="item in filteredList"
            :key="item._id"
            class="border-b pa-4"
          >
            <v-list-item-title class="text-h6">
              {{ item.title }}
            </v-list-item-title>

            <template v-slot:append>
              <div class="d-flex ga-2">
                <v-btn
                  :to="`/add-and-subtract/edit/${item._id}`"
                  icon
                  size="small"
                  variant="text"
                  color="success"
                >
                  <Icon icon="solar:pen-linear" height="20" />
                  <v-tooltip activator="parent" location="bottom">ویرایش</v-tooltip>
                </v-btn>

                <v-btn
                  @click="confirmDelete(item._id)"
                  icon
                  size="small"
                  variant="text"
                  color="error"
                >
                  <Icon icon="solar:trash-bin-minimalistic-linear" height="20" />
                  <v-tooltip activator="parent" location="bottom">حذف</v-tooltip>
                </v-btn>
              </div>
            </template>
          </v-list-item>
        </v-list>
      </div>

      <EmptyList v-if="!loading && !filteredList.length" />
    </v-card-item>
  </v-card>

  <!-- Delete Confirmation Dialog -->
  <v-dialog v-model="showConfirmation" max-width="400">
    <v-card>
      <v-card-title class="text-h6">تأیید حذف</v-card-title>
      <v-card-text>آیا از حذف این عملیات اطمینان دارید؟</v-card-text>
      <v-card-actions>
        <v-spacer />
        <v-btn color="grey" variant="text" @click="showConfirmation = false">
          انصراف
        </v-btn>
        <v-btn color="error" variant="flat" @click="deleteItem">حذف</v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue';
import { useRouter } from 'vue-router';
import { useNuxtApp } from '#app';
import { Icon } from '@iconify/vue';
import Loading from '~/components/Loading.vue';
import EmptyList from '~/components/EmptyList.vue';

definePageMeta({
  layout: 'admin-spike',
  middleware: 'auth',
  requiresAuth: true,
  requiresRole: 'admin'
});

const router = useRouter();
const { $notify } = useNuxtApp();

const loading = ref(true);
const list = ref([]);
const searchValue = ref('');
const showConfirmation = ref(false);
const itemIdToDelete = ref(null);

const filteredList = computed(() => {
  if (!searchValue.value) return list.value;
  
  const search = searchValue.value.toLowerCase();
  return list.value.filter(item => 
    item.title?.toLowerCase().includes(search)
  );
});

const getOperations = async () => {
  loading.value = true;
  try {
    const data = await useApiService.get('add-and-subtract?perPage=100');
    list.value = data.list || [];
  } catch (error) {
    $notify('مشکلی در بارگذاری داده‌ها پیش آمد', 'error');
  } finally {
    loading.value = false;
  }
};

const confirmDelete = (id) => {
  itemIdToDelete.value = id;
  showConfirmation.value = true;
};

const deleteItem = async () => {
  try {
    await useApiService.remove(`add-and-subtract/${itemIdToDelete.value}`);
    $notify('عملیات با موفقیت حذف شد', 'success');
    await getOperations();
  } catch (error) {
    $notify('مشکلی در حذف عملیات پیش آمد', 'error');
  } finally {
    itemIdToDelete.value = null;
    showConfirmation.value = false;
  }
};

onMounted(() => {
  getOperations();
});
</script>

<style scoped lang="scss">
@import '@/assets/scss/spike-theme.scss';

.border-b {
  border-bottom: 1px solid #e5eaef;
}

.border-b:last-child {
  border-bottom: none;
}
</style>
