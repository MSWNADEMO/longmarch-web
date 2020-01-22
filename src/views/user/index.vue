<template>
  <div class="app-container">
    <div class="filter-container">
      <el-input v-model="listQuery.fuzzySearch" :placeholder="$t('userInfo.username')" style="width: 200px;" class="filter-item" @keyup.enter.native="handleFilter" />
      <el-button v-waves class="filter-item" type="primary" icon="el-icon-search" @click="handleFilter">
        {{ $t('table.search') }}
      </el-button>
      <el-button v-permission="['sys:user:create']" class="filter-item" style="margin-left: 10px;" type="primary" icon="el-icon-edit" @click="handleCreate">
        {{ $t('table.add') }}
      </el-button>
      <el-button v-permission="['sys:user:delete']" :disabled="batchDeleteButtonStatus" class="filter-item" style="margin-left: 10px;" type="primary" icon="el-icon-edit" @click="handleDelete()">
        {{ $t('table.batchDelete') }}
      </el-button>
    </div>
    <el-table
      :key="tableKey"
      v-loading="listLoading"
      :data="list"
      border
      fit
      highlight-current-row
      style="width: 100%;"
      @sort-change="sortChange"
      @selection-change="handleSelectionChange"
    >
      <el-table-column
        type="selection"
        width="55"
      />
      <el-table-column :label="$t('userInfo.username')" width="100px">
        <template slot-scope="scope">
          <span>{{ scope.row.username }}</span>
        </template>
      </el-table-column>
      <el-table-column :label="$t('userInfo.roleNames')" min-width="100px">
        <template slot-scope="scope">
          <span>{{ scope.row.roleNames }}</span>
        </template>
      </el-table-column>
      <el-table-column :label="$t('userInfo.phone')" min-width="120px">
        <template slot-scope="scope">
          <span>{{ scope.row.phone }}</span>
        </template>
      </el-table-column>
      <el-table-column :label="$t('userInfo.status')" width="80px" align="center">
        <template slot-scope="scope">
          <el-tag :type="scope.row.status | dictFirst(dictionary.style_dict)">
            <span>{{ scope.row.status | dictFirst(dictionary.status_dict) }}</span>
          </el-tag>
        </template>
      </el-table-column>
      <el-table-column :label="$t('userInfo.loginCount')" width="80px" align="center">
        <template slot-scope="scope">
          <span>{{ scope.row.loginCount }}</span>
        </template>
      </el-table-column>
      <el-table-column :label="$t('userInfo.lastLoginTime')" width="180px" align="center">
        <template slot-scope="scope">
          <span>{{ scope.row.lastLoginTime }}</span>
        </template>
      </el-table-column>
      <!-- <el-table-column :label="$t('userInfo.createTime')" width="180px" align="center">
        <template slot-scope="scope">
          <span>{{ scope.row.createTime }}</span>
        </template>
      </el-table-column> -->
      <el-table-column :label="$t('table.actions')" align="center" width="230" class-name="small-padding fixed-width">
        <template slot-scope="{row}">
          <el-button v-permission="['sys:user:update']" class="filter-item" style="margin-left: 10px;" type="primary" @click="handleUpdate(row)">
            {{ $t('table.edit') }}
          </el-button>
          <el-button v-if="row.userStatus=='0'" v-permission="['sys:user:publish']" class="filter-item" style="margin-left: 10px;" type="success" @click="handleModifyStatus(row,'published')">
            {{ $t('table.publish') }}
          </el-button>
          <el-button v-if="row.userStatus=='1'" v-permission="['sys:user:draft']" class="filter-item" style="margin-left: 10px;" @click="handleModifyStatus(row,'draft')">
            {{ $t('table.draft') }}
          </el-button>
          <el-button v-permission="['sys:user:delete']" class="filter-item" style="margin-left: 10px;" type="danger" @click="handleDelete(row)">
            {{ $t('table.delete') }}
          </el-button>
        </template>
      </el-table-column>
    </el-table>
    <pagination v-show="total>0" :total="total" :page.sync="listQuery.current" :limit.sync="listQuery.size" @pagination="getList" />
    <el-dialog :title="textMap[dialogStatus]" :visible.sync="dialogFormVisible">
      <el-form ref="dataForm" :rules="rules" :model="temp" label-position="right" label-width="80px" style="width: 400px; margin-left:50px;">
        <el-form-item :label="$t('userInfo.username')" prop="username">
          <el-input v-model="temp.username" :disabled="dialogStatus==='update'" />
        </el-form-item>
        <el-form-item :label="$t('userInfo.status')">
          <el-select v-model="temp.status" class="filter-item">
            <el-option v-for="item in dictionary.status_dict" :key="item.value" :label="item.label" :value="item.value" />
          </el-select>
        </el-form-item>
        <el-tooltip class="item" effect="dark" :content="$t('contentMessage.roleMsg')" placement="top">
          <el-form-item :label="$t('contentMessage.roleLabel')" prop="roleIds">
            <el-select v-model="selectedRoles" multiple :placeholder="$t('contentMessage.select')">
              <el-option
                v-for="item in roleList"
                :key="item.value"
                :label="item.label"
                :value="item.value"
              />
            </el-select>
          </el-form-item>
        </el-tooltip>
        <el-form-item :label="$t('userInfo.headImgUrl')" prop="headImgUrl">
          <el-upload
            class="avatar-uploader"
            :action="uploadActionUrl"
            :show-file-list="false"
            :with-credentials="true"
            :on-success="handlePictureCardPreview"
          >
            <img v-if="temp.headImgUrl" style="margin-top: 6px;border-radius: 100px;width: 100px; height: 100px" :src="temp.headImgUrl" class="avatar">
            <i v-else class="el-icon-plus avatar-uploader-icon" />
          </el-upload>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="dialogFormVisible = false">
          {{ $t('table.cancel') }}
        </el-button>
        <el-button type="primary" @click="dialogStatus==='create'?createData():updateData()">
          {{ $t('table.confirm') }}
        </el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import permission from '@/directive/permission/index.js'
import { fetchList, create, update, remove, loadRoles } from '@/api/SysUser'
import waves from '@/directive/waves'
import Pagination from '@/components/Pagination'
import { mapGetters } from 'vuex'

export default {
  name: 'UserManage',
  components: { Pagination },
  directives: { waves, permission },
  filters: {
    dictFirst(status, dict) {
      const statusMap = {}
      dict.forEach(item => {
        statusMap[item.value] = item.label
      })
      return statusMap[status]
    }
  },
  data() {
    return {
      tableKey: 0,
      list: null,
      total: 0,
      listLoading: true,
      listQuery: {
        current: 1,
        size: 10,
        fuzzySearch: undefined,
        sort: '+id'
      },
      ids: [],
      temp: {
        id: null,
        username: null,
        password: null,
        status: null,
        headImgUrl: null,
        createTime: null,
        roleIds: null
      },
      dialogFormVisible: false,
      dialogStatus: 'create',
      textMap: {
        update: '编辑用户',
        create: '添加用户'
      },
      rules: {
        username: [{ required: true, message: 'username is required', trigger: 'blur' }]
      },
      roleList: null,
      selectedRoles: null,
      uploadActionUrl: process.env.VUE_APP_BASE_API + '/file/upload'
    }
  },
  computed: {
    ...mapGetters(['dictionary']),
    batchDeleteButtonStatus() {
      return this.ids.length <= 0
    }
  },
  created() {
    this.getList()
  },
  methods: {
    getList() {
      this.listLoading = true
      fetchList(this.listQuery).then(response => {
        this.list = response.data.records
        this.total = response.data.total
        this.listLoading = false
      })
    },
    handleFilter() {
      this.listQuery.current = 1
      this.getList()
    },
    handleModifyStatus(row, status) {
      this.$message({
        message: '操作成功',
        type: 'success'
      })
      row.status = status
    },
    sortChange(data) {
      const { prop, order } = data
      if (prop === 'id') {
        this.sortByID(order)
      }
    },
    sortByID(order) {
      if (order === 'ascending') {
        this.listQuery.sort = '+id'
      } else {
        this.listQuery.sort = '-id'
      }
      this.handleFilter()
    },
    resetTemp() {
      this.temp = {
        id: null,
        username: null,
        status: null,
        headImgUrl: null
      }
    },
    handleCreate() {
      this.selectedRoles = []
      this.resetTemp()
      this.dialogStatus = 'create'
      this.dialogFormVisible = true
      this.$nextTick(() => {
        this.$refs['dataForm'].clearValidate()
      })
      this.loadRoles()
    },
    createData() {
      this.$refs['dataForm'].validate((valid) => {
        if (valid) {
          this.temp.roleIds = this.selectedRoles.join()
          create(this.temp).then(() => {
            this.list.unshift(this.temp)
            this.dialogFormVisible = false
            this.$notify({
              title: '成功',
              message: '创建成功',
              type: 'success',
              duration: 2000
            })
          })
        }
      })
    },
    handleUpdate(row) {
      this.selectedRoles = []
      if (row.roleIds && row.roleIds.split(',') && row.roleIds.split(',').length) {
        row.roleIds.split(',').forEach(item => {
          this.selectedRoles.push(Number.parseInt(item))
        })
      }
      this.temp = Object.assign({}, row)
      this.dialogStatus = 'update'
      this.dialogFormVisible = true
      this.$nextTick(() => {
        this.$refs['dataForm'].clearValidate()
      })
      this.loadRoles()
    },
    updateData() {
      this.$refs['dataForm'].validate((valid) => {
        if (valid) {
          const tempData = Object.assign({}, this.temp)
          tempData.roleIds = this.selectedRoles.join()
          update(tempData).then(() => {
            for (const v of this.list) {
              if (v.id === this.temp.id) {
                const index = this.list.indexOf(v)
                this.list.splice(index, 1, this.temp)
                break
              }
            }
            this.getList()
            this.dialogFormVisible = false
            this.$notify({
              title: '成功',
              message: '更新成功',
              type: 'success',
              duration: 2000
            })
          })
        }
      })
    },
    handleDelete(row) {
      const _ids = row !== undefined ? [row.id] : this.ids.map(obj => {
        return obj.id
      })
      remove(_ids).then(() => {
        this.getList()
        this.dialogFormVisible = false
        this.$notify({
          title: '成功',
          message: '删除成功',
          type: 'success',
          duration: 2000
        })
      })
    },
    handleSelectionChange(val) {
      this.ids = val
    },
    loadRoles() {
      loadRoles().then(response => {
        this.roleList = response.data
      })
    },
    handlePictureCardPreview(response, file, fileList) {
      this.temp.headImgUrl = response.data.url
    }
  }
}
</script>