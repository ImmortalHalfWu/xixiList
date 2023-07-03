<template>
    <div>
      <div style="margin-bottom: 5px; ">
        <el-input 
          placeholder="请输入游戏名称"
          v-model="searchInput"
          clearable>
        </el-input>
      </div>

      <el-table
            ref="multipleTable"
            :data="tableData"
            border
            tooltip-effect="dark"
            :height="tableHeight" 
            empty-text="游戏没找到？喊客服帮你"
            style="width: 100%"
            @row-click="clickRow"
            @select="select"
            @selection-change="handleSelectionChange">
            <el-table-column
                type="selection"
                width="55">
            </el-table-column>
            <el-table-column
                prop="name"
                sortable
                label="游戏">
                <!-- <template slot-scope="scope">{{ scope.row.date }}</template> -->
            </el-table-column>
            <el-table-column
                prop="size"
                label="大小(G)"
                sortable
                width="100">
            </el-table-column>
      </el-table>

      <div style="margin-top: 5px; " @click="showDrawer">
        <el-row type="flex" align="middle">


          <el-col :span="24" style="font-weight: bold;" >
            G/容量：{{ selectedSize }}
            <br/>
            <sub>{{ diskSize }}</sub>
            <!-- <el-button @click="drawer = true">购物车</el-button> -->
          </el-col>

          <el-badge :value="selectedCount" class="item" type="primary" style="margin-right: 10px;">
            <i class="el-icon-s-shop" style="font-size: xx-large; vertical-align:middle; color: #409eff;" ></i>
          </el-badge>
          <!-- <el-button @click="drawer = true" type="primary">确定</el-button> -->

        </el-row>
      </div>

      <el-drawer
        title="选择的游戏"
        :visible.sync="drawer"
        :with-header="false"
        size="80%"
        direction="ltr">
        
        <div :span="24" style="font-weight: bold; padding: 15px;">
          选择的游戏
          </div>

          <el-table
                :data="selectedData"
                border
                tooltip-effect="dark"
                :height="tableHeight" 
                empty-text="未选择游戏"
                style="width: 100%">
                <el-table-column
                    prop="name"
                    label="游戏">
                    <!-- <template slot-scope="scope">{{ scope.row.date }}</template> -->
                </el-table-column>
                <el-table-column
                    prop="size"
                    label="大小"
                    width="50">
                </el-table-column>
                <el-table-column
                  fixed="right"
                  label="操作"
                  width="50">
                  <template slot-scope="scope">
                    <el-button @click="delSelect(scope.row)" type="text" size="small">删除</el-button>
                  </template>
                </el-table-column>
          </el-table>
          <div style="background-color: #409eff;">
            <el-button type="primary" 
            style="width: 100%; height: 57px; font-size: 16px; font-weight: bold;"
            :round="false"
            @click="submit()"
          >提交</el-button>
          </div>
      </el-drawer>

    </div>
  </template>
  
  <script>
    import emailjs from '@emailjs/browser';

    export default {
      data() {
        return {
          sourceData: [],
          pageIndex: 1,
          pageSize: 20,
          tableData: [],
          selectedData: [],
          multipleSelection: [],
          selectedCount: 0,
          selectedSize: 0,
          tableHeight: 0,
          searchInput: "",
          searchTaskId: null,
          drawer: false,
          diskSize: "建议选购：256G",
          dataFilePath: ""
        }
      },

      watch: {
        "searchInput"(newVal, oldVal) {
          if (this.searchTaskId != null) {
            clearTimeout(this.searchTaskId);
          }

          this.searchTaskId = setTimeout(() => {
            this.pageIndex = 0 
            this.refreshTableData()
          }, 500);
        },
        "selectedSize"(newVal) {
          this.diskSize = newVal < 220 ? "建议选购：256G" : 
          newVal < 415 ? "建议选购：512G" : 
          newVal < 900 ? "建议选购：1T" :
          newVal < 1800 ? "建议选购：2T" :
          "2T以上欢迎定制"
        }
      },

      created () {
        document.title = '好游荐电玩'
      },
      mounted() {
        let that = this;
        this.$nextTick(() => {
            this.tableHeight = window.innerHeight - 110;  //窗口高度 - 表格距离顶部距离 - 需求高度
            this.$refs.multipleTable.doLayout(); //更新表格 防止表格内容变化表格异常情况
            // 监听窗口大小变化
            window.onresize = function () {
                that.tableHeight =
                window.innerHeight - 80;
                that.$refs.multipleTable.doLayout();
            };

            this.$refs.multipleTable.bodyWrapper.addEventListener('scroll', (res) => {
            const height = res.target
            const clientHeight = height.clientHeight
            const scrollTop = height.scrollTop
            const scrollHeight = height.scrollHeight
            
            if (clientHeight + scrollTop === scrollHeight) {
                this.refreshTableData();
            }
            }, true)

        });

        this.$confirm('', '请先选择游戏平台', {
          confirmButtonText: 'PC电脑',
          cancelButtonText: 'PS4｜PS5',
          // type: 'success',
          center: true,
          closeOnClickModal: false,
          closeOnPressEscape: false
        }).then(() => {
          this.dataFilePath = "pc-data-base64.txt"
          this.loadGameData()
        }).catch(() => {
          this.dataFilePath = "ps-data-base64.txt"
          this.loadGameData()
        });

      },
  
      methods: {
        async handleSelect(key, keyPath) {
          console.log(key, keyPath);
        },
        async filterGame(newVal) {
            this.searchTaskId = null
            this.tableData = this.sourceData.filter((value) => value.name.includes(newVal))

            this.$nextTick(() => {
              this.tableData.forEach((value) => {
                  if (value.selected) {
                    this.$refs.multipleTable.toggleRowSelection(value)
                  }
                })
            });
        },
        async select(selecteds, row) {
          row.selected = selecteds.some((value) => {
            return value.name == row.name
          })
        },
        async cancelSelection() {
          this.$refs.multipleTable.clearSelection();
        },
        async handleSelectionChange(val) {
          this.selectedSize = 0
          this.selectedCount = 0
          this.multipleSelection = val;
          this.sourceData.forEach((value) => {
            if (value.selected) {
              this.selectedSize += value.size
              this.selectedCount += 1
            }
          })
        },
        async clickRow(row) {
             // 根据id判断当前点击的是否被选中
             const selected = this.multipleSelection.some(item => item.name === row.name)
             row.selected = !selected
             if (!selected) {
                // 选择
                this.multipleSelection.push(row)
                this.$refs.multipleTable.toggleRowSelection(row, true);
             } else {
                // 取消
                var finalArr = this.multipleSelection.filter((item) => {
                  return item.name !== row.name
                })
                // 取消后剩余选中的
                this.multipleSelection = finalArr  
                this.$refs.multipleTable.toggleRowSelection(row, false);
             }             
        },
        async submit() {

          if (this.selectedCount == 0) {
            this.$message({
              message: '请选择游戏',
              center: true,
              type: 'warning'
            });
            return
          }

          this.$confirm(`${this.diskSize}以上硬盘，确定提交?`, '提示', {
            confirmButtonText: '确定',
            cancelButtonText: '取消',
            center: true
          }).then(() => {

            const loading = this.$loading({
              lock: true,
              text: 'Loading',
              spinner: 'el-icon-loading',
              background: 'rgba(0, 0, 0, 0.7)'
            });
            var data = this.selectedData.map((value) => `${value.name}\t${value.size}`).join("\n")
            var code_fill_str = ["000000", "00000", "0000", "000", "00", "0", ""];
            var code = '' + parseInt(Math.random()*1000000);
            code = code_fill_str[code.length] + code; 
            // console.log(`订单号：${code}\n数量：${this.selectedCount}\n总大小：${this.selectedSize}\n${this.diskSize}\n${data}`);
            this.sendEmail(
              code, 
              `${this.dataFilePath}\n单号：${code}\n数量：${this.selectedCount}\n总大小：${this.selectedSize}\n${this.diskSize}\n${data}`, 
              loading);
          }).catch(() => {
          });
        },
        async sendEmail(code, data, loading) {
            emailjs.send("service_8mk21ke","template_qiluuhl",{
                // to_name: "yuqustudio@163.com",
                // from_name: "yuqustudio@163.com",
                code: code,
                message: data,
                }, 'kV3FqN4eMcfBzrTfV')
            .then((result) => {
              loading.close()
              
              this.$prompt('请复制以下单号，在下单硬盘时填写在备注中，或发送给客服', '提交成功', {
                confirmButtonText: '确定',
                inputValue: code,
                center: true,
                closeOnClickModal: false,
                closeOnPressEscape: false
              }).then(({ value }) => {
                location.reload();
              }).catch(() => {   
                location.reload(); 
              });

            }, (error) => {
              loading.close()
              this.$notify.error({
                title: '错误',
                message: '提交失败，请将加购的游戏截图发给客服'
              });
            });
        },
        async loadGameData() {
            this.axios.get(this.dataFilePath).then(res => {
                  //请求成功，触发then中的函数
                  // console.log(res)  
                  const data = res.data
                  const lineData = decodeURIComponent(atob(data)).split('\n')
                  lineData.forEach((value => {
                      var gameInfo = value.split('\,')
                      this.sourceData.push({
                          name: gameInfo[0].replaceAll("- ", ""),
                          size: parseInt(gameInfo[1]),
                          selected: false
                      })
                  }))

                  this.pageIndex = 0;
                  this.refreshTableData();
                })
        },
        async refreshTableData() {
          if (this.pageIndex == 0) {
            this.tableData = []
            this.selectedSize = 0
          }

          var tempSourceData = []

          if (this.searchInput != null && this.searchInput.length > 0) {
            tempSourceData = this.sourceData.filter((value) => value.name.includes(this.searchInput))
          } else {
            tempSourceData = this.sourceData
          }

          var startIndex = this.pageIndex * this.pageSize
          if (startIndex >= tempSourceData.length) { return }
          var endIndex = this.pageIndex * this.pageSize + this.pageSize
          endIndex = tempSourceData.length <= tempSourceData ? tempSourceData.length : endIndex
          var newData = tempSourceData.slice(startIndex, endIndex)
          this.tableData.push(...newData)
          ++this.pageIndex

          this.$nextTick(() => {
            newData.forEach((value) => {
                // console.log(value.name + " " + value.selected);
                if (value.selected) {
                  this.$refs.multipleTable.toggleRowSelection(value)
                }
              })
          });
        },
        async showDrawer() {
          this.selectedData = this.sourceData.filter((value) => value.selected)
          this.drawer = true
        },
        async delSelect(row) {
          row.selected = false
          this.selectedData = this.selectedData.filter((value) => value !== row)
          this.selectedCount = this.selectedData.length
          this.selectedSize -= row.size
          this.$refs.multipleTable.toggleRowSelection(row, false)
          if (this.selectedData.length == 0) {
            this.drawer = false
          }
        }
      }
    }
  </script>

  <style type="text/css" scoped>
::v-deep .el-table__header-wrapper .el-checkbox {
  visibility: hidden;
}
::v-deep .el-message-box {
  width: 60%;
}
@media screen and (max-width: 720px) {
  .el-message-box {
    width: 60% !important;
  }
}
</style>