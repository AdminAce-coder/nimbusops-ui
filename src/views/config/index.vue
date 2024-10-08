<template>
    <div>
      <el-page-header content="详情页面" @back="goBack" />
  
      <!-- 步骤导航 -->
      <div class="steps-container">
        <el-steps :active="2" align-center>
          <el-step title="步骤1" description="在云平台为IAM用户创建AccessKey"></el-step>
          <el-step title="步骤2" description="点击添加按钮新增配置"></el-step>
        </el-steps>
      </div>
  
      <!-- 新增配置按钮 -->
      <div class="button-container">
        <el-button type="primary" @click="dialog = true">
          新增配置<i class="el-icon-upload el-icon--right"></i>
        </el-button>
      </div>
  
      <!-- 抽屉内容 -->
      <el-drawer
        title="实例配置"
        :before-close="handleClose"
        :visible.sync="dialog"
        direction="ltr"
        custom-class="demo-drawer"
        ref="drawer"
        size="50%"
      >
        <div class="demo-drawer__content">
          <el-form :model="form" label-width="180px">
            <!-- ACCESS_KEY_ID 输入框 -->
            <el-form-item label="ACCESS_KEY_ID">
              <el-input v-model="form.accessKeyId" autocomplete="off"></el-input>
            </el-form-item>
  
            <!-- SECRET_ACCESS_KEY 输入框 -->
            <el-form-item label="SECRET_ACCESS_KEY">
              <el-input v-model="form.secretAccessKey" autocomplete="off"></el-input>
            </el-form-item>
  
            <!-- 平台选择 -->
            <el-form-item label="平台">
              <el-select v-model="form.platform" placeholder="请选择所属平台">
                <el-option label="AWS" value="AWS"></el-option>
                <el-option label="Aliyun" value="Aliyun"></el-option>
              </el-select>
            </el-form-item>
          </el-form>
  
          <div class="demo-drawer__footer">
            <el-button @click="cancelForm">取 消</el-button>
            <el-button type="primary" @click="submitForm" :loading="loading">
              {{ loading ? '提交中 ...' : '确 定' }}
            </el-button>
          </div>
        </div>
      </el-drawer>
  
      <!-- 表格展示配置信息 -->
      <el-table :data="tableData" border style="width: 100%">
        <el-table-column fixed prop="ID" label="ID" width="150" />
        <el-table-column prop="secret_id" label="AWS_ACCESS_KEY_ID" width="300" />
        <el-table-column prop="secret_key" label="AWS_SECRET_ACCESS_KEY" width="300" />
        <el-table-column prop="platform" label="平台" width="120" />
        <el-table-column prop="active" label="配置生效" width="120">
          <template v-slot="scope">
            <span>{{ scope.row.active ? '已生效' : '' }}</span>
          </template>
        </el-table-column>
        <el-table-column prop="remark" label="备注" width="120">
          <template v-slot="scope">
            <span>{{ scope.row.remark || '' }}</span>
          </template>
        </el-table-column>
  
        <!-- 操作列 -->
        <el-table-column label="操作" width="120">
          <template v-slot="scope">
            <!-- 应用按钮，只有未锁定且未生效时才可点击 -->
            <el-button 
              type="text" 
              size="small" 
              @click="handleApply(scope.row)" 
              :disabled="scope.row.locked || scope.row.active"
            >
              应用
            </el-button>
  
            <!-- 编辑按钮下拉菜单 -->
            <el-dropdown @command="handleDropdownCommand(scope.row, $event)">
              <el-button type="text" size="small">
                编辑<i class="el-icon-arrow-down el-icon--right"></i>
              </el-button>
              <el-dropdown-menu slot="dropdown">
                <el-dropdown-item command="delete">删除配置</el-dropdown-item>
                <el-dropdown-item command="cancel">取消配置应用</el-dropdown-item>
              </el-dropdown-menu>
            </el-dropdown>
          </template>
        </el-table-column>
      </el-table>
    </div>
  </template>
  
  <script>
  import axios from 'axios'; // 确保引入axios
  
  export default {
    data() {
      return {
        dialog: false,   // 控制抽屉的显示
        loading: false,  // 控制提交按钮的加载状态
        form: {
          accessKeyId: '',      // AWS_ACCESS_KEY_ID 的输入值
          secretAccessKey: '',  // AWS_SECRET_ACCESS_KEY 的输入值
          platform: ''          // 平台选择
        },
        tableData: [] // 表格数据从数据库中获取
      };
    },
    methods: {
      // 返回按钮处理
      goBack() {
        console.log('go back');
      },
  
      // 表格中的“应用”按钮事件处理
      async handleApply(row) {
        // 检查当前行是否已经生效，如果生效，则提示并阻止重复操作
        if (row.active) {
          this.$message.info('该配置已经应用，无需重复应用');
          return;
        }
  
        // 取消当前平台的其他生效配置，并锁定所有同平台的配置
        this.tableData.forEach(item => {
          if (item.platform === row.platform) {
            item.active = false;
            item.locked = true; // 锁定相同平台的配置
          }
        });
  
        // 应用新的配置
        row.active = true;
        row.locked = false; // 已应用的配置保持未锁定
        try {
          const response = await axios.post('http://1.92.75.225:8000/config/apply', {
            id: row.ID, 
            secret_id: row.secret_id,
            platform: row.platform
          });
          this.$message.success(`配置 ${row.platform} 已应用`);
          console.log('应用成功:', response.data);
        } catch (error) {
          console.error('应用失败:', error);
          this.$message.error('配置应用失败: ' + error.message);
        }
      },
  
      // 处理取消应用的逻辑
      async handleCancel(row) {
        if (!row.active) {
          this.$message.info('该配置未应用，无法取消');
          return;
        }
  
        row.active = false; // 取消应用
        row.locked = false; // 解除锁定
  
        // 解锁相同平台的其他配置
        this.tableData.forEach(item => {
          if (item.platform === row.platform) {
            item.locked = false; // 解锁所有同平台的配置
          }
        });
  
        this.$message.success('应用已取消');
      },
  
      // 处理删除操作
      async handleDelete(row) {
        try {
          // 使用 row 数据构建 payload
          const payload = {
            input: {
            secret_id: row.secret_id,   // 使用 row 中的 secret_id
            secret_key: row.secret_key, // 使用 row 中的 secret_key
            platform: row.platform      // 使用 row 中的 platform
          }
          };
          
          // 调用后端删除接口
          await axios.post('http://1.92.75.225:8000/config/delete', payload);
  
          this.$message.success(`配置已删除`);
          
          // 刷新表格数据
          this.fetchData();
        } catch (error) {
          console.error('删除失败:', error);
          this.$message.error('删除失败: ' + error.message);
        }
      },
  
      // 处理下拉菜单的命令
      async handleDropdownCommand(row, command) {
        if (command === 'delete') {
          this.handleDelete(row); // 调用删除处理
        }
        if (command === 'cancel') {
          this.handleCancel(row); // 调用取消应用处理
        }
      },
  
      // 抽屉关闭时的处理
      handleClose(done) {
        if (this.loading) {
          return;
        }
        this.$confirm('确定要提交表单吗？')
          .then(() => {
            this.loading = true;
            this.timer = setTimeout(() => {
              done();
              setTimeout(() => {
                this.loading = false;
              }, 400);
            }, 2000);
          })
          .catch(() => {});
      },
  
      // 取消表单处理
      cancelForm() {
        this.loading = false;
        this.dialog = false;
        clearTimeout(this.timer);
      },
  
      // 获取配置数据
      async fetchData() {
        this.loading = true;
        try {
          const response = await axios.get('http://1.92.75.225:8000/config/query');
          if (Array.isArray(response.data)) {
            this.tableData = response.data.map((item, index) => ({
              ID: index + 1,
              secret_id: item.secret_id, 
              secret_key: item.secret_key, 
              platform: item.platform,
              active: false,  // 默认未生效
              locked: false,  // 默认未锁定
              remark: ''      // 备注默认为空
            }));
          } else {
            this.$message.error('返回的数据格式不正确');
          }
        } catch (error) {
          console.error('获取数据失败:', error);
          this.$message.error('获取实例数据失败: ' + error.message);
        } finally {
          this.loading = false;
        }
      },
  
      // 表单提交函数
      async submitForm() {
        this.loading = true;
        try {
          const payload = {
            secret_id: this.form.accessKeyId,
            secret_key: this.form.secretAccessKey,
            platform: this.form.platform
          };
          const response = await axios.post('http://1.92.75.225:8000/config/create', payload);
          this.$message.success('新增配置成功');
          this.dialog = false;
          this.fetchData();
          // 重置表单
          this.form = {
            accessKeyId: '',
            secretAccessKey: '',
            platform: ''
          };
        } catch (error) {
          console.error('新增配置失败:', error);
          this.$message.error('新增配置失败: ' + error.message);
        } finally {
          this.loading = false;
        }
      }
    },
    mounted() {
      // 组件挂载后获取数据
      this.fetchData();
    }
  };
  </script>
  
  <style scoped>
  .steps-container {
    margin-top: 20px;
    margin-bottom: 50px;
  }
  
  .button-container {
    display: flex;
    justify-content: flex-end;
    margin-bottom: 20px;
  }
  
  .demo-drawer__content {
    padding: 20px;
  }
  
  .demo-drawer__footer {
    text-align: right;
    margin-top: 20px;
  }
  </style>
  