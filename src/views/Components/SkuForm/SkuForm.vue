<template>
  <div class="sku-container">
    <div v-if="!disabled" class="sku-check">
      <div v-if="theme == 1" class="theme-1">
        <el-card v-for="(item, index) in myAttribute" :key="index" class="item" shadow="never">
          <template #header>
            <div>{{ item.name }}</div>
          </template>
          <el-checkbox
            v-for="(item2, index2) in item.item"
            :key="index2"
            v-model="item2.checked"
            :label="item2.name"
            size="small"
          />
          <el-input
            v-if="item.canAddAttribute"
            v-model="item.addAttribute"
            size="small"
            placeholder="新增一个规格"
            class="add-attr"
            @keyup.enter="onAddAttribute(index)"
          >
            <template #append>
              <el-button size="small" icon="el-icon-plus" @click="onAddAttribute(index)"
                >添加</el-button
              >
            </template>
          </el-input>
        </el-card>
      </div>
      <el-table v-else :data="myAttribute" :show-header="false" class="theme-2">
        <el-table-column prop="name" width="120" :resizable="false" />
        <el-table-column>
          <template #default="scope">
            <el-checkbox
              v-for="(item2, index2) in scope.row.item"
              :key="index2"
              v-model="item2.checked"
              :label="item2.name"
              size="small"
            />
          </template>
        </el-table-column>
        <el-table-column width="250">
          <template #default="scope">
            <el-input
              v-model="scope.row.addAttribute"
              size="small"
              placeholder="新增一个规格"
              class="add-attr"
              @keyup.enter="onAddAttribute(scope.$index)"
            >
              <template #append>
                <el-button size="small" icon="el-icon-plus" @click="onAddAttribute(scope.$index)"
                  >添加</el-button
                >
              </template>
            </el-input>
          </template>
        </el-table-column>
      </el-table>
    </div>
    <div class="sku-list">
      <el-form ref="form" :model="form" status-icon inline-message>
        <el-table :data="form.skuData" stripe border highlight-current-row>
          <el-table-column
            v-if="emitAttribute.length > 0"
            type="index"
            width="50"
            align="center"
            :resizable="false"
          />
          <el-table-column
            v-for="(attr, index) in emitAttribute"
            :key="`attribute-${index}`"
            :label="attr.name"
            :prop="attr.name"
            width="120"
            align="center"
            :resizable="false"
            sortable
          />
          <el-table-column
            v-for="(item, index) in structure"
            :key="`structure-${index}`"
            :label="item.label"
            :prop="item.name"
            align="center"
            :resizable="false"
            min-width="120px"
          >
            <template #header>
              <span :class="{ required_title: item.required }">
                {{ item.label }}
              </span>
              <el-tooltip v-if="item.tip" effect="dark" :content="item.tip" placement="top">
                <i class="el-icon-info"></i>
              </el-tooltip>
            </template>
            <template #default="scope">
              <el-form-item
                v-if="item.type == 'input'"
                :key="`structure-input-${index}-${scope.row.sku}`"
                :prop="'skuData.' + scope.$index + '.' + item.name"
                :rules="rules[item.name]"
              >
                <el-input
                  v-model="scope.row[item.name]"
                  :placeholder="`请输入${item.label}`"
                  size="small"
                />
              </el-form-item>
              <el-form-item
                v-else-if="item.type == 'slot'"
                :key="`structure-input-${index}-${scope.row.sku}`"
                :prop="'skuData.' + scope.$index + '.' + item.name"
                :rules="rules[item.name]"
              >
                <slot
                  :name="item.name"
                  :$index="scope.$index"
                  :row="scope.row"
                  :column="scope.column"
                ></slot>
              </el-form-item>
            </template>
          </el-table-column>
          <template v-if="isBatch && form.skuData.length > 2" #append>
            <el-table :data="[{}]" :show-header="false">
              <el-table-column
                :width="attribute.length * 120 + 50"
                align="center"
                :resizable="false"
                >批量设置</el-table-column
              >
              <el-table-column
                v-for="(item, index) in structure"
                :key="`batch-structure-${index}`"
                align="center"
                :resizable="false"
                min-width="120px"
              >
                <el-input
                  v-if="item.type == 'input' && item.batch != false"
                  v-model="batch[item.name]"
                  :placeholder="`填写一个${item.label}`"
                  size="small"
                  @keyup.enter="onBatchSet(item.name)"
                />
              </el-table-column>
            </el-table>
          </template>
        </el-table>
      </el-form>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, computed, watch, onMounted } from 'vue'
import { ElMessage } from 'element-plus'

// 定义 props
const props = defineProps({
  sourceAttribute: {
    type: Array,
    default: () => []
  },
  attribute: {
    type: Array,
    default: () => []
  },
  sku: {
    type: Array,
    default: () => []
  },
  structure: {
    type: Array,
    default: () => [
      { name: 'price', type: 'input', label: '价格' },
      { name: 'stock', type: 'input', label: '库存' }
    ]
  },
  separator: {
    type: String,
    default: ';'
  },
  emptySku: {
    type: String,
    default: ''
  },
  disabled: {
    type: Boolean,
    default: false
  },
  theme: {
    type: Number,
    default: 1
  },
  async: {
    type: Boolean,
    default: false
  }
})

console.log('props', props);
// 定义 emit
const emit = defineEmits(['update:attribute', 'update:sku'])

// 响应式数据
const isInit = ref(false)
const myAttribute = ref([])
const form = reactive({
  skuData: []
})
const batch = reactive({})

// 计算属性
const rules = computed(() => {
  const rules = {}
  props.structure.forEach((v) => {
    if (v.type === 'input') {
      rules[v.name] = []
      if (v.required) {
        rules[v.name].push({ required: true, message: `${v.label}不能为空`, trigger: 'blur' })
      }
      if (v.validate) {
        rules[v.name].push({ validator: customizeValidate, trigger: 'blur' })
      }
    } else if (v.type === 'slot') {
      rules[v.name] = []
      if (v.required) {
        rules[v.name].push({
          required: true,
          message: `${v.label}不能为空`,
          trigger: ['change', 'blur']
        })
      }
      if (v.validate) {
        rules[v.name].push({ validator: customizeValidate, trigger: ['change', 'blur'] })
      }
    }
  })
  return rules
})

const isBatch = computed(() => {
  return props.structure.some((item) => item.type === 'input' && item.batch !== false)
})

const emitAttribute = computed(() => {
  const attribute = []
  myAttribute.value.forEach((v1) => {
    const obj = {
      name: v1.name,
      item: []
    }
    v1.item.forEach((v2) => {
      if (v2.checked) {
        obj.item.push(v2.name)
      }
    })
    if (obj.item.length !== 0) {
      attribute.push(obj)
    }
  })
  return attribute
})

// 监听器
watch(
  myAttribute,
  () => {
    if (!isInit.value) {
      emit('update:attribute', emitAttribute.value)
    }
    setTimeout(() => {
      if (props.attribute.length !== 0) {
        combinationAttribute()
      } else {
        form.skuData = []
        const obj = {
          sku: props.emptySku
        }
        props.structure.forEach((v) => {
          if (!(v.type === 'slot' && v.skuProperty === false)) {
            obj[v.name] = typeof v.defaultValue !== 'undefined' ? v.defaultValue : ''
          }
        })
        form.skuData.push(obj)
      }
      clearValidate()
    }, 0)
  },
  { deep: true }
)

watch(
  () => form.skuData,
  (newValue, oldValue) => {
    if (!isInit.value || (newValue.length === 1 && newValue[0].sku === props.emptySku)) {
      if (oldValue.length || !props.sku.length) {
        const arr = []
        newValue.forEach((v1) => {
          const obj = {
            sku: v1.sku
          }
          props.structure.forEach((v2) => {
            if (!(v2.type === 'slot' && v2.skuProperty === false)) {
              obj[v2.name] =
                v1[v2.name] || (typeof v2.defaultValue !== 'undefined' ? v2.defaultValue : '')
            }
          })
          arr.push(obj)
        })
        emit('update:sku', arr)
      }
    }
  },
  { deep: true }
)

// 生命周期钩子
onMounted(() => {
  if (!props.async) {
    init()
  }
})

// 方法
const init = () => {
  isInit.value = true
  const myAttributeTemp = []
  props.sourceAttribute.forEach((v) => {
    const temp = {
      name: v.name,
      canAddAttribute: typeof v.canAddAttribute !== 'undefined' ? v.canAddAttribute : true,
      addAttribute: '',
      item: []
    }
    v.item.forEach((itemName) => {
      temp.item.push({
        name: itemName,
        checked: false
      })
    })
    myAttributeTemp.push(temp)
  })
  props.attribute.forEach((attrVal) => {
    myAttributeTemp.forEach((myAttrVal) => {
      if (attrVal.name === myAttrVal.name) {
        attrVal.item.forEach((attrName) => {
          if (
            !myAttrVal.item.some((myAttrItem) => {
              if (attrName === myAttrItem.name) {
                myAttrItem.checked = true
              }
              return attrName === myAttrItem.name
            })
          ) {
            myAttrVal.item.push({
              name: attrName,
              checked: true
            })
          }
        })
      }
    })
  })
  myAttribute.value = myAttributeTemp
  setTimeout(() => {
    props.sku.forEach((skuItem) => {
      form.skuData.forEach((skuDataItem) => {
        if (skuItem.sku === skuDataItem.sku) {
          props.structure.forEach((structureItem) => {
            skuDataItem[structureItem.name] = skuItem[structureItem.name]
          })
        }
      })
    })
    isInit.value = false
  }, 0)
}

const combinationAttribute = (index = 0, dataTemp = []) => {
  if (index === 0) {
    for (let i = 0; i < props.attribute[0].item.length; i++) {
      const obj = {
        sku: props.attribute[0].item[i],
        [props.attribute[0].name]: props.attribute[0].item[i]
      }
      props.structure.forEach((v) => {
        if (!(v.type === 'slot' && v.skuProperty === false)) {
          obj[v.name] = typeof v.defaultValue !== 'undefined' ? v.defaultValue : ''
        }
      })
      dataTemp.push(obj)
    }
  } else {
    const temp = []
    for (let i = 0; i < dataTemp.length; i++) {
      for (let j = 0; j < props.attribute[index].item.length; j++) {
        temp.push(JSON.parse(JSON.stringify(dataTemp[i])))
        temp[temp.length - 1][props.attribute[index].name] = props.attribute[index].item[j]
        temp[temp.length - 1]['sku'] = [
          temp[temp.length - 1]['sku'],
          props.attribute[index].item[j]
        ].join(props.separator)
      }
    }
    dataTemp = temp
  }
  if (index !== props.attribute.length - 1) {
    combinationAttribute(index + 1, dataTemp)
  } else {
    if (!isInit.value || props.async) {
      for (let i = 0; i < form.skuData.length; i++) {
        for (let j = 0; j < dataTemp.length; j++) {
          if (form.skuData[i].sku === dataTemp[j].sku) {
            dataTemp[j] = form.skuData[i]
          }
        }
      }
    }
    form.skuData = dataTemp
  }
}

const onAddAttribute = (index) => {
  myAttribute.value[index].addAttribute = myAttribute.value[index].addAttribute.trim()
  if (myAttribute.value[index].addAttribute !== '') {
    if (!myAttribute.value[index].addAttribute.includes(props.separator)) {
      const flag = myAttribute.value[index].item.some((item) => {
        return item.name === myAttribute.value[index].addAttribute
      })
      if (!flag) {
        myAttribute.value[index].item.push({
          name: myAttribute.value[index].addAttribute,
          checked: true
        })
        myAttribute.value[index].addAttribute = ''
      } else {
        ElMessage({
          type: 'warning',
          message: '请勿添加相同规格'
        })
      }
    } else {
      ElMessage({
        type: 'warning',
        message: `规格里不允许出现「 ${props.separator} 」字符，请检查后重新添加`
      })
    }
  }
}

const onBatchSet = (type) => {
  if (batch[type] !== '') {
    form.skuData.forEach((v) => {
      v[type] = batch[type]
    })
    batch[type] = ''
    validateFieldByColumns([type], () => {})
  }
}

const customizeValidate = (rule, value, callback) => {
  const [model, index, name] = rule.field.split('.')
  props.structure.forEach((v) => {
    if (v.name === name) {
      v.validate(form[model], index, callback)
    }
  })
}

const validate = (callback) => {
  formRef.value.validate((valid) => {
    callback(valid)
  })
}

const validateFieldByColumns = (columns, callback) => {
  const props = []
  form.skuData.forEach((v, i) => {
    columns.forEach((col) => {
      props.push(`skuData.${i}.${col}`)
    })
  })
  formRef.value.validateField(props, (valid) => {
    callback(valid)
  })
}

const validateFieldByRows = (index, prop, callback) => {
  formRef.value.validateField([`skuData.${index}.${prop}`], (valid) => {
    callback(valid)
  })
}

const clearValidate = () => {
  formRef.value.clearValidate()
}

// 暴露方法
defineExpose({
  validate,
  validateFieldByColumns,
  validateFieldByRows,
  clearValidate
})
</script>

<style lang="scss" scoped>
.sku-container {
  ::v-deep .el-card {
    margin: 10px 0;
    .el-card__header {
      line-height: initial;
      padding: 10px 20px;
    }
    .el-card__body {
      padding: 10px 20px 20px;
    }
  }
  .sku-check {
    .theme-1 {
      display: flex;
      flex-wrap: wrap;
      justify-content: space-between;
      margin-bottom: 10px;
      .item {
        width: 32%;
        &:last-child:nth-child(3n - 1) {
          margin-right: calc(100% - 32% * 2 - 4% / 2) !important;
        }
        .add-attr {
          width: 100%;
          margin-top: 10px;
        }
      }
    }
    .theme-2 {
      border: 1px solid #ebeef5;
      border-bottom: 0;
      margin-bottom: 20px;
    }
  }
  .sku-name {
    text-align: right;
  }
  .batch-set {
    width: 100%;
    margin-top: 5px;
  }
  .sku-list {
    line-height: initial;
    ::v-deep .el-input__inner {
      text-align: center;
    }
    ::v-deep .el-table__append-wrapper {
      overflow: initial;
      .el-table {
        overflow: initial;
        .el-table__body-wrapper {
          overflow: initial;
        }
      }
    }
    ::v-deep .el-form-item {
      margin-bottom: 0;
      .el-form-item__content {
        line-height: initial;
        .el-form-item__error {
          margin-left: 0;
        }
      }
    }
    .required_title::before {
      content: '*';
      color: #f56c6c;
    }
  }
}
</style>
