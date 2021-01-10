<template>
  <div>
    <el-switch v-model="draggable" active-text="开启拖拽" inactive-text="关闭拖拽"></el-switch>
    <el-tree
      :data="gategories"
      :props="defaultProps"
      @node-click="handleNodeClick"
      show-checkbox
      node-key="catId"
      :expand-on-click-node=false
      :default-expanded-keys="expandedKeys"
      :draggable="draggable"
      :allow-drop="allowDrop"
      @node-drag-start="nodeDragStart"
      @node-drop="handleDrop"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <!-- 只有一级分类和二级分类有追加 -->
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => toAppend(data)"
          >
            <i class="el-icon-circle-plus"></i>
          </el-button>
          <!-- 移除按钮，只有三级分类或者没有子节点的分类有移除操作 -->
          <el-button
            v-if="node.childNodes.length == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >
            <i class="el-icon-delete"></i>
          </el-button>
          <!-- 修改按钮 -->
          <el-button type="text" size="mini" @click="() => toEdit(data)">
            <i class="el-icon-edit"></i>
          </el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog
      :title="dialogTitle"
      :visible.sync="dialogFormVisible"
      :close-on-click-modal="false"
    >
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input
            v-model="category.productUnit"
            autocomplete="off"
          ></el-input>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="dialogFormVisible = false">取 消</el-button>
        <el-button type="primary" @click="confirm">确 定</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
export default {
  data() {
    return {
      draggable: true,
      dialogTitle: "",
      dialogType: "",
      category: {
        catId: null,
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        icon: "",
        productUnit: "",
      },
      levels: new Set(),
      dialogFormVisible: false,
      gategories: [],
      expandedKeys: [],
      defaultProps: {
        children: "subCategories",
        label: "name",
      },
      toUpdateNodes: [],
      parentCid: []
    };
  },
  methods: {
    handleNodeClick(data) {
      
    },
    getGategories() {
      this.$http({
        url: this.$http.adornUrl("/product/category/tree"),
        method: "get",
      }).then(({ data }) => {
        this.gategories = data.data;
      });
    },
    toAppend(data) {
      this.dialogTitle = "新增分类";
      this.dialogType = "append";
      this.dialogFormVisible = true;
      this.category = {
        catId: null,
        name: "",
        parentCid: data.catId,
        catLevel: data.catLevel * 1 + 1,
        showStatus: 1,
        sort: 0,
        icon: "",
        productUnit: ""
      };
    },
    toEdit(data) {
      this.dialogTitle = "修改分类";
      this.dialogType = "edit";
      this.dialogFormVisible = true;
      // 发送请求查询当前分类的信息
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        this.category.name = data.category.name;
        this.category.catId = data.category.catId;
        this.category.parentCid = data.category.parentCid;
        this.category.icon = data.category.icon;
        this.category.productUnit = data.category.productUnit;
      });
    },
    // 确认提交表单
    confirm() {
      if (this.dialogType === "append") {
        this.save();
      }

      if (this.dialogType === "edit") {
        this.update();
      }
    },
    save() {
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        this.$message({
          type: "success",
          message: "新增成功！",
        });
        // 关闭对话框
        this.dialogFormVisible = false;
        // 重新刷新分类树
        this.getGategories();
        // 同时自动展开当前删除子分类的父分类
        this.expandedKeys = [this.category.parentCid];
      });
    },
    update() {
      let { catId, name, icon, productUnit } = this.category;
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData({ catId, name, icon, productUnit }, false),
      }).then(({ data }) => {
        this.$message({
          type: "success",
          message: "修改成功！",
        });
        this.dialogFormVisible = false;
        this.getGategories();
        this.expandedKeys = [this.category.parentCid];
      });
    },
    remove(node, data) {
      let ids = [data.catId];
      this.$confirm(`请确定是否删除【${data.name}】分类？`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      }).then(() => {
          // 删除请求
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            this.$message({
              type: "success",
              message: "删除成功！",
            });
            // 重新刷新分类树
            this.getGategories();
            // 同时自动展开当前删除子分类的父分类
            this.expandedKeys = [node.parent.data.catId];
          });
      }).catch(() => {});
    },
    allowDrop(draggingNode, dropNode, type) {
      // 判断是否允许拖拽成功的依据是：当前拖拽节点的总层数与目标位置所在父节点的层数之和不大于3
      let levels = this.countLevels(draggingNode);
      if ("inner" == type) {
        return levels + dropNode.level <= 3;
      } else {
        return levels + dropNode.parent.level <= 3;
      }
    },
    countLevels(node, level = 1) { // 统计指定节点的层级数
      // 如果当前节点的所有子节点都没有子节点了，终止递归
      if (node.childNodes.length === 0) {
        this.levels.add(level);
      } else {
        // 递归当前节点的子节点
        node.childNodes.forEach(item => {
          this.countLevels(item, level + 1);
        });
      }
      return Math.max(...Array.from(this.levels));
    },
    nodeDragStart(node, ev) {
      this.levels.clear();
      this.toUpdateNodes.splice(0, this.toUpdateNodes.length);
    },
    updateSubCategoriesLevel(node) {
      if (node.childNodes.length > 0) {
        node.childNodes.forEach(item => {
          let subCategory = item.data;
          this.toUpdateNodes.push({catId: subCategory.catId, catLevel: item.level});
          this.updateSubCategoriesLevel(item);
        });
      }
    },
    handleDrop(draggingNode, dropNode, dropType, ev) {
      let pCid = 0;
      let siblings = null; 
      if ("inner" === dropType) {
        pCid = dropNode.data.catId;
        siblings = dropNode.childNodes;
      } else {
        pCid = dropNode.data.parentCid;
        siblings = dropNode.parent.childNodes;
      }
      this.parentCid = [pCid];
      siblings.forEach((item, index) => {
        if (item.data.catId === draggingNode.data.catId) {
          let catLevel = draggingNode.level;
          // 如果当前拖拽节点的层级发生变化，则当前拖拽节点及其子节点的catLevel都应该变化
          if (item.level != draggingNode.level) {
            catLevel = item.level;
            this.updateSubCategoriesLevel(item);
          }
          this.toUpdateNodes.push({
            catId: item.data.catId,
            sort: index,
            parentCid: pCid,
            catLevel: catLevel
          });
        } else {
          this.toUpdateNodes.push({catId: item.data.catId, sort: index});
        }
      });
      this.$http({
        url: this.$http.adornUrl("/product/category/updateBatch"),
        method: "post",
        data: this.$http.adornData(this.toUpdateNodes, false),
      }).then(({ data }) => {
        this.$message({
          type: "success",
          message: "拖拽成功！",
        });
        this.getGategories();
        this.expandedKeys = [this.parentCid];
      });
      
    }
  },
  // 创建完成时触发
  created() {
    this.getGategories();
  },
};
</script>
<style>
</style>