<template>
  <el-container>
    <el-header>
      <h2>实例管理面板</h2>
    </el-header>

    <el-main>
      <el-row :gutter="20">
        <!-- 搜索框 -->
        <el-col :span="18">
          <el-input
            v-model="searchQuery"
            style="width: 300px"
            size="medium"
            placeholder="请输入关键词搜索"
            clearable
          />
          <el-button
            style="margin-left: 10px"
            type="primary"
            icon="el-icon-search"
            @click="handleSearch"
          >
            搜索
          </el-button>
        </el-col>
        <!-- 同步按钮 -->
        <el-col :span="6" class="text-right">
          <el-button type="primary" @click="refreshData">
            <i class="el-icon-refresh" /> 同步数据
          </el-button>
        </el-col>
      </el-row>

      <!-- 操作选择框 -->
      <el-row :gutter="20" style="margin-top: 20px;">
        <el-col :span="18">
          <el-select
            v-model="selectedOperation"
            placeholder="选择操作"
            style="width: 300px;"
          >
            <el-option
              v-for="item in options"
              :key="item.value"
              :label="item.label"
              :value="item.value"
            />
          </el-select>

          <el-button
            style="margin-left: 10px"
            type="success"
            @click="confirmOperation"
          >
            执行操作
          </el-button>
        </el-col>
      </el-row>

      <!-- 表格卡片布局 -->
      <el-card style="margin-top: 20px;" shadow="hover">
        <el-table
          ref="multipleTable"
          :data="displayedTableData"
          tooltip-effect="dark"
          border
          style="width: 100%; min-width: 1300px;"
          @selection-change="handleSelectionChange"
        >
          <!-- 多行选择的复选框列 -->
          <el-table-column
            type="selection"
            width="55"
          />

          <!-- 区域作为第一列 -->
          <el-table-column
            label="区域"
            prop="region"
            width="150"
          />

          <!-- 实例名称列 -->
          <el-table-column
            label="实例名称"
            prop="iname"
            width="150"
          />

          <!-- IP地址列 -->
          <el-table-column
            label="IP地址"
            prop="ip"
            width="150"
          >
            <template slot-scope="scope">
              <el-tooltip content="查看 IP 详情" placement="top">
                <span>{{ scope.row.ip }}</span>
              </el-tooltip>
            </template>
          </el-table-column>

          <!-- 实例类型列 -->
          <el-table-column
            label="实例类型"
            prop="itype"
            width="150"
          />

          <!-- 创建时间列 (倒数第二列) -->
          <el-table-column
            label="创建时间"
            prop="ctime"
            width="150"
          />

          <!-- 状态列作为最后一列 -->
          <el-table-column
            label="状态"
            prop="state"
            width="150"
          >
            <template slot-scope="scope">
              <el-tag :type="statusColor(scope.row.state)">
                {{ scope.row.state }}
              </el-tag>
            </template>
          </el-table-column>
        </el-table>

        <!-- 分页组件 -->
        <el-pagination
          background
          layout="total, sizes, prev, pager, next"
          :current-page="currentPage"
          :page-sizes="[10, 20, 30, 40]"
          :page-size="pageSize"
          :total="totalItems"
          @current-change="handlePaginationChange"
          @size-change="handlePageSizeChange"
          style="margin-top: 20px;"
        >
        </el-pagination>
      </el-card>

      <!-- 全选和取消选择按钮 -->
      <div style="margin-top: 20px" class="text-right">
        <el-button type="info" @click="toggleSelection(filteredTableData)">全选</el-button>
        <el-button type="warning" @click="toggleSelection()">取消选择</el-button>
      </div>
    </el-main>
  </el-container>
</template>

<script>
import axios from 'axios'

export default {
  data() {
    return {
      searchQuery: '', // 搜索框输入内容
      selectedOperation: '', // 下拉选择框的操作值
      options: [ // 下拉选择框的选项，操作实例
        { value: 'stop', label: '停止实例' },
        { value: 'start', label: '启动实例' },
        { value: 'restart', label: '重启实例' },
        { value: 'delete', label: '删除实例' }
      ],
      tableData: [], // 完整的表格数据
      filteredTableData: [], // 过滤后的表格数据
      displayedTableData: [], // 当前页面显示的数据
      multipleSelection: [], // 选中的实例
      currentPage: 1, // 当前页码
      pageSize: 10, // 每页显示的条数
      totalItems: 0 // 数据总数
    }
  },
  mounted() {
    this.fetchData()
  },
  methods: {
    statusColor(state) {
      switch (state) {
        case '启动中':
          return 'success'
        case '已停止':
          return 'danger'
        default:
          return 'info'
      }
    },
    handlePaginationChange(page) {
      this.currentPage = page
      this.updateTableData()
    },
    handlePageSizeChange(size) {
      this.pageSize = size
      this.updateTableData()
    },
    updateTableData() {
      const start = (this.currentPage - 1) * this.pageSize
      const end = start + this.pageSize
      this.displayedTableData = this.filteredTableData.slice(start, end)
    },
    toggleSelection(rows) {
      if (rows) {
        // 先清空之前的选择
        this.$refs.multipleTable.clearSelection()
        rows.forEach(row => {
          this.$refs.multipleTable.toggleRowSelection(row, true) // 选中
        })
      } else {
        this.$refs.multipleTable.clearSelection() // 取消所有选择
      }
    },
    handleSelectionChange(val) {
      this.multipleSelection = val
    },
    handleSearch() {
      const query = this.searchQuery.trim().toLowerCase()

      if (query) {
        // 正则表达式匹配，支持多个字段搜索：实例名称、IP地址、区域和实例类型
        const regex = new RegExp(query, 'i') // 'i' 表示忽略大小写
        this.filteredTableData = this.tableData.filter(item => {
          return (
            regex.test(item.iname) || // 匹配实例名称
            regex.test(item.ip) || // 匹配IP地址
            regex.test(item.region) || // 匹配区域
            regex.test(item.itype) // 匹配实例类型
          )
        })
      } else {
        // 如果没有搜索条件，显示所有数据
        this.filteredTableData = [...this.tableData]
      }

      // 更新分页总数
      this.totalItems = this.filteredTableData.length
      this.currentPage = 1
      this.updateTableData()
    },
    confirmOperation() {
      if (!this.selectedOperation) {
        this.$message.error('请选择一个操作')
        return
      }
      if (this.multipleSelection.length === 0) {
        this.$message.error('请选择至少一个实例')
        return
      }

      const operationText = {
        stop: '停止',
        start: '启动',
        restart: '重启',
        delete: '删除'
      }[this.selectedOperation]

      this.$confirm(`此操作将${operationText}选中的实例, 是否继续?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      })
        .then(() => {
          this.executeOperation() // 用户确认后执行操作
        })
        .catch(() => {
          this.$message({
            type: 'info',
            message: '已取消操作'
          })
        })
    },
    async executeOperation() {
      const selectedInstances = this.multipleSelection.map(item => item.iname)
      const selectedregions = this.multipleSelection.map(item => item.region)

      let apiUrl = ''
      switch (this.selectedOperation) {
        case 'stop':
          apiUrl = 'http://1.92.75.225:8000/info/stop'
          break
        case 'start':
          apiUrl = 'http://1.92.75.225:8000/info/start'
          break
        case 'restart':
          apiUrl = 'http://1.92.75.225:8000/info/reboot'
          break
        case 'delete':
          apiUrl = 'http://1.92.75.225:8000/info/terminate'
          break
        default:
          this.$message.error('无效的操作')
          return
      }

      try {
        const response = await axios.post(apiUrl, {
          stopinput: selectedInstances.map((iname, index) => ({
            iname: iname,
            region: selectedregions[index]
          }))
        })

        this.$message.success(`${this.selectedOperation}操作成功`)
        this.fetchData() // 刷新数据
      } catch (error) {
        this.$message.error(
          `${this.selectedOperation}操作失败: ${
            error.response?.data || error.message
          }`
        )
      }
    },
    async fetchData() {
      try {
        const response = await axios.get('http://1.92.75.225:8000/info', {
          params: {
            info: 'getnow'
          }
        })
        this.tableData = response.data
        this.filteredTableData = [...this.tableData] // 初始化显示所有数据
        this.totalItems = this.filteredTableData.length
        this.currentPage = 1
        this.updateTableData()
      } catch (error) {
        this.$message.error('获取实例数据失败: ' + error.message)
      }
    },
    async refreshData() {
      try {
        await axios.get('http://1.92.75.225:8000/info', {
          params: {
            info: 'update'
          }
        })
        this.$message.success('数据同步成功')
        this.fetchData() // 刷新数据
      } catch (error) {
        this.$message.error('数据同步失败: ' + error.message)
      }
    }
  }
}
</script>

<style>
h2 {
  margin-bottom: 20px;
}

.text-right {
  text-align: right;
}

.el-header {
  background-color: #409eff;
  color: #fff;
  padding: 5px;
  font-size: 20px;
  text-align: center;
  line-height: 0;
}
</style>
