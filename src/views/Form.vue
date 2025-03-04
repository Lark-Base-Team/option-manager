<!--
 * @Version    : v1.00
 * @Author     : itchaox
 * @Date       : 2023-09-26 15:10
 * @LastAuthor : Wang Chao
 * @LastTime   : 2025-03-04 11:26
 * @desc       : 主要页面
-->
<script setup>
  import { bitable } from '@lark-base-open/js-sdk';
  import { Plus, Delete, Rank, Check, Star, User, Refresh, CloseBold, Remove } from '@element-plus/icons-vue';
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

    // 监听视图变更事件
    const off = base.onSelectionChange((event) => {
      if (event.data.tableId === databaseId.value) {
        // 视图变更时更新视图列表
        table.getViewMetaList().then((views) => {
          viewList.value = views;
        });
      }
    });

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
  const selectedOptions = ref([]); // 选中的选项
  const isAllSelected = computed(() => {
    return fieldOptions.value.length > 0 && selectedOptions.value.length === fieldOptions.value.length;
  });

  // 处理全选/取消全选
  const handleSelectAll = () => {
    if (isAllSelected.value) {
      selectedOptions.value = [];
    } else {
      selectedOptions.value = fieldOptions.value.map((option) => option.value);
    }
  };

  // 处理单个选项的选择
  const handleSelect = (optionValue) => {
    const index = selectedOptions.value.indexOf(optionValue);
    if (index === -1) {
      selectedOptions.value.push(optionValue);
    } else {
      selectedOptions.value.splice(index, 1);
    }
  };

  // 处理批量删除选中的选项
  const handleBatchDelete = async () => {
    if (selectedOptions.value.length === 0) {
      ElMessage.warning(t('message.selectOptionsFirst'));
      return;
    }

    try {
      await ElMessageBox.confirm(
        t('dialog.batchDelete.description', { count: selectedOptions.value.length }),
        t('dialog.batchDelete.title'),
        {
          confirmButtonText: t('button.confirm'),
          cancelButtonText: t('button.cancel'),
          type: 'warning',
        },
      );

      fieldOptions.value = fieldOptions.value.filter((option) => !selectedOptions.value.includes(option.value));
      selectedOptions.value = [];
      hasChanges.value = true;
      ElMessage.success(t('message.batchDeleteSuccess'));
    } catch (error) {
      // 用户取消删除操作
    }
  };

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

  // 防抖函数
  function debounce(fn, delay) {
    let timer = null;
    return function (...args) {
      if (timer) clearTimeout(timer);
      timer = setTimeout(() => {
        fn.apply(this, args);
      }, delay);
    };
  }

  // 检查选项名称是否重复
  function checkDuplicateName(currentOption, name) {
    return fieldOptions.value.some((option) => option.value !== currentOption.value && option.label === name);
  }

  // 处理选项名称变更
  async function handleOptionNameChange(option) {
    const optionIndex = fieldOptions.value.findIndex((o) => o.value === option.value);
    if (optionIndex !== -1) {
      // 检查是否重复
      const duplicateOption = fieldOptions.value.find((o) => o.value !== option.value && o.label === option.label);
      if (duplicateOption) {
        // 获取最近一次有效值
        const lastValidLabel =
          originalOptions.value.find((o) => o.value === option.value)?.label || fieldOptions.value[optionIndex].label;

        // 显示重名确认弹窗
        try {
          await ElMessageBox.confirm(
            t('dialog.rename.description', { name: duplicateOption.label }),
            t('dialog.rename.title'),
            {
              confirmButtonText: t('button.confirm'),
              showCancelButton: false,
              type: 'warning',
            },
          );
        } finally {
          // 无论用户点击确认还是取消，都恢复为最近一次有效值
          option.label = lastValidLabel;
          // 确保视图更新
          fieldOptions.value[optionIndex].label = lastValidLabel;
          // 重置hasChanges状态
          hasChanges.value = false;
        }
        return;
      }
      fieldOptions.value[optionIndex].label = option.label;
      hasChanges.value = true;
    }
  }

  // 防抖处理的选项名称变更
  const debouncedHandleOptionNameChange = debounce((option) => {
    fieldOptions.value.find((o) => o.value === option.value).label = option.label;
    hasChanges.value = true;
  }, 500);

  // 处理删除选项
  function handleDeleteOption(option) {
    const optionIndex = fieldOptions.value.findIndex((o) => o.value === option.value);
    if (optionIndex !== -1) {
      fieldOptions.value.splice(optionIndex, 1);
      hasChanges.value = true;
    }
  }

  // 处理添加选项
  function handleAddOption() {
    // 添加空选项名称校验
    const newOption = {
      label: '',
      value: `temp_${Date.now()}`, // 使用临时ID
    };

    // 检查是否已存在相同名称的选项
    const isDuplicate = fieldOptions.value.some((option) => option.label === newOption.label);
    if (isDuplicate) {
      ElMessage.warning('选项名称已存在，请输入其他名称');
      return;
    }

    fieldOptions.value.push(newOption);
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
    const newTableId = event.data.tableId;
    const newViewId = event.data.viewId;

    // 如果数据表发生变化，更新相关数据
    if (newTableId !== databaseId.value) {
      databaseId.value = newTableId;
      const table = await base.getTable(newTableId);
      // 获取新的视图列表
      viewList.value = await table.getViewMetaList();
      // 获取新的字段列表
      const _list = await table.getFieldMetaList();
      selectFieldList.value = _list.filter((item) => item.type === 3 || item.type === 4);
      // 自动选中第一个可用的选项字段
      if (selectFieldList.value && selectFieldList.value.length > 0) {
        selectFieldId.value = selectFieldList.value[0].id;
      }
    }

    viewId.value = newViewId;

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

  // 控制批量添加弹窗的显示
  const showBatchAddDialog = ref(false);
  const batchInputText = ref('');

  // 处理批量添加选项
  const handleBatchAddOptions = () => {
    showBatchAddDialog.value = true;
  };

  // 处理批量添加确认
  // 添加重置功能
  const handleReset = () => {
    ElMessageBox.confirm(t('dialog.reset.description'), t('dialog.reset.title'), {
      confirmButtonText: t('button.confirm'),
      cancelButtonText: t('button.cancel'),
      type: 'warning',
    })
      .then(() => {
        // 恢复到原始数据
        fieldOptions.value = JSON.parse(JSON.stringify(originalOptions.value));
        // 清除未保存状态
        hasChanges.value = false;
        ElMessage.success(t('message.resetSuccess'));
      })
      .catch(() => {
        // 用户取消操作，不做任何处理
      });
  };

  const handleEnterKeydown = async ($index) => {
    const option = fieldOptions.value[$index];
    if (!option) return;

    // 检查是否重复
    const duplicateOption = fieldOptions.value.find((o) => o.value !== option.value && o.label === option.label);
    if (duplicateOption) {
      // 获取最近一次有效值
      const lastValidLabel =
        originalOptions.value.find((o) => o.value === option.value)?.label || fieldOptions.value[$index].label;

      // 显示重名确认弹窗
      try {
        await ElMessageBox.confirm(
          t('dialog.rename.description', { name: duplicateOption.label }),
          t('dialog.rename.title'),
          {
            confirmButtonText: t('button.confirm'),
            showCancelButton: false,
            type: 'warning',
          },
        );
      } finally {
        // 无论用户点击确认还是取消，都恢复为最近一次有效值
        option.label = lastValidLabel;
        // 确保视图更新
        fieldOptions.value[$index].label = lastValidLabel;
        // 重置hasChanges状态
        hasChanges.value = false;
      }
      return;
    }

    // 如果是最后一行且输入框不为空，则添加新选项
    if ($index === fieldOptions.value.length - 1 && option.label.trim() !== '') {
      handleAddOption();
      return;
    }

    // 如果不是最后一行且没有重名，则切换到下一个输入框
    if ($index < fieldOptions.value.length - 1) {
      nextTick(() => {
        // 获取所有输入框
        const inputs = document.querySelectorAll('.el-table__body .el-input__inner');
        // 聚焦下一个输入框
        if (inputs[$index + 1]) {
          inputs[$index + 1].focus();
        }
      });
    }
  };

  const handleBatchAddConfirm = () => {
    if (!batchInputText.value.trim()) {
      ElMessage.warning(t('message.emptyInput'));
      return;
    }

    const newOptions = batchInputText.value
      .split('\n')
      .map((text) => text.trim())
      .filter((text) => text); // 过滤空行

    // 检查重复选项
    const duplicateNames = newOptions.filter((name) => fieldOptions.value.some((option) => option.label === name));

    if (duplicateNames.length > 0) {
      ElMessageBox.confirm(
        t('message.duplicateOptions', { names: duplicateNames.join('\n') }),
        t('dialog.batchAdd.title'),
        {
          confirmButtonText: t('button.confirm'),
          cancelButtonText: t('button.cancel'),
          type: 'warning',
        },
      )
        .then(() => {
          // 用户确认后，仅添加非重复的选项
          const uniqueOptions = newOptions.filter((name) => !duplicateNames.includes(name));
          uniqueOptions.forEach((name) => {
            fieldOptions.value.push({
              label: name,
              value: `temp_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
            });
          });

          hasChanges.value = true;
          showBatchAddDialog.value = false;
          batchInputText.value = '';

          // 滚动到底部
          nextTick(() => {
            const optionsContainer = document.querySelector('.options-container');
            if (optionsContainer) {
              optionsContainer.scrollTop = optionsContainer.scrollHeight;
            }
          });
        })
        .catch(() => {
          // 用户取消操作，不做任何处理
        });
      return;
    }

    // 添加新选项
    newOptions.forEach((name) => {
      fieldOptions.value.push({
        label: name,
        value: `temp_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
      });
    });

    hasChanges.value = true;
    showBatchAddDialog.value = false;
    batchInputText.value = '';

    // 滚动到底部
    nextTick(() => {
      const tableContainer = document.querySelector('.el-table__body-wrapper');
      if (tableContainer) {
        tableContainer.scrollTop = tableContainer.scrollHeight;
      }
    });

    ElMessage.success(t('message.batchAddSuccess', { count: newOptions.length }));
  };
</script>

<template>
  <div class="main">
    <div class="support-buttons">
      <el-button
        class="support-button"
        style="
          background-color: #ffd666;
          color: #000000;
          border: none;
          padding: 8px 16px;
          border-radius: 4px;
          font-size: 14px;
          font-weight: 700;
        "
        @click="handleSponsorClick"
      >
        <el-icon style="margin-right: 4px; color: #ff6b6b"
          ><svg
            viewBox="0 0 1024 1024"
            xmlns="http://www.w3.org/2000/svg"
            style="width: 14px; height: 14px"
          >
            <path
              d="M923 283.6c-13.4-31.1-32.6-58.9-56.9-82.8-24.3-23.8-52.5-42.4-84-55.5-32.5-13.5-66.9-20.3-102.4-20.3-49.3 0-97.4 13.5-139.2 39-10 6.1-19.5 12.8-28.5 20.1-9-7.3-18.5-14-28.5-20.1-41.8-25.5-89.9-39-139.2-39-35.5 0-69.9 6.8-102.4 20.3-31.4 13-59.7 31.7-84 55.5-24.4 23.9-43.5 51.7-56.9 82.8-13.9 32.3-21 66.6-21 101.9 0 33.3 6.8 68 20.3 103.3 11.3 29.5 27.5 60.1 48.2 91 32.8 48.9 77.9 99.9 133.9 151.6 92.8 85.7 184.7 144.9 188.6 147.3l23.7 15.2c10.5 6.7 24 6.7 34.5 0l23.7-15.2c3.9-2.5 95.7-61.6 188.6-147.3 56-51.7 101.1-102.7 133.9-151.6 20.7-30.9 37-61.5 48.2-91 13.5-35.3 20.3-70 20.3-103.3 0.1-35.3-7-69.6-20.9-101.9z"
              fill="currentColor"
            ></path></svg
        ></el-icon>
        {{ $t('label.sponsor') }}
      </el-button>

      <!-- 赞助弹窗 -->
      <el-dialog
        v-model="showSponsorDialog"
        :title="$t('dialog.sponsor.title')"
        width="95%"
        align-center
        style="text-align: center"
        :close-on-click-modal="false"
      >
        <div class="sponsor-content">
          <p style="margin-bottom: 12px">{{ $t('dialog.sponsor.description') }}</p>
          <p style="margin-bottom: 12px">{{ $t('dialog.sponsor.note0') }}</p>
          <p style="margin-bottom: 12px">{{ $t('dialog.sponsor.note1') }}</p>
        </div>
        <div>
          <div>
            <img
              style="width: 210px; height: 200px; margin-bottom: 20px"
              src="@/assets/images/wx.png"
              alt=""
            />
          </div>
          <div>
            <img
              style="width: 210px; height: 240px"
              src="@/assets/images/zfb.png"
              alt=""
            />
          </div>
        </div>
      </el-dialog>
      <!-- 批量添加弹窗 -->
      <el-dialog
        v-model="showBatchAddDialog"
        :title="$t('dialog.batchAdd.title')"
        width="500px"
        align-center
      >
        <el-input
          v-model="batchInputText"
          type="textarea"
          :rows="10"
          :placeholder="$t('placeholder.batchAddOptions')"
        />
        <template #footer>
          <span class="dialog-footer">
            <el-button @click="showBatchAddDialog = false">{{ $t('button.cancel') }}</el-button>
            <el-button
              type="primary"
              @click="handleBatchAddConfirm"
            >
              {{ $t('button.confirm') }}
            </el-button>
          </span>
        </template>
      </el-dialog>
      <el-button
        type="primary"
        class="support-button"
        :icon="User"
        style="
          background-color: #ff85c0;
          color: #ffffff;
          border: none;
          font-size: 14px;
          font-weight: 700;
          padding: 8px 16px;
          border-radius: 4px;
        "
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
          <div style="font-size: 14px; color: #909399; display: flex; align-items: center">
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
            type="selection"
            width="55"
            align="center"
            @selection-change="handleSelect"
          >
            <template #header>
              <el-checkbox
                v-model="isAllSelected"
                @change="handleSelectAll"
              />
            </template>
            <template #default="{ row }">
              <el-checkbox
                :value="selectedOptions.includes(row.value)"
                @change="() => handleSelect(row.value)"
              />
            </template>
          </el-table-column>
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
            <template #default="{ row, $index }">
              <el-input
                v-model="row.label"
                size="small"
                @input="debouncedHandleOptionNameChange(row)"
                @blur="handleOptionNameChange(row)"
                @keydown.enter="handleEnterKeydown($index)"
                :placeholder="$t('placeholder.optionName')"
              />
            </template>
          </el-table-column>
          <el-table-column
            width="80"
            align="center"
          >
            <template #header>
              <el-button
                type="text"
                size="small"
                @click="handleReset"
                style="
                  padding: 0;
                  width: 32px;
                  height: 32px;
                  display: flex;
                  align-items: center;
                  justify-content: center;
                  color: #999;
                  transition: color 0.2s;
                  font-size: 18px;
                "
                :class="{ 'delete-btn-hover': true }"
                class="delete-btn"
              >
                <el-icon><Refresh /></el-icon>
              </el-button>
            </template>
            <template #default="{ row, $index }">
              <el-button
                type="text"
                size="small"
                @click="handleDeleteOption(row, $index)"
                style="
                  padding: 0;
                  width: 32px;
                  height: 32px;
                  display: flex;
                  align-items: center;
                  justify-content: center;
                  color: #999;
                  transition: color 0.2s;
                  font-size: 16px;
                "
                :class="{ 'delete-btn-hover': true }"
                class="delete-btn"
              >
                <el-icon><CloseBold /></el-icon>
              </el-button>
            </template>
          </el-table-column>
        </el-table>
        <div class="options-footer">
          <div class="options-actions">
            <div class="button-groups">
              <div class="button-row">
                <el-button
                  type="primary"
                  @click="handleAddOption"
                  :icon="Plus"
                  style="
                    background-color: #ffffff;
                    border: 1px solid #0442d2;
                    color: #0442d2;
                    font-weight: 500;
                    padding: 8px 16px;
                    border-radius: 4px;
                    flex: 1;
                  "
                  >{{ $t('button.addOption') }}</el-button
                >
                <el-button
                  type="primary"
                  @click="handleBatchAddOptions"
                  :icon="Plus"
                  style="
                    background-color: #ffffff;
                    border: 1px solid #0442d2;
                    color: #0442d2;
                    font-weight: 500;
                    padding: 8px 16px;
                    border-radius: 4px;
                    flex: 1;
                  "
                  >{{ $t('button.batchAddOptions') }}</el-button
                >
              </div>
              <div class="button-row">
                <el-button
                  type="danger"
                  plain
                  @click="handleBatchDelete"
                  :icon="Delete"
                  style="background-color: #ffffff; font-weight: 500; padding: 8px 16px; border-radius: 4px; flex: 1"
                  >批量删除</el-button
                >
                <el-button
                  plain
                  type="danger"
                  @click="handleRemoveDuplicates"
                  :icon="Remove"
                  style="background-color: #ffffff; font-weight: 500; padding: 8px 16px; border-radius: 4px; flex: 1"
                  >去除重复</el-button
                >
              </div>
            </div>
            <el-button
              type="success"
              @click="handleSaveChanges"
              :disabled="!hasChanges"
              :loading="loading"
              :icon="Check"
              class="save-button"
              style="background-color: #0442d2; border: none; font-weight: 500; padding: 8px 16px; border-radius: 4px"
              >{{ $t('button.saveChanges') }}</el-button
            >
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
  .delete-btn-hover:hover {
    color: #409eff !important;
  }

  .delete-btn:hover {
    color: #409eff !important;
  }
  .main {
    font-weight: normal;
  }

  .support-buttons {
    display: flex;
    gap: 12px;
    margin-bottom: 8px;
  }

  .support-button {
    flex: 1;
    font-weight: 300;

    &:hover {
      /* 放大 1.1 倍 */
      transform: scale(1.1);
    }
  }

  .label {
    display: flex;
    align-items: center;
    margin-bottom: 8px;

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
    display: flex;
    flex-direction: column;
    gap: 12px;
    margin-top: 20px;
  }

  .options-actions {
    display: flex;
    flex-direction: column;
    gap: 12px;
  }

  .button-groups {
    display: flex;
    flex-direction: column;
    gap: 12px;
  }

  .button-row {
    display: flex;
    gap: 12px;
  }

  .add-buttons .el-button {
    flex: 1;
  }

  .save-button {
    width: 100%;
  }

  .drag-handle {
    cursor: move;
    color: #999;
  }

  .delete-btn-hover:hover {
    color: #ff4d4f !important;
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

  .el-table .el-table__cell {
    padding: 2px 0 !important;
  }
  .el-table :deep(.el-checkbox__input.is-checked .el-checkbox__inner) {
    background-color: #0442d2 !important;
    border-color: #0442d2 !important;
  }

  .el-table :deep(.el-checkbox__input.is-indeterminate .el-checkbox__inner) {
    background-color: #0442d2 !important;
    border-color: #0442d2 !important;
  }

  .el-table :deep(.el-checkbox__inner:hover) {
    border-color: #0442d2;
  }

  .el-table :deep(.el-checkbox__input.is-focus .el-checkbox__inner) {
    border-color: #0442d2;
  }

  .el-table :deep(.el-checkbox__inner) {
    &:hover {
      border-color: #0442d2;
    }
  }
</style>
