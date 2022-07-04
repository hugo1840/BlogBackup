@[TOC](vue2+element-ui配置表格信息展示页)

# 创建表格
编辑`client/src/views/FundList.vue`，创建资金流水表格展示页。从后端接口获取的数据通过element-ui表单组件的`prop`属性绑定到对应列显示。
```html
<template>
    <el-table v-if="tableData.length > 0"
              :data="tableData"
              max-height="450"
              border
              style="width: 100%">
        <el-table-column type="index"
                         label="序号"
                         align="center"
                         width="70">
        </el-table-column>
        <el-table-column prop="date"
                         label="创建时间"
                         align="center"
                         width="240">
        </el-table-column>
        <el-table-column prop="type"
                         label="收支类型"
                         align='center'
                         width="150">
        </el-table-column>
        <el-table-column prop="description"
                         label="收支描述"
                         align='center'
                         width="180">
        </el-table-column>
        <el-table-column prop="income"
                         label="收入"
                         align='center'
                         width="170">
        </el-table-column>
        <el-table-column prop="expense"
                         label="支出"
                         align='center'
                         width="170">
        </el-table-column>
        <el-table-column prop="cash"
                         label="账户现金"
                         align='center'
                         width="170">
        </el-table-column>
        <el-table-column prop="remark"
                         label="备注"
                         align='center'
                         width="220">
        </el-table-column>
    </el-table>
</template>

<script>
    export default {
        name: "fundList",
        data() {
            return {
                tableData: []
            };
        },
        created() {
            this.getProfile();
        },
        methods: {
            getProfile() {
                this.$axios.get('/api/profiles')
                    .then(res => {
                        this.tableData = res.data;
                    })
                    .catch(err => console.log(err));
            }
        }
    }

</script>
```

编辑路由，添加fundlist：
```javascript
import FundList from '../views/FundList'

Vue.use(VueRouter)

const routes = [
  {
    path: '/',
    redirect: 'index'
  },
  {
    path: '/index',
    name: 'index',
    component: Index,
      children: [
          { path: '', component: Home },
          { path: '/home', name: 'home', component: Home },
          { path: '/infoshow', name: 'infoshow', component: InfoShow },
          { path: '/fundlist', name: 'fundlist', component: FundList }
      ]
   },
  {
      path: '/register',
      name: 'register',
      component: Register
    },
  {
        path: '/login',
        name: 'login',
        component: Login
    },
  {
      path: '*',
      name: '/404',
      component: NotFound
  }
]
```

# 配置样式
编辑`Index.vue`，为`router-view`配置样式：
```html
<template>
    <div class="index">
        <HeadNav></HeadNav>
        <LeftMenu></LeftMenu>
        <div class="rightContainer">
            <router-view></router-view>
        </div>
    </div>
</template>

<script>
    import HeadNav from "../components/HeadNav"
    import LeftMenu from "../components/LeftMenu"
    export default {
      name: 'index',
      components: {
          HeadNav,
          LeftMenu
      }
    };
</script>

<style>
    .index {
        width: 100%;
        height: 100%;
        overflow: hidden;
    }
    .rightContainer {
        position: relative;
        top: 0;
        left: 180px;
        width: calc(100% - 180px);
        height: calc(100% - 71px);
        overflow: auto;
    }
</style>
```

# 添加按钮
编辑`FundList.vue`，优化表格细节，并添加按钮：
```html
<template>
    <div class="fillContainer">
        <el-table v-if="tableData.length > 0"
                  :data="tableData"
                  max-height="450"
                  border
                  style="width: 100%">
            <el-table-column type="index"
                             label="序号"
                             align="center"
                             width="70">
            </el-table-column>
            <el-table-column prop="date"
                             label="创建时间"
                             align="center"
                             width="240">
                <template slot-scope="scope">
                    <i class="el-icon-time"></i>
                    <span style="margin-left: 10px">{{ scope.row.date }}</span>
                </template>
            </el-table-column>
            <el-table-column prop="type"
                             label="收支类型"
                             align='center'
                             width="150">
            </el-table-column>
            <el-table-column prop="description"
                             label="收支描述"
                             align='center'
                             width="200">
            </el-table-column>
            <el-table-column prop="income"
                             label="收入"
                             align='center'
                             width="110">
                <template slot-scope="scope">
                    <span style="color: #00d053">{{ scope.row.income }}</span>
                </template>
            </el-table-column>
            <el-table-column prop="expense"
                             label="支出"
                             align='center'
                             width="110">
                <template slot-scope="scope">
                    <span style="color: #f56767">{{ scope.row.expense }}</span>
                </template>
            </el-table-column>
            <el-table-column prop="cash"
                             label="账户现金"
                             align='center'
                             width="110">
                <template slot-scope="scope">
                    <span style="color: #4db3ff">{{ scope.row.cash }}</span>
                </template>
            </el-table-column>
            <el-table-column prop="remark"
                             label="备注"
                             align='center'
                             width="220">
            </el-table-column>
            <el-table-column prop="operation"
                             align="center"
                             fixed="right"
                             width="180"
                             label="操作">
                <template slot-scope="scope">
                    <el-button size="small"
                               icon="edit"
                               @click="handleEdit(scope.$index, scope.row)">编辑</el-button>
                    <el-button size="small"
                               icon="delete"
                               type="danger"
                               @click="handleDelete(scope.$index, scope.row)">删除</el-button>
                </template>
            </el-table-column>
        </el-table>
    </div>
</template>

<script>
    export default {
        name: "fundList",
        data() {
            return {
                tableData: []
            };
        },
        created() {
            this.getProfile();
        },
        methods: {
            getProfile() {
                this.$axios.get('/api/profiles')
                    .then(res => {
                        this.tableData = res.data;
                    })
                    .catch(err => console.log(err));
            },
            handleEdit(index, row) {
                console.log("HANDLE EDIT");
            },
            handleDelete(index, row) {
                console.log("HANDLE DELETE");
            }
        }
    }

</script>

<style scoped>
    .fillContainer {
        width: 100%;
        height: 100%;
        padding: 16px;
        box-sizing: border-box;
    }
</style>
```










