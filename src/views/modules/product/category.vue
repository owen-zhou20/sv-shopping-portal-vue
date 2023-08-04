<template>
  <div>
    <el-tree
      :data="menus"
      :props="defaultProps"
      :expand-on-click-node="false"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expendedKey"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(data)"
          >
            Append
          </el-button>
          <el-button
            type="text"
            size="mini"
            @click="edit(data)"
          >
            Edit
          </el-button>
          <el-button
            v-if="node.childNodes.length == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >
            Delete
          </el-button>
        </span>
      </span>
    </el-tree>
    <el-dialog title="提示" :visible.sync="dialogVisible" width="30%" :close-on-click-modal="false">
      <el-form :model="category">
        <el-form-item label="Category Name">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="Category Icon">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="Product Unit">
          <el-input v-model="category.productUnit" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="addCategory()">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
// 这里可以导入其他文件（比如：组件，工具js，第三方插件js，json文件，图片文件等等）
// 例如：import 《组件名称》from '《组件路径》';

export default {
  // import 引入的组件需要注入到对象中才能使用
  components: {},
  props: {},
  data () {
    return {
      category: { name: '', parentCid: 0, catLevel: 0, showStatus: 1, sort: 0 },
      dialogVisible: false,
      menus: [],
      expendedKey: [],
      defaultProps: {
        children: 'children',
        label: 'name'
      }
    }
  },
  // 计算属性类似于data 概念
  computed: {},
  // 监控data 中的数据变化
  watch: {},
  // 方法集合
  methods: {
    getMenus () {
      this.$http({
        url: this.$http.adornUrl('/product/category/list/tree'),
        method: 'get'
      }).then(({ data }) => {
        console.log('success to get menus data..', data.data)
        this.menus = data.data
      })
    },
    edit (data) {
      console.log('要修改的数据', data)
    },
    append (data) {
      console.log('append', data)
      this.dialogVisible = true
      this.category.parentCid = data.catId
      this.category.catLevel = data.catLevel * 1 + 1
    },

    addCategory () {
      console.log('提交的三级分类数据', this.category)
      this.$http({
        url: this.$http.adornUrl('/product/category/save'),
        method: 'post',
        data: this.$http.adornData(this.category, false)
      }).then(({ data }) => {
        this.$message({
          type: 'success',
          message: `Success to add menus [${this.category.name}]!`
        })
        this.dialogVisible = false
        this.getMenus()
        this.expendedKey = [this.category.parentCid]
      })
    },

    remove (node, data) {
      var ids = [data.catId]
      this.$confirm(`Do you want to delete [${data.name}] menus?`, 'Notice', {
        confirmButtonText: 'Confirm',
        cancelButtonText: 'Cancel',
        type: 'warning'
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl('/product/category/delete'),
            method: 'post',
            data: this.$http.adornData(ids, false)
          }).then(({ data }) => {
            console.log('success to delete...')
            this.getMenus()
            this.expendedKey = [node.parent.data.catId]
          })
          console.log('remove', node, data)
          this.$message({
            type: 'success',
            message: `Success to delete menus [${data.name}]!`
          })
        })
        .catch(() => {
          this.$message({
            type: 'info',
            message: `Cancel delete menus [${data.name}]!`
          })
        })
    }
  },
  // 生命周期- 创建完成（可以访问当前this 实例）
  created () {
    this.getMenus()
  },
  // 生命周期- 挂载完成（可以访问DOM 元素）
  mounted () {},
  beforeCreate () {}, // 生命周期- 创建之前
  beforeMount () {}, // 生命周期- 挂载之前
  beforeUpdate () {}, // 生命周期- 更新之前
  updated () {}, // 生命周期- 更新之后
  beforeDestroy () {}, // 生命周期- 销毁之前
  destroyed () {}, // 生命周期- 销毁完成
  activated () {} // 如果页面有keep-alive 缓存功能，这个函数会触发
}
</script>
<style scoped>
</style>