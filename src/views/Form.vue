<!--
 * @Version    : v1.00
 * @Author     : itchaox
 * @Date       : 2023-09-26 15:10
 * @LastAuthor : Wang Chao
 * @LastTime   : 2025-03-03 13:03
 * @desc       : 主要页面
-->
<script setup>
  import { bitable } from '@lark-base-open/js-sdk';
  import { Plus, Delete, Rank, Check, Star, User } from '@element-plus/icons-vue';
  import { ElMessage } from 'element-plus';

  // 国际化
  import { useI18n } from 'vue-i18n';
  const { t } = useI18n();

  // 选择模式 cell 单元格; field 字段; database 数据表
  const selectModel = ref('cell');

  const databaseId = ref();
  const viewList = ref();
  const viewId = ref();
  const fieldList = ref();
  const fieldId = ref();

  const base = bitable.base;

  // 当前点击字段id
  const currentFieldId = ref();
  const recordId = ref();

  const currentValue = ref();
  const loading = ref(false);

  onMounted(async () => {
    const selection = await base.getSelection();
    databaseId.value = selection.tableId;
    const table = await base.getTable(databaseId.value);
    viewList.value = await table.getViewMetaList();
    viewId.value = selection.viewId;

    // 获取字段列表并自动选中第一个单选/多选字段
    const view = await table.getViewById(viewId.value);
    const _list = await view.getFieldMetaList();
    selectFieldList.value = _list.filter((item) => item.type === 3 || item.type === 4);

    // 如果有可用的单选/多选字段，自动选中第一个
    if (selectFieldList.value && selectFieldList.value.length > 0) {
      selectFieldId.value = selectFieldList.value[0].id;
    }
  });

  const selectFieldList = ref();
  const selectFieldId = ref();
  const fieldOptions = ref([]);
  const selectedOption = ref();
  const originalOptions = ref([]); // 保存原始选项数据
  const hasChanges = ref(false); // 标记是否有未保存的修改

  // 监听选择的字段变化，获取字段选项值
  watch(selectFieldId, async (newValue) => {
    try {
      if (!newValue) {
        fieldOptions.value = [];
        originalOptions.value = [];
        selectedOption.value = null;
        hasChanges.value = false;
        return;
      }
      const table = await base.getTable(databaseId.value);
      const field = await table.getFieldById(newValue);
      const fieldMeta = await field.getMeta();

      // 根据字段类型获取选项值
      if (fieldMeta.type === 3 || fieldMeta.type === 4) {
        // 单选或多选字段
        const options = await field.getOptions();
        const mappedOptions = options.map((option) => ({
          label: option.name,
          value: option.id,
        }));
        fieldOptions.value = mappedOptions;
        originalOptions.value = JSON.parse(JSON.stringify(mappedOptions)); // 深拷贝保存原始数据
        hasChanges.value = false;
      }
    } catch (error) {
      console.error('获取字段选项值失败:', error);
      fieldOptions.value = [];
      originalOptions.value = [];
      selectedOption.value = null;
      hasChanges.value = false;
    }
  });

  // 处理选项名称变更
  async function handleOptionNameChange(option) {
    hasChanges.value = true;
  }

  // 处理删除选项
  function handleDeleteOption(option, index) {
    fieldOptions.value.splice(index, 1);
    hasChanges.value = true;
  }

  // 处理添加选项
  function handleAddOption() {
    fieldOptions.value.push({
      label: '',
      value: `temp_${Date.now()}`, // 使用临时ID
    });
    hasChanges.value = true;

    // 等待DOM更新后执行滚动和聚焦
    nextTick(() => {
      // 获取表格容器并滚动到底部
      const tableContainer = document.querySelector('.el-table__body-wrapper');
      if (tableContainer) {
        tableContainer.scrollTop = tableContainer.scrollHeight;
      }

      // 获取最后一个输入框并聚焦
      const inputs = document.querySelectorAll('.el-table__body .el-input__inner');
      const lastInput = inputs[inputs.length - 1];
      if (lastInput) {
        lastInput.focus();
      }
    });
  }

  // 保存所有修改
  async function handleSaveChanges() {
    try {
      loading.value = true;
      const table = await base.getTable(databaseId.value);
      const field = await table.getFieldById(selectFieldId.value);

      // 获取需要删除的选项
      const deletedOptions = originalOptions.value.filter(
        (original) => !fieldOptions.value.some((current) => current.value === original.value),
      );

      // 获取需要添加的选项
      const addedOptions = fieldOptions.value.filter((current) => current.value.startsWith('temp_'));

      // 获取需要更新的选项
      const updatedOptions = fieldOptions.value.filter(
        (current) =>
          !current.value.startsWith('temp_') &&
          originalOptions.value.some(
            (original) => original.value === current.value && original.label !== current.label,
          ),
      );

      // 执行删除操作
      for (const option of deletedOptions) {
        await field.deleteOption(option.value);
      }

      // 执行添加操作
      for (const option of addedOptions) {
        const newOption = await field.addOption(option.label);
        // 更新临时ID为实际ID
        const index = fieldOptions.value.findIndex((o) => o.value === option.value);
        if (index !== -1) {
          fieldOptions.value[index].value = newOption.id;
        }
      }

      // 执行更新操作
      for (const option of updatedOptions) {
        await field.setOption(option.value, { name: option.label });
      }

      // 更新原始数据
      originalOptions.value = JSON.parse(JSON.stringify(fieldOptions.value));
      hasChanges.value = false;
      ElMessage.success('保存成功');
    } catch (error) {
      console.error('保存修改失败:', error);
      ElMessage.error('保存失败');
    } finally {
      loading.value = false;
    }
  }

  // 根据视图列表获取字段列表
  watch(viewId, async (newValue, oldValue) => {
    const table = await base.getTable(databaseId.value);
    const view = await table.getViewById(newValue);
    const _list = await view.getFieldMetaList();

    // 只展示文本相关字段
    fieldList.value = _list.filter((item) => item.type === 1);
    // 获取单选/多选字段
    selectFieldList.value = _list.filter((item) => item.type === 3 || item.type === 4);
  });

  // 切换选择模式时,重置选择
  watch(selectModel, async (newValue, oldValue) => {
    if (newValue === 'cell') return;
    // 单列和数据表模式，默认选中当前数据表和当前视图

    const selection = await base.getSelection();
    databaseId.value = selection.tableId;

    if (newValue === 'field') {
      fieldId.value = '';
      fieldList.value = [];
      viewId.value = '';

      const table = await base.getTable(databaseId.value);
      viewList.value = await table.getViewMetaList();
      viewId.value = selection.viewId;
    }
  });

  base.onSelectionChange(async (event) => {
    // 获取点击的字段id和记录id
    currentFieldId.value = event.data.fieldId;
    recordId.value = event.data.recordId;

    // 获取当前数据表和视图
    databaseId.value = event.data.tableId;
    viewId.value = event.data.viewId;

    const table = await base.getActiveTable();
    if (currentFieldId.value && recordId.value) {
      // 修改当前数据
      let data = await table.getCellValue(currentFieldId.value, recordId.value);
      if (data && data[0].text !== currentValue.value) {
        currentValue.value = data[0].text;
      }
    }
  });
  const searchQuery = ref('');

  const filteredOptions = computed(() => {
    if (!searchQuery.value) return fieldOptions.value;
    return fieldOptions.value.filter((option) => option.label.toLowerCase().includes(searchQuery.value.toLowerCase()));
  });
  // 处理关注按钮点击
  const handleFollowClick = () => {
    window.open('https://space.bilibili.com/521041866', '_blank');
  };

  // 控制赞助弹窗的显示
  const showSponsorDialog = ref(false);

  // 处理赞助按钮点击
  const handleSponsorClick = () => {
    showSponsorDialog.value = true;
  };
</script>

<template>
  <div class="main">
    <div class="support-buttons">
      <el-button
        type="warning"
        class="support-button"
        :icon="Star"
        plain
        @click="handleSponsorClick"
        >{{ $t('label.sponsor') }}</el-button
      >

      <!-- 赞助弹窗 -->
      <el-dialog
        v-model="showSponsorDialog"
        :title="$t('dialog.sponsor.title')"
        width="300px"
        align-center
      >
        <div class="sponsor-content">
          <p style="margin-bottom: 8px">{{ $t('dialog.sponsor.description') }}</p>
          <p style="color: #909399; font-size: 12px; margin-bottom: 8px">{{ $t('dialog.sponsor.note') }}</p>
          <p style="margin-bottom: 8px">{{ $t('dialog.sponsor.note1') }}</p>
        </div>
        <div>
          <div>
            <img
              style="width: 200px; height: 200px; margin-bottom: 20px"
              src="@/assets/images/wx.png"
              alt=""
            />
          </div>
          <div>
            <img
              style="width: 200px; height: 240px"
              src="@/assets/images/zfb.png"
              alt=""
            />
          </div>
        </div>
      </el-dialog>
      <el-button
        type="primary"
        class="support-button"
        :icon="User"
        plain
        @click="handleFollowClick"
        >{{ $t('label.follow') }}</el-button
      >
    </div>
    <div class="label">
      <div class="text">{{ $t('label.view') }}</div>
      <el-select
        v-model="viewId"
        :placeholder="$t('placeholder.view')"
        popper-class="selectStyle"
        filterable
      >
        <el-option
          v-for="item in viewList"
          :key="item.id"
          :label="item.name"
          :value="item.id"
        />
      </el-select>
    </div>
    <div class="label">
      <div class="text">{{ $t('label.field') }}</div>
      <el-select
        v-model="selectFieldId"
        :placeholder="$t('placeholder.field')"
        popper-class="selectStyle"
        filterable
      >
        <el-option
          v-for="item in selectFieldList"
          :key="item.id"
          :label="item.name"
          :value="item.id"
        />
      </el-select>
    </div>

    <div
      class="label"
      v-if="selectFieldId"
    >
      <div class="options-container">
        <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 12px">
          <el-input
            v-model="searchQuery"
            :placeholder="$t('placeholder.search')"
            clearable
            style="width: calc(100% - 80px)"
          />
          <div style="font-size: 12px; color: #909399; display: flex; align-items: center">
            {{ $t('label.optionCount') }}：<span style="color: #0442d2">{{ filteredOptions.length }}</span>
          </div>
        </div>
        <el-table
          :data="filteredOptions"
          style="width: 100%"
          row-key="value"
          :max-height="400"
          v-loading="loading"
        >
          <el-table-column
            type="index"
            width="60"
            align="center"
            :label="$t('label.serialNumber')"
            class-name="no-wrap-column"
          />
          <el-table-column
            prop="label"
            :label="$t('label.optionName')"
          >
            <template #default="{ row }">
              <el-input
                v-model="row.label"
                size="small"
                @change="handleOptionNameChange(row)"
                :placeholder="$t('placeholder.optionName')"
              />
            </template>
          </el-table-column>
          <el-table-column
            width="80"
            align="center"
          >
            <template #default="{ row, $index }">
              <el-button
                type="danger"
                size="small"
                @click="handleDeleteOption(row, $index)"
                :icon="Delete"
                circle
              />
            </template>
          </el-table-column>
        </el-table>
        <div class="options-footer">
          <div class="options-actions">
            <el-button
              type="primary"
              @click="handleAddOption"
              :icon="Plus"
              style="margin-right: 12px"
              >{{ $t('button.addOption') }}</el-button
            >
            <el-button
              type="success"
              @click="handleSaveChanges"
              :disabled="!hasChanges"
              :loading="loading"
              :icon="Check"
              >{{ $t('button.saveChanges') }}</el-button
            >
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
  .main {
    font-weight: normal;
  }

  .support-buttons {
    display: flex;
    gap: 12px;
    margin-bottom: 20px;
  }

  .support-button {
    flex: 1;
    font-weight: 300;
  }

  .label {
    display: flex;
    align-items: center;
    margin-bottom: 20px;

    .text {
      width: 70px;
      margin-right: 10px;
      white-space: nowrap;
      font-size: 14px;
    }

    :deep(.el-radio-button__original-radio:checked + .el-radio-button__inner) {
      color: #fff;
      background-color: rgb(20, 86, 240);
      border-color: rgb(20, 86, 240);
      box-shadow: 1px 0 0 0 rgb(20, 86, 240);
    }

    :deep(.el-radio-button__inner) {
      font-weight: 300;
    }

    :deep(.el-radio-button__inner:hover) {
      color: rgb(20, 86, 240);
    }

    :deep(.el-input__inner) {
      font-weight: 300;
    }
  }

  .show-data {
    margin-top: 20px;
    width: 90%;
    height: 50vh;
    border: 1px solid #ccc;
    padding: 10px;
    border-radius: 10px;
  }

  .options-container {
    flex: 1;
    min-width: 0;
  }

  .options-footer {
    margin-top: 12px;
    display: flex;
    justify-content: flex-start;
  }

  .drag-handle {
    cursor: move;
    color: #999;
  }
</style>

<style>
  .selectStyle {
    .el-select-dropdown__item {
      font-weight: 300 !important;
    }

    .el-select-dropdown__item.selected {
      color: rgb(20, 86, 240);
    }
  }
</style>
