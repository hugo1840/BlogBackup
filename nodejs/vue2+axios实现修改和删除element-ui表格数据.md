@[TOC](vue2+axios实现修改和删除element-ui表格数据)

# 共用Dialog组件
为了使添加按钮和编辑按钮能够共用Dialog组件，把`Dialog.vue`中的`formData`定义移动到父组件`FundList.vue`中，再将其传递到`Dialog.vue`中。
```html
<template>
	<div>
	<!--此处省略-->
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
                    show: false
                }
            };
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

在`Dialog.vue`中通过`props`获取从`FundList.vue`传递过来的数据：
```html
<script>
    export default {
        name: "dialogue",
        data() {
            return {
             
                format_type_list: [
                    "提现","手续费","管理费","充值","转账","优惠劵"
                ],
                form_rules: {
				    //...
                }
            }
        },
        props: {
            dialogue: Object,
            formData: Object
        },
        methods: {
            //...
        }
    };
</script>
```

# 根据按钮显示对话框
编辑`FundList.vue`，修改点击添加和编辑按钮时调用的方法，分别展示不同的对话框信息。
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
                console.log("HANDLE DELETE");
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
            }
        }
    }

</script>
```

# 根据按钮提交表格
编辑`Dialog.vue`，修改`onSubmit`提交方法，根据按钮类型区分为添加数据和编辑数据，分别调用不同的后端接口。
```html
<template>
    <div class="dialogue">
        <el-dialog :title="dialogue.title"
                   :visible.sync="dialogue.show"
                   :close-on-click-modal="false"
                   :modal-append-to-body="false">
            <!--此处省略-->
        </el-dialog>
    </div>
</template>

<script>
    export default {
        name: "dialogue",
        data() {
            return {
             
                format_type_list: [
                    "提现","手续费","管理费","充值","转账","优惠劵"
                ],
                form_rules: {
                    //...
                }
            }
        },
        props: {
            dialogue: Object,
            formData: Object
        },
        methods: {
            onSubmit(form) {
                this.$refs[form].validate(valid => {
                    if (valid) {
                        //console.log(this.formData);
                        const msg = this.dialogue.option == 'add' ? "添加成功" : "修改成功";
                        const url = this.dialogue.option == 'add' ? 'add' : `edit/${this.formData.id}`;
                        this.$axios.post(`/api/profiles/${url}`, this.formData)
                            .then(res => {
                                this.$message({
                                    message: msg,
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

# 删除表格记录
编辑`FundList.vue`，定义点击删除按钮时调用的方法：
```javascript
handleDelete(index, row) {
    //console.log("HANDLE DELETE");
    this.$axios.delete(`/api/profiles/delete/${row._id}`)
        .then(res => {
            this.$message("删除成功");
            this.getProfile();
        })
}
```
