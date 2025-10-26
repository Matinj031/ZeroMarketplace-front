<template>
  <v-form class="mx-5" @submit.prevent="submit" ref="addAndSubtractForm" validate-on="submit lazy">
    <v-row class="mt-2">
      <!--      Title      -->
      <v-col cols="12" md="4">
        <v-label class="text-subtitle-2 font-weight-semibold pb-2 text-textPrimary">عنوان</v-label>
        <v-text-field v-model="form.title" placeholder="وارد کنید" :readonly="loading" :rules="[rules.required]"
          density="compact" variant="outlined" hide-details="auto" />
      </v-col>

      <!--      Default      -->
      <v-col cols="12" md="4">
        <v-label class="text-subtitle-2 font-weight-semibold pb-2 text-textPrimary">مقدار پیش فرض</v-label>
        <PercentOrPriceInput v-model="form.default" placeholder="وارد کنید" :readonly="loading"
          :rules="[rules.required]" />
      </v-col>

      <!--      Operation      -->
      <v-col cols="12" md="4">
        <v-label class="text-subtitle-2 font-weight-semibold pb-2 text-textPrimary">نوع عملیات</v-label>
        <v-select v-model="form.operation" placeholder="انتخاب کنید" :items="operations" :readonly="loading"
          :rules="[rules.required]" density="compact" variant="outlined" hide-details="auto" />
      </v-col>

      <!--      _Account      -->
      <v-col cols="12" md="4">
        <v-label class="text-subtitle-2 font-weight-semibold pb-2 text-textPrimary">حساب</v-label>
        <AccountInput v-model="form._account" />
      </v-col>

      <!--     Actions       -->
      <v-col class="mt-6" cols="12">
        <!--       Submit       -->
        <v-btn class="border rounded-lg" :loading="form.loading" prepend-icon="mdi-check-circle-outline" height="40"
          width="100" variant="text" type="submit" density="compact">
          ثبت
        </v-btn>

        <!--       Reset       -->
        <v-btn class="border mx-2 rounded-lg" color="pink" prepend-icon="mdi-delete-outline" height="40" width="100"
          variant="text" @click="reset" density="compact">
          بازنگری
        </v-btn>
      </v-col>
    </v-row>
  </v-form>
</template>

<script setup>
import { ref } from "vue";
import { useNuxtApp } from "#app";
import AccountInput from "~/components/accounts/AccountInput.vue";
import { rules } from "~/utils/validationRules";
import PercentOrPriceInput from "~/components/price/PercentOrPriceInput.vue";

// Define reactive state
const form = ref({
  title: "",
  default: "",
  operation: "add",
  _account: "",
});

const operations = [
  { title: "اضافه کردن", value: "add" },
  { title: "کم کردن", value: "subtract" },
];

const action = ref("add");
const loading = ref(false);
const addAndSubtractForm = ref(null);
const { $notify } = useNuxtApp();
const emit = defineEmits(["exit", "refresh"]);

// Reset form
const reset = () => {
  form.value = {
    title: "",
    default: "",
    operation: "add",
    _account: "",
  };
  loading.value = false;
  action.value = "add";
};

// Add new item
const add = async () => {
  try {
    const data = await useApiService.post("add-and-subtract", form.value);
    if (data) {
      $notify("عملیات با موفقیت انجام شد", "success");

      //  reset and exit
      reset();
      emit("exit");
      emit("refresh");
    }
  } catch (error) {
    $notify("مشکلی در عملیات پیش آمد؛ لطفا دوباره تلاش کنید", "error");
  }
};

// Edit existing item
const edit = async () => {
  try {
    const data = await useApiService.put(
      "add-and-subtract/" + form.value._id,
      form.value
    );
    if (data) {
      $notify("عملیات با موفقیت انجام شد", "success");

      //  reset and exit
      reset();
      emit("exit");
      emit("refresh");
    }
  } catch (error) {
    $notify("مشکلی در عملیات پیش آمد؛ لطفا دوباره تلاش کنید", "error");
  }
};

// Handle form submission
const submit = async () => {
  addAndSubtractForm.value?.validate();
  if (addAndSubtractForm.value?.isValid) {
    loading.value = true;
    if (action.value === "add") {
      await add();
    } else if (action.value === "edit") {
      await edit();
    }
    loading.value = false;
  }
};

// Set edit mode
const setEdit = (data) => {
  form.value = { ...data };
  action.value = "edit";
};

defineExpose({
  action,
  setEdit,
});
</script>

<style scoped></style>
