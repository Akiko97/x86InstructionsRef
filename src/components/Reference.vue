<template>
  <div id="reference">
    <el-table
      v-loading="loading"
      :data="filterData"
      :border="false"
      stripe
      :height="viewportHeight"
      style="width: 100%"
    >
      <el-table-column type="expand" >
        <template #default="props">
          <div class="idetails">
            <h2>{{ props.row.mnemonic.toUpperCase() }} - {{ props.row.summary }}</h2>
            <div v-for="(item, index) in props.row.table" :key="index">
              <el-table
                v-if="item.type === 'table'"
                :data="item.value.items"
                :border="true"
              >
                <el-table-column
                  v-for="(tableItem, tableIndex) in item.value.title"
                  :key="tableIndex"
                  :label="tableItem.label"
                  :prop="tableItem.prop"
                />
              </el-table>
              <blockquote v-if="item.type === 'quote'">{{ item.value }}</blockquote>
            </div>
            <div class="instruction" v-if="'instruction-info' in props.row">
              <el-divider content-position="right">
                <el-button
                  type="primary"
                  link
                  @click="showInstructionDetails = !showInstructionDetails"
                >
                  Instruction Details
                </el-button>
              </el-divider>
              <div v-if="showInstructionDetails">
                <div class="instruction-details">
                  <div v-for="(value, name, index) in props.row['instruction-info']" :key="index">
                    <h3>{{ value.title }}</h3>
                    <div v-for="(dataItem, dataIndex) in value.data" :key="dataIndex">
                      <p v-if="dataItem.type === 'string'">{{ dataItem.value }}</p>
                      <pre v-else-if="dataItem.type === 'code'"><code></code>{{ dataItem.value }}</pre>
                      <blockquote v-else-if="dataItem.type === 'quote'">{{ dataItem.value }}</blockquote>
                      <figcaption v-else-if="dataItem.type === 'caption'">{{ dataItem.value }}</figcaption>
                      <ul v-else-if="dataItem.type === 'list'">
                        <li v-for="(listItem, listIndex) in dataItem.value" :key="listIndex">
                          {{ listItem }}
                        </li>
                      </ul>
                      <div v-else v-html="dataItem.value" />
                    </div>
                  </div>
              </div>
              </div>
            </div>
            <div class="intrinsics" v-if="'intrinsics' in props.row">
              <el-divider content-position="right">
                <el-button
                  type="primary"
                  link
                  @click="showIntrinsicsDetails = !showIntrinsicsDetails"
                >
                Intrinsics Details
                </el-button>
              </el-divider>
              <div v-if="showIntrinsicsDetails">
                <div class="intrinsic-details" v-for="(item, index) in props.row.intrinsics" :key="index">
                  <h3>{{ item.name }}:</h3>
                  <pre><code>{{ item.signature }}</code></pre>
                  <pre v-for="(info, i) in item.other_infos" :key="i"><code>{{ info }}</code></pre>
                  <p>Set: {{ item.set }}</p>
                  <p>Flags: <span v-for="(flag, i) in item.flags" :key="i">{{ flag }}</span></p>
                  <div v-if="'description' in item && item.description !== ''">
                    <h3>Description</h3>
                    <p>{{ item.description }}</p>
                  </div>
                  <div v-if="'operation' in item && item.operation !== ''">
                    <h3>Operation</h3>
                    <pre><code>{{ item.operation }}</code></pre>
                  </div>
                  <div v-if="'performance' in item && item.performance !== ''">
                    <h3>Latency and Throughput</h3>
                    <div v-html="item.performance"></div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </template>
      </el-table-column>
      <el-table-column label="Mnemonic" sortable width="150" prop="mnemonic" />
      <el-table-column
        label="Set"
        width="95"
        prop="set"
        :filters="[
          {text: 'MMX', value: 'MMX'},
          {text: 'SSE Family', value: 'SSE_ALL'},
          {text: 'AVX Family', value: 'AVX_ALL'},
          {text: 'AVX-512 Family', value: 'AVX_512'},
          {text: 'AMX Family', value: 'AMX'},
          {text: 'SVML', value: 'SVML'},
          {text: 'Other', value: 'Other'},
          {text: '-', value: '-'},
        ]"
        :filter-method="filterSet"
      />
      <el-table-column label="Flags" width="95" prop="flags" />
      <el-table-column label="Summary" prop="summary">
        <template #header>
          <div class="summary-header">
            <div>Summary</div>
            <el-input
              v-model="search"
              size="small"
              placeholder="Type to search"
              style="width: 200px;"
              clearable
            />
          </div>
        </template>
      </el-table-column>
    </el-table>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue'
const viewportHeight = ref(window.innerHeight - 200)
window.addEventListener('resize', () => {
  viewportHeight.value = window.innerHeight - 200
})
const loading = ref(true)
const refData = ref([])
onMounted(async () => {
  viewportHeight.value = window.innerHeight - 200
  import('../data/instructions.json').then((data) => {
    refData.value = data.default
    loading.value = false
  }).catch((error) => {
    refData.value = [{
      mnemonic: 'ERROR',
      summary: error,
      set: 'ERROR',
      flags: 'ERROR',
    }]
  });
})
const search = ref('')
const filterData = computed(() => {
  return refData.value.filter((data) => {
    const check_intrinsics = (intrinsics, text) => {
      for (let i = 0; i < intrinsics.length; i++) {
        if (intrinsics[i].name.toLowerCase().includes(text.toLowerCase())) {
          return true
        }
        if ('instruction' in intrinsics[i] && intrinsics[i].instruction.toLowerCase().includes(text.toLowerCase())) {
          return true
        }
      }
      return false
    }
    const check_table = (table, text) => {
      for (let i = 0; i < table.length; i++) {
        if (table[i].type === 'table') {
          const tableItems = table[i].value.items
          for (let j = 0; j < tableItems.length; j++) {
            if ('instruction' in tableItems[j]) {
              if (tableItems[j].instruction.split(' ')[0].toLowerCase().includes(text.toLowerCase())) {
                return true
              }
            }
            else if ('opcodeinstruction' in tableItems[j]) {
              const regex = /(?<=\/r\s)(.*?)(?=\s)/
              const match = tableItems[j].opcodeinstruction.match(regex)
              if (match && match[0].toLowerCase().includes(text.toLowerCase())) {
                return true
              }
            }
          }
        }
      }
      return false
    }
    return !search.value ||
      data.mnemonic.toLowerCase().includes(search.value.toLowerCase()) ||
      ('intrinsics' in data && check_intrinsics(data.intrinsics, search.value)) ||
      ('table' in data && check_table(data.table, search.value))
  })
})
const showInstructionDetails = ref(false)
const showIntrinsicsDetails = ref(false)
const filterSet = (value, row) => {
  return row.set.includes(value)
}
</script>

<style scoped>
#reference {
  border-radius: 8px; 
  border: 1px solid #494949;
  max-width: calc(100vw - 40px);
  max-height:  calc(100vh - 40px);
  background: #181818;
  color: #f7f7f7;
  padding: 10px;
}
.summary-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.idetails {
  padding-left: 20px;
}
.intrinsic-details {
  margin: 5px;
  border: 1px dashed rgb(165, 210, 255);
  border-radius: 8px;
  padding: 10px;
}
.instruction-details {
  margin: 5px;
  border: 1px dashed rgb(165, 210, 255);
  border-radius: 8px;
  padding: 10px;
}
pre code {
  width: 100%;
}
</style>