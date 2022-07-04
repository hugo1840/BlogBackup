@[TOC](vue2+element-ui+axios实现表格添加内容对话框)

# el-form | el-table | el-button
编辑`FundList.vue`，在表格右上方添加按钮来打开对话框：
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
            },
            handleAdd() {
                console.log("HANDLE ADD");
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

# 创建el-dialog对话框
创建`client/src/components/Dialog.vue`，表格数据添加的弹窗：
```html
<template>
    <div class="dialogue">
        <el-dialog title="添加资金信息"
                   :visible="dialogue.show"
                   :close-on-click-modal="false"
                   :modal-append-to-body="false">

        </el-dialog>
    </div>
</template>

<script>
    export default {
        name: "dialogue",
        props: {
            dialogue: Object
        }
    };
</script>

<style scoped>

</style>
```

# 实现点击按钮即打开对话框
编辑`FundList.vue`，引入并使用Dialog组件：
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
                <!--此处省略表格内容-->
            </el-table>
        </div>

        <Dialog :dialogue="dialogue"></Dialog>
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
                tableData: [],
                dialogue: {
                    show: false
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
                console.log("HANDLE EDIT");
            },
            handleDelete(index, row) {
                console.log("HANDLE DELETE");
            },
            handleAdd() {
                //console.log("HANDLE ADD");
                this.dialogue.show = true;
            }
        }
    }

</script>
```


# 提交对话框填写内容
编辑`Dialog.vue`，完善对话框并绑定后端接口`/api/profiles/add`：
```html
<template>
    <div class="dialogue">
        <el-dialog title="添加资金信息"
                   :visible.sync="dialogue.show"
                   :close-on-click-modal="false"
                   :modal-append-to-body="false">
            <div class="form">
                <el-form ref="form"
                         :model="formData"
                         :rules="form_rules"
                         label-width="120px"
                         style="margin:10px;width:auto;">
                    <el-form-item label="收支类型：">
                        <el-select v-model="formData.type" placeholder="收支类型">
                            <el-option v-for="(formtype, index) in format_type_list"
                                       :key="index"
                                       :label="formtype"
                                       :value="formtype">
                            </el-option>
                        </el-select>
                    </el-form-item>
                    <el-form-item prop="description" label="收支描述：">
                        <el-input type="description" v-model="formData.description"></el-input>
                    </el-form-item>
                    <el-form-item prop="income" label="收入：">
                        <el-input type="income" v-model="formData.income"></el-input>
                    </el-form-item>
                    <el-form-item prop="expense" label="支出：">
                        <el-input type="expense" v-model="formData.expense"></el-input>
                    </el-form-item>
                    <el-form-item prop="cash" label="账户现金：">
                        <el-input type="cash" v-model="formData.cash"></el-input>
                    </el-form-item>
                    <el-form-item label="备注：">
                        <el-input type="textarea" v-model="formData.remark"></el-input>
                    </el-form-item>
                    <el-form-item class="text_right">
                        <el-button @click="dialogue.show = false">取消</el-button>
                        <el-button type="primary" @click="onSubmit('form')">提交</el-button>
                    </el-form-item>
                </el-form>
            </div>
        </el-dialog>
    </div>
</template>

<script>
    export default {
        name: "dialogue",
        data() {
            return {
                formData: {
                    type: "",
                    description: "",
                    income: "",
                    expense: "",
                    cash: "",
                    remark: "",
                    id:""
                },
                format_type_list: [
                    "提现","手续费","管理费","充值","转账","优惠劵"
                ],
                form_rules: {
                    description: [
                        {
                            required: true, message: "收支描述不能为空", trigger: "blur"
                        }
                    ],
                    income: [
                        {
                            required: true, message: "收入不能为空！", trigger: "blur"
                        }
                    ],
                    expense: [
                        {
                            required: true, message: "支出不能为空！", trigger: "blur"
                        }
                    ],
                    cash: [
                        {
                            required: true, message: "账户现金不能为空！", trigger: "blur"
                        }
                    ]
                }
            }
        },
        props: {
            dialogue: Object
        },
        methods: {
            onSubmit(form) {
                this.$refs[form].validate(valid => {
                    if (valid) {
                        //console.log(this.formData);
                        this.$axios.post('/api/profiles/add', this.formData)
                            .then(res => {
                                this.$message({
                                    message: "添加成功",
                                    type: "successs"
                                });
                                // 关闭对话框
                                this.dialogue.show = false;
                               // 刷新表格信息
                                this.$emit('update');
                            });
                    }
                })
            }
        }
    };
</script>
```

编辑`FundList.vue`，当通过Dialog对话框新增记录后，自动刷新表格信息：
```html
<Dialog :dialogue="dialogue" @update="getProfile"></Dialog>
```








