<!--
 * Copyright (c) 2013-Now http://jeesite.com All rights reserved.
 * No deletion without permission, or be held responsible to law.
 * @author ThinkGem
-->
<template>
  <BasicDrawer
    v-bind="$attrs"
    :showFooter="true"
    :okAuth="'sys:empUser:edit'"
    @register="registerDrawer"
    @ok="handleSubmit"
    width="75%"
  >
    <template #title>
      <Icon :icon="getTitle.icon" class="pr-1 m-1" />
      <span> {{ getTitle.value }} </span>
    </template>
    <BasicForm @register="registerForm">
      <template #dataScopeTrees>
        <div class="flex flex-row flex-wrap">
          <div class="mr-5 mb-5" v-for="item in dataScopes" :key="item.moduleCode">
            <BasicTree
              v-if="moduleCodes.includes(item.moduleCode) && ['0', '2'].includes(item.ctrlPermi)"
              class="bg-gray"
              style="min-width: 300px"
              :title="t(item['ctrlName' + localeStore.getLocale] || item.ctrlName)"
              :toolbar="true"
              :checkable="true"
              :api="ctrlDataTreeData"
              :params="{ url: item.ctrlDataUrl, ctrlPermi }"
              :immediate="immediate"
              :defaultExpandLevel="2"
              :ref="setTreeRefs(item.ctrlType)"
              @tree-data-change="handleTreeDataChange"
            />
          </div>
        </div>
      </template>
    </BasicForm>
  </BasicDrawer>
</template>
<script lang="ts">
  export default defineComponent({
    name: 'ViewsSysEmpUserAuthDataScope',
  });
</script>
<script lang="ts" setup>
  import { defineComponent, ref, nextTick } from 'vue';
  import { useI18n } from '/@/hooks/web/useI18n';
  import { useMessage } from '/@/hooks/web/useMessage';
  import { useLocaleStore } from '/@/store/modules/locale';
  import { router } from '/@/router';
  import { Icon } from '/@/components/Icon';
  import { BasicForm, FormSchema, useForm } from '/@/components/Form';
  import { BasicDrawer, useDrawerInner } from '/@/components/Drawer';
  import { ctrlDataTreeData } from '/@/api/sys/empUser';
  // import { officeTreeData } from '/@/api/sys/office';
  import { secAdminForm, secAdminSave } from '/@/api/sys/secAdmin';
  import { BasicTree, TreeActionType } from '/@/components/Tree';

  const emit = defineEmits(['success', 'register']);

  const { t } = useI18n('sys.empUser');
  const { showMessage } = useMessage();
  const localeStore = useLocaleStore();
  const record = ref<Recordable>({});
  const getTitle = {
    icon: router.currentRoute.value.meta.icon || 'ant-design:book-outlined',
    value: t('???????????????'),
  };
  const ctrlPermi = ref<String>('');
  const moduleCodes = ref<Array<String>>([]);
  const dataScopes = ref<Array<Recordable>>([]);
  const userDataScopeList = ref<Array<Recordable>>([]);
  const immediate = ref(false);

  const inputFormSchemas: FormSchema[] = [
    {
      label: t('????????????'),
      field: 'userCode',
      fieldLabel: 'userName',
      // component: 'TreeSelect',
      // componentProps: {
      //   api: officeTreeData,
      //   params: { isLoadUser: true, userIdPrefix: '' },
      //   canSelectParent: false,
      //   allowClear: true,
      //   onChange: (userCode) => {
      //     loadData({ userCode });
      //   },
      // },
      component: 'ListSelect',
      componentProps: {
        selectType: 'empUserSelect',
        onChange: (userCode) => {
          loadData({ userCode });
        },
      },
    },
    {
      label: t('????????????'),
      field: 'userName',
      component: 'Input',
      componentProps: {
        disabled: true,
      },
    },
    {
      label: t('????????????????????????'),
      field: 'dataScopeInfo',
      component: 'FormGroup',
      colProps: { lg: 24, md: 24 },
    },
    {
      field: 'userDataScopeListJson',
      component: 'Input',
      colProps: { lg: 24, md: 24 },
      slot: 'dataScopeTrees',
    },
  ];

  const treeRefs: Recordable<TreeActionType> = {};
  const setTreeRefs = (key: string) => (el: any) => {
    if (el) treeRefs[key] = el;
  };

  const [registerForm, { resetFields, setFieldsValue, validate }] = useForm({
    schemas: inputFormSchemas,
    baseColProps: { lg: 12, md: 24 },
    labelWidth: 120,
  });

  const [registerDrawer, { setDrawerProps, closeDrawer }] = useDrawerInner(async (data) => {
    await resetFields();
    loadData(data);
  });

  async function loadData(data) {
    setDrawerProps({ loading: true });
    const res = await secAdminForm(data);
    record.value = (res.user || {}) as Recordable;
    ctrlPermi.value = res.ctrlPermi || '2';
    moduleCodes.value = res.moduleCodes || [];
    dataScopes.value = res.dataScopes || [];
    userDataScopeList.value = res.userDataScopeList || [];
    setFieldsValue(record.value);
    await loadTreeDatas();
    setDrawerProps({ loading: false });
  }

  let loadTreeDataNum: number;
  async function loadTreeDatas() {
    loadTreeDataNum = 0;
    nextTick(() => {
      if (immediate.value) {
        for (const key of Object.keys(treeRefs)) {
          treeRefs[key].setCheckedKeys([]);
          treeRefs[key].reload();
        }
      } else {
        immediate.value = true;
      }
    });
  }

  function handleTreeDataChange() {
    const keys = Object.keys(treeRefs);
    if (++loadTreeDataNum == keys.length) {
      let checkedKeys = {};
      userDataScopeList.value.forEach((item) => {
        if (!checkedKeys[item.ctrlType]) {
          checkedKeys[item.ctrlType] = [];
        }
        checkedKeys[item.ctrlType].push(item.ctrlData);
      });
      for (const key of keys) {
        treeRefs[key].setCheckedKeys(checkedKeys[key] || []);
      }
    }
  }

  function getUserDataScopeListJson() {
    const keys = Object.keys(treeRefs);
    let dataScopeData: Array<any> = [];
    for (const key of keys) {
      const ks = treeRefs[key].getCheckedKeys();
      for (const k of ks as Array<any>) {
        dataScopeData.push({
          ctrlType: key,
          ctrlData: k,
        });
      }
    }
    return JSON.stringify(dataScopeData);
  }

  async function handleSubmit() {
    try {
      if (!record.value.userCode) {
        showMessage(t('??????????????????????????????????????????????????????'));
        return;
      }
      const data = await validate();
      setDrawerProps({ confirmLoading: true });
      const params: any = {
        ...data,
        isNewRecord: record.value.isNewRecord,
        userCode: record.value.userCode,
        userDataScopeListJson: getUserDataScopeListJson(),
      };
      // console.log('submit', params, data, record);
      const res = await secAdminSave(params);
      showMessage(res.message);
      setTimeout(closeDrawer);
      emit('success', data);
    } catch (error: any) {
      if (error && error.errorFields) {
        showMessage(t('???????????????????????????????????????????????????'));
      }
      console.log('error', error);
    } finally {
      setDrawerProps({ confirmLoading: false });
    }
  }
</script>
