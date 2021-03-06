<template>
  <v-dialog
    :model-value="value"
    :fullscreen="smAndDown"
  >
    <v-card
      :width="smAndDown?'auto':500"
    >
      <!-- close button -->
      <v-btn
        icon
        elevation="0"
        size="small"
        class="ma-1"
        style="position: absolute; right: 0; top: 0"
        @click="emit('update:modelValue', false)"
      >
        <v-icon>mdi-close</v-icon>
      </v-btn>

      <!-- title -->
      <v-card-title
        class="pb-0 pr-1"
      >
        <template
          v-if="!isEditing"
        >
          创建新纪录
        </template>
        <template v-else>
          编辑记录
        </template>
      </v-card-title>

      <!-- alert -->
      <v-card-text :class="{'mt-3': smAndDown}">
        <v-col
          cols="12"
          class="py-0"
        >
          <v-alert
            v-model="error.model"
            type="error"
            variant="contained-text"
            closable
          >
            {{ error.indication }}
          </v-alert>
        </v-col>
        <v-col
          cols="12"
          class="py-0"
          :class="{'mt-4': error.model}"
        >
          <v-alert
            v-model="warning.model"
            type="warning"
            variant="contained-text"
            closable
          >
            {{ warning.indication }}
          </v-alert>
        </v-col>

        <!-- theme -->
        <v-col
          cols="12"
          class="py-0"
          :class="{'mt-6': error.model || warning.model}"
        >
          <v-text-field
            v-if="isEditing"
            v-model="form.theme"
            variant="outlined"
            density="comfortable"
            label="主题*"
            readonly
          />
          <v-select
            v-else
            v-model="form.theme"
            variant="outlined"
            density="comfortable"
            label="主题*"
            :items="themes"
            no-data-text="请先创建主题"
          />
        </v-col>

        <!-- date -->
        <v-col
          cols="12"
          class="py-0"
        >
          <v-text-field
            v-model="form.date"
            variant="outlined"
            density="comfortable"
            label="日期*"
            readonly
          />
        </v-col>

        <!-- value -->
        <v-col
          cols="12"
          class="py-0 d-flex align-center"
          style="margin-bottom: 38px"
        >
          <v-col
            cols="12"
            style="position: relative;"
            class="pa-0 d-flex bordered-col"
            :class="{'error-col': errorMessages.value.length>0}"
          >
            <label
              class="label"
              :class="{'label-rise-up': valueFocused || hasValue(form.value)}"
            >
              数值*
            </label>
            <div
              class="overflow-hidden flex-grow-1"
              style="padding: 0 16px"
            >
              <input
                v-model="form.value"
                step="0.01"
                class="customized-input"
                type="number"
                @focus="valueFocused = true"
                @blur="valueFocused = false; validate('value')"
                @input="resetValidation('value')"
              >
            </div>
            <div
              v-if="!!errorMessages.value"
              class="error-message"
            >
              {{ errorMessages.value }}
            </div>
          </v-col>
        </v-col>

        <!-- comment -->
        <v-col
          cols="12"
          class="py-0"
        >
          <v-textarea
            v-model="form.comment"
            variant="outlined"
            density="comfortable"
            label="备注"
          />
        </v-col>

        <!-- submit button -->
        <v-col
          cols="12"
          class="py-0 mb-4"
        >
          <v-btn
            color="primary"
            block
            :disabled="submitting"
            @click="submit"
          >
            <v-progress-circular
              v-if="submitting"
              indeterminate
              width="2"
              size="20"
            />
            <template v-else>
              提交
            </template>
          </v-btn>
        </v-col>
      </v-card-text>
    </v-card>
  </v-dialog>
</template>
<script setup>
import {
  computed,
  reactive, ref, watchEffect,
} from 'vue';
import { useDisplay } from 'vuetify';
import { hasValue } from '@/utils/common';
import notify from '@/utils/notification';
import dayjs from 'dayjs';
import { useStore } from 'vuex';
import AV from 'leancloud-storage';

// props & emit
const props = defineProps({
  value: {
    type: Boolean,
    default: () => false,
  },
  themes: {
    type: Array,
    default: () => [],
  },
  editing: {
    type: Object,
    default: () => ({}),
  },
});
const emit = defineEmits(['update:modelValue', 'reload']);

// static
const fieldLabelMap = {
  theme: '主题',
  value: '数值',
  date: '日期',
};

// reactive
const { smAndDown } = useDisplay();
const store = useStore();
const valueFocused = ref(false);
const submitting = ref(false);

const form = reactive({
  theme: '',
  date: '',
  value: '',
  comment: '',
});
const errorMessages = reactive({
  theme: '',
  value: '',
});
const error = reactive({
  model: false,
  indication: '',
});
const warning = reactive({
  model: false,
  indication: '',
});

const isEditing = computed(() => JSON.stringify(props.editing) !== JSON.stringify({}));

// funcs
function resetValidation(field = 'all') {
  let fields;
  if (field === 'all') {
    fields = Object.keys(form);
  } else {
    fields = [field];
  }

  fields.forEach((f) => { errorMessages[f] = ''; });
}

function validate(field = 'all') {
  let fields;
  let valid = true;
  if (field === 'all') {
    fields = ['theme', 'value'];
  } else {
    fields = [field];
  }

  fields.forEach((f) => {
    if (!hasValue(form[f])) {
      errorMessages[f] = `${fieldLabelMap[f]} 不可为空`;
      valid = false;
    }
  });

  return valid;
}

async function getTheme() {
  const query = new AV.Query('Theme');
  return query.equalTo('name', form.theme).find();
}

async function createRecord() {
  const Record = AV.Object.extend('Record');
  const record = new Record();
  const theme = await getTheme();
  record.set('theme', theme);
  record.set('comment', form.comment);
  record.set('value', form.value);
  record.set('date', new Date(form.date));
  await record.save();
}

async function updateRecord(id) {
  const record = AV.Object.createWithoutData('Record', id);
  record.set('value', form.value);
  record.set('comment', form.comment);
  await record.save();
}

async function getExist() {
  try {
    const queryTheme = new AV.Query('Record');
    const queryDate = new AV.Query('Record');
    const [theme] = await getTheme();
    queryTheme.equalTo('theme', theme);
    queryDate.greaterThanOrEqualTo(
      'date',
      new Date(dayjs(form.date).format('YYYY-MM-DD')),
    );
    const [result] = await AV.Query.and(queryTheme, queryDate).find();
    return result || {};
  } catch {
    // Record 尚未创建
    return {};
  }
}

async function submit() {
  const valid = validate();
  if (valid) {
    submitting.value = true;
    try {
      const { id } = await getExist();
      if (id) {
        await updateRecord(id);
      } else {
        await createRecord();
      }
      emit('update:modelValue', false);
      emit('reload', form.theme);
      notify.success(store, '已提交');
    } catch (err) {
      error.model = true;
      error.indication = err.message || 'Unknown Exception';
    }
    submitting.value = false;
  } else {
    warning.model = true;
    warning.indication = '请先确保输入有效';
  }
}

// life circle
watchEffect(() => {
  const mixed = {};
  Object.assign(mixed, {
    theme: props.themes[0],
    date: dayjs().format('YYYY-MM-DD'),
    value: '',
    comment: '',
  });
  Object.assign(mixed, props.editing);
  Object.assign(form, mixed);
});
</script>
<style scoped>
.customized-input{
  width: 100%;
  outline: none;
  height: 48px;
  color: #434343;
}

.customized-input::placeholder{
  color: rgba(0, 0, 0, .87);
}

.bordered-col{
  border: thin solid #ababab;
  border-radius: 4px;
  box-sizing: border-box;
}

.bordered-col:hover{
  border-color: #7a7a7a;
}

.bordered-col:focus-within{
  border-color: #666;
  border-width: 2px;
}

.bordered-col:focus-within .customized-input{
  height: 46px;
}

.bordered-col:focus-within .label{
  color: #434343;
}

.error-col,
.error-col:focus-within,
.error-col .label,
.error-col:focus-within .label,
.error-col:hover {
  border-color: #bf3946;
  color: #bf3946;
}

.label {
  pointer-events: none;
  position: absolute;
  top: 12px;
  display: flex;
  align-items: center;
  color: #a3a3a3;
  margin: 0 16px;
  transform-origin: 0 12px;
  transition-duration: .15s;
  transition-timing-function: cubic-bezier(.4, 0, .2, 1);
}

.label-rise-up {
  top: -12px;
  font-size: 12px;
  background-color: #fff;
  height: 24px;
  padding: 0 5px;
  margin: 0 11px;
}

.error-message{
  position: absolute;
  padding: 6px 16px 0 16px;
  top: 50px;
  font-size: 12px;
  color: #bf3946;
}

.error-col:focus-within .error-message{
  top: 49px;
  padding-left: 15px;
}
</style>
