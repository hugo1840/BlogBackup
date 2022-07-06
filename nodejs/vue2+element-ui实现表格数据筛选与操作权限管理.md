
@[TOC](vue2+element-ui实现表格数据筛选与操作权限管理)

# 添加el-date-picker
编辑`FundList.vue`，添加时间筛选组件：
```html
<template>
    <div class="fillContainer">
        <div>
            <el-form :inline="true" ref="add_data" :model="search_data">
                <el-form-item label="按时间筛选">
                    <el-date-picker v-model="search_data.startTime"
                                    type="datetime"
                                    placeholder="选择开始时间">
                    </el-date-picker>
                    --
                    <el-date-picker v-model="search_data.endTime"
                                    type="datetime"
                                    placeholder="选择结束时间">
                    </el-date-picker>
                </el-form-item>
                <el-form-item>
                    <el-button type="primary"
                               size="small"
                               icon="search"
                               @click="handleSearch()">筛选</el-button>
                </el-form-item>
                <el-form-item class="btnRight">
                    <el-button type="primary"
                               size="small"
                               icon="view"
                               @click="handleAdd()">添加</el-button>
                </el-form-item>
            </el-form>
        </div>
        <div class="table_container">
             <!--此处省略-->
        </div>
        <Dialog :dialogue="dialogue" :formData="formData" @update="getProfile"></Dialog>
    </div>
</template>
```

# 实现筛选方法
补充根据时间筛选表格数据的方法`handleSearch`：
```html
<script>
    import Dialog from '../components/Dialog';
    export default {
        name: "fundList",
        components: {
            Dialog
        },
        data() {
            return {
                search_data: {
                    startTime: "",
                    endTime: ""
                },
                paginations: {
                    page_index: 1, //当前页面
                    total: 0,      //总页数
                    page_size: 5,  //每页显示的记录条数
                    page_sizes: [5, 10, 15, 20],  //可选择的page_size范围
                    layout: "total, sizes, prev, pager, next, jumper"  //翻页属性
                },
                tableData: [],
                allTableData: [],
                filterTableData: [],
                formData: {
                    type: "",
                    description: "",
                    income: "",
                    expense: "",
                    cash: "",
                    remark: "",
                    id: ""
                },
                dialogue: {
                    show: false,
                    title: '',
                    option: 'edit'
                }
            };
        },
        created() {
            this.getProfile();
        },
        methods: {
            getProfile() {
                this.$axios.get('/api/profiles')
                    .then(res => {
                        this.allTableData = res.data;
                        this.filterTableData = res.data;
                        //设置分页数据
                        this.setPaginations();
                    })
                    .catch(err => console.log(err));
            },
            handleSearch() {
                if (!this.search_data.startTime || !this.search_data.endTime) {
                    this.$message({
                        type: "warning",
                        message: "请选择时间区间"
                    });
                    this.getProfile();
                    return;
                }

                const start_t = this.search_data.startTime.getTime();
                const end_t = this.search_data.endTime.getTime();
                this.allTableData = this.filterTableData.filter(item => {
                    //console.log(item);
                    let datee = new Date(item.date);
                    let timee = datee.getTime();
                    return timee >= start_t && timee <= end_t;
                });

                this.setPaginations();
            },
            handleEdit(index, row) {
                //...
            },
            handleDelete(index, row) {
                //...
            },
            handleAdd() {
                //...
            },
            handleSizeChange(page_size) {
                //...
            },
            handleCurrentChange(page) {
                //...
            },
            setPaginations() {
                this.paginations.total = this.allTableData.length;
                this.paginations.page_index = 1;
                this.paginations.page_size = 5;
                //设置默认的分页数据
                this.tableData = this.allTableData.filter((item, index) => {
                    return index < this.paginations.page_size;
                });
            }
        }
    }

</script>
```


# 限制增删改权限
通过Vuex Store获取用户身份，使用`v-if`给增删改操作加上权限管理，只允许身份为`manager`的用户操作。
```html
<template>
    <div class="fillContainer">
        <div>
            <el-form :inline="true" ref="add_data" :model="search_data">
                <el-form-item label="按时间筛选">
                    <el-date-picker v-model="search_data.startTime"
                                    type="datetime"
                                    placeholder="选择开始时间">
                    </el-date-picker>
                    --
                    <el-date-picker v-model="search_data.endTime"
                                    type="datetime"
                                    placeholder="选择结束时间">
                    </el-date-picker>
                </el-form-item>
                <el-form-item>
                    <el-button type="primary"
                               size="small"
                               icon="search"
                               @click="handleSearch()">筛选</el-button>
                </el-form-item>
                <el-form-item class="btnRight">
                    <el-button type="primary"
                               size="small"
                               icon="view"
                               v-if="user.identity == 'manager'"
                               @click="handleAdd()">添加</el-button>
                </el-form-item>
            </el-form>
        </div>
        <div class="table_container">
            <el-table v-if="tableData.length > 0"
                      :data="tableData"
                      max-height="450"
                      border
                      style="width: 100%">
                <!--省略表格其他列-->
                <el-table-column prop="operation"
                                 align="center"
                                 fixed="right"
                                 width="180"
                                 label="操作"
                                 v-if="user.identity == 'manager'">
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
            <el-row>
                <!--分页-->
            </el-row>
        </div>

        <Dialog :dialogue="dialogue" :formData="formData" @update="getProfile"></Dialog>
    </div>
</template>

<script>
    import Dialog from '../components/Dialog';
    export default {
        name: "fundList",
        components: {
            Dialog
        },
        data() {
            return {
                //...
            };
        },
        computed: {
            user() {
                return this.$store.getters.user;
            }
        },
        created() {
            this.getProfile();
        },
        methods: {
            //...
        }
    }

</script>
```


