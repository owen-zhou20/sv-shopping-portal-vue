<template>
  <div>
    <el-switch
      v-model="draggable"
      active-text="Can drag"
      inactive-text="Can't drag">
    </el-switch>
    <el-button v-if="draggable" @click="batchSave">
      Batch Save
    </el-button>
    <el-button type="danger" @click="batchDelete">Batch Delete</el-button>
    <el-tree :data="menus"
             :props="defaultProps"
             :expand-on-click-node="false"
             show-checkbox
             node-key="catId"
             :default-expanded-keys="expendedKey"
             :draggable="draggable"
             :allow-drop="allowDrop"
             @node-drop="handleDrop"
             ref="menuTree">
      <span class="custom-tree-node"
            slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button v-if="node.level <= 2"
                     type="text"
                     size="mini"
                     @click="() => append(data)">
            Append
          </el-button>
          <el-button type="text"
                     size="mini"
                     @click="edit(data)">
            Edit
          </el-button>
          <el-button v-if="node.childNodes.length == 0"
                     type="text"
                     size="mini"
                     @click="() => remove(node, data)">
            Delete
          </el-button>
        </span>
      </span>
    </el-tree>
    <el-dialog :title="title"
               :visible.sync="dialogVisible"
               width="30%"
               :close-on-click-modal="false">
      <el-form :model="category">
        <el-form-item label="Category Name">
          <el-input v-model="category.name"
                    autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="Category Icon">
          <el-input v-model="category.icon"
                    autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="Product Unit">
          <el-input v-model="category.productUnit"
                    autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer"
            class="dialog-footer">
        <el-button @click="dialogVisible = false">Cancel</el-button>
        <el-button type="primary"
                   @click="submitData()">Confirm</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  components: {},
  props: {},
  data () {
    return {
      pCid: [],
      draggable: false,
      category: {
        name: '',
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        icon: '',
        productUnit: '',
        sort: 0,
        catId: null
      },
      title: '',
      dialogType: '',
      dialogVisible: false,
      menus: [],
      expendedKey: [],
      defaultProps: {
        children: 'children',
        label: 'name'
      },
      maxLevel: 0,
      updateNodes: []
    }
  },
  computed: {},
  watch: {},
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
    batchDelete () {
      let catIds = []
      let catNames = []
      let checkedNodes = this.$refs.menuTree.getCheckedNodes()
      for (let i = 0; i < checkedNodes.length; i++) {
        catIds.push(checkedNodes[i].catId)
        catNames.push(checkedNodes[i].name)
      }
      this.$confirm(`Do you want to batch delete [${catNames}] menus?`, 'Notice', {
        confirmButtonText: 'Confirm',
        cancelButtonText: 'Cancel',
        type: 'warning'
      })
      .then(() => {
        this.$http({
          url: this.$http.adornUrl('/product/category/delete'),
          method: 'post',
          data: this.$http.adornData(catIds, false)
        }).then(({ data }) => {
          this.$message({
            type: 'success',
            message: `Success to batch delete menus!`
          })
          this.getMenus()
        })
      })
      .catch(() => {

      })
    },
    batchSave () {
      this.$http({
        url: this.$http.adornUrl('/product/category/update/sort'),
        method: 'post',
        data: this.$http.adornData(this.updateNodes, false)
      }).then(({ data }) => {
        this.$message({
          type: 'success',
          message: `Success to change menus!`
        })
        this.getMenus()
        this.expendedKey = this.pCid
        this.maxLevel = 0
        this.updateNodes = []
        // this.pCid = 0
      })
    },
    handleDrop (draggingNode, dropNode, dropType, ev) {
      let pCid = 0
      let siblings = null
      if (dropType == 'inner') {
        pCid = dropNode.data.catId
        siblings = dropNode.childNodes
      } else {
        pCid = dropNode.parent.data.catId == undefined ? 0 : dropNode.parent.data.catId
        siblings = dropNode.parent.childNodes
      }
      this.pCid.push(pCid)

      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId == draggingNode.data.catId) {
          let catLevel = draggingNode.level
          if (siblings[i].level != draggingNode.level) {
            catLevel = siblings[i].level
            this.updateChildNodeLevel(siblings[i])
          }
          this.updateNodes.push({catId: siblings[i].data.catId, sort: i, parentCid: pCid, catLevel: catLevel})
        } else {
          this.updateNodes.push({catId: siblings[i].data.catId, sort: i})
        }
      }
    },
    updateChildNodeLevel (node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data
          this.updateNodes.push({catId: cNode.catId, catLevel: node.childNodes[i].level})
          this.updateChildNodeLevel(node.childNodes[i])
        }
      }
    },
    allowDrop (draggingNode, dropNode, type) {
      this.maxLevel = 0
      this.updateNodes = []
      this.countNodeLevel(draggingNode)
      let deep = Math.abs(this.maxLevel - draggingNode.level) + 1
      if (type == 'inner') {
        return (deep + dropNode.level) <= 3
      } else {
        return (deep + dropNode.parent.level) <= 3
      }
    },
    countNodeLevel (node) {
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level
          }
          this.countNodeLevel(node.childNodes[i])
        }
      }
    },
    submitData () {
      if (this.dialogType == 'add') {
        this.addCategory()
      } else if (this.dialogType == 'edit') {
        this.editCategory()
      }
    },
    editCategory () {
      var {catId, name, icon, productUnit} = this.category
      this.$http({
        url: this.$http.adornUrl('/product/category/update'),
        method: 'post',
        data: this.$http.adornData({catId, name, icon, productUnit}, false)
      }).then(({ data }) => {
        this.$message({
          type: 'success',
          message: `Success to edit menus [${this.category.name}]!`
        })
        this.dialogVisible = false
        this.getMenus()
        this.expendedKey = [this.category.parentCid]
      })
      this.category.catId = null
      this.category.name = ''
      this.category.icon = ''
      this.category.productUnit = ''
      this.category.parentCid = 0
      this.category.catLevel = 0
      this.category.sort = 0
      this.category.showStatus = 1
    },
    edit (data) {
      this.title = 'Edit Category'
      this.dialogType = 'edit'
      this.dialogVisible = true
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: 'get'
      }).then(({data}) => {
        this.category.catId = data.data.catId
        this.category.name = data.data.name
        this.category.icon = data.data.icon
        this.category.productUnit = data.data.productUnit
        this.category.parentCid = data.data.parentCid
      })
    },
    append (data) {
      this.title = 'Add Category'
      this.dialogType = 'add'
      this.dialogVisible = true
      this.category.parentCid = data.catId
      this.category.catLevel = data.catLevel * 1 + 1
    },

    addCategory () {
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
            this.$message({
              type: 'success',
              message: `Success to delete menus [${data.name}]!`
            })
            this.getMenus()
            this.expendedKey = [node.parent.data.catId]
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
  created () {
    this.getMenus()
  },
  mounted () { },
  beforeCreate () { },
  beforeMount () { },
  beforeUpdate () { },
  updated () { },
  beforeDestroy () { },
  destroyed () { },
  activated () { }
}
</script>
<style scoped></style>