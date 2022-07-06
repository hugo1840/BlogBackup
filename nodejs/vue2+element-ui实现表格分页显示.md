@[TOC](vue2+element-ui实现表格分页显示)

# 添加分页el-pagination
编辑`FundList.vue`，给表格添加分页：
```html
 <template>
    <div class="fillContainer">
        <div>
            <el-form :inline="true" ref="add_data">
                <el-form-item class="btnRight">
                    <el-button type="primary"
                               size="small"
                               icon="view"
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
                <!--此处省略-->
            </el-table>
            <!--分页-->
            <el-row>
                <el-col :span="24">
                    <div class="pagination">
                        <el-pagination @size-change="handleSizeChange"
                                       @current-change="handleCurrentChange"
                                       :current-page="paginations.page_index"
                                       :page-sizes="paginations.page_sizes"
                                       :page-size="paginations.page_size"
                                       :layout="paginations.layout"
                                       :total="paginations.total">
                        </el-pagination>
                    </div>
                </el-col>
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
                paginations: {
                    page_index: 1, //当前页面
                    total: 0,      //总页数
                    page_size: 5,  //每页显示的记录条数
                    page_sizes: [5, 10, 15, 20],  //可选择的page_size范围
                    layout: "total, sizes, prev, pager, next, jumper"  //翻页属性
                },
                tableData: [],
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
                        this.tableData = res.data;
                    })
                    .catch(err => console.log(err));
            },
            handleEdit(index, row) {
                //console.log("HANDLE EDIT");
                this.dialogue = {
                    show: true,
                    title: "修改资金信息",
                    option: "edit"
                };
                this.formData = {
                    type: row.type,
                    description: row.description,
                    income: row.income,
                    expense: row.expense,
                    cash: row.cash,
                    remark: row.remark,
                    id: row._id
                };
            },
            handleDelete(index, row) {
                //console.log("HANDLE DELETE");
                this.$axios.delete(`/api/profiles/delete/${row._id}`)
                    .then(res => {
                        this.$message("删除成功");
                        this.getProfile();
                    })
            },
            handleAdd() {
                //console.log("HANDLE ADD");
                this.dialogue = {
                    show: true,
                    title: "添加资金信息",
                    option: "add"
                };
                this.formData = {
                    type: '',
                    description: '',
                    income: '',
                    expense: '',
                    cash: '',
                    remark: '',
                    id: ''
                };
            },
            handleSizeChange(page_size) {
                //
            },
            handleCurrentChange(page) {
                //
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

    .btnRight {
        float: right;
    }

    .pagination {
        text-align: right;
        margin-top: 10px;
    }
</style>
```

# 实现分页相关方法
引入`allTableData`，并完善`handleSizeChange`、`handleCurrentChange`、`setPaginations`方法。
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
                paginations: {
                    page_index: 1, //当前页面
                    total: 0,      //总页数
                    page_size: 5,  //每页显示的记录条数
                    page_sizes: [5, 10, 15, 20],  //可选择的page_size范围
                    layout: "total, sizes, prev, pager, next, jumper"  //翻页属性
                },
                tableData: [],
                allTableData: [],
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
                        //设置分页数据
                        this.setPaginations();
                    })
                    .catch(err => console.log(err));
            },
            handleEdit(index, row) {
                //console.log("HANDLE EDIT");
                this.dialogue = {
                    show: true,
                    title: "修改资金信息",
                    option: "edit"
                };
                this.formData = {
                    type: row.type,
                    description: row.description,
                    income: row.income,
                    expense: row.expense,
                    cash: row.cash,
                    remark: row.remark,
                    id: row._id
                };
            },
            handleDelete(index, row) {
                //console.log("HANDLE DELETE");
                this.$axios.delete(`/api/profiles/delete/${row._id}`)
                    .then(res => {
                        this.$message("删除成功");
                        this.getProfile();
                    })
            },
            handleAdd() {
                //console.log("HANDLE ADD");
                this.dialogue = {
                    show: true,
                    title: "添加资金信息",
                    option: "add"
                };
                this.formData = {
                    type: '',
                    description: '',
                    income: '',
                    expense: '',
                    cash: '',
                    remark: '',
                    id: ''
                };
            },
            handleSizeChange(page_size) {
                this.paginations.page_index = 1;
                this.paginations.page_size = page_size;
                this.tableData = this.allTableData.filter((item, index) => {
                    return index < page_size;
                });
            },
            handleCurrentChange(page) {
                let index = this.paginations.page_size * (page - 1);
                let nums = this.paginations.page_size * page;
                let tables = [];   // 存储跳转页面要显示的记录

                for (let i = index; i < nums; i++) {
                    if (this.allTableData[i]) {
                        tables.push(this.allTableData[i]);
                    }
                    this.tableData = tables;
                }
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




