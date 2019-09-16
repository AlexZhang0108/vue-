<template>
  <el-container>
    <el-main id="getMain" class="xw-yz-main">
      <el-row>
        <el-col>
          <el-breadcrumb>
            <el-breadcrumb-item :to="{path:(yzEnable?'/yz/gxQuotedPriceSchoolTabs/gxQuoteSchoolCtr':'/gxQuotedPriceSchoolTabs/gxQuoteSchoolCtr')}">报价单</el-breadcrumb-item>
            <el-breadcrumb-item>审核</el-breadcrumb-item>
          </el-breadcrumb>
        </el-col>
      </el-row>
      <div class="slm-content-box" style="padding:16px 28px;">
        <div class="zfd-content-headline">
          <div>
            <span class="slm-content-title">报价单详情</span>
          </div>
        </div>
        <el-row class="zfd-form" style="margin-top: 16px;">
          <el-form :inline="true" class="demo-form-inline" :model="formInline" ref="formInline" :rules="rules">
            <el-row>
              <el-col :span="8">
                <el-form-item label="调价单号:">
                  <span>{{formInline.adjustPriceCode}}</span>
                </el-form-item>
              </el-col>
              <el-col :span="8">
                <el-form-item label="供应商:">
                  <span>{{formInline.providerName}}</span>
                </el-form-item>
              </el-col>
              <el-col :span="8">
                <el-form-item label="价格生效日期:">
                  <span v-if="formInline.effectiveImmediatelyFlag==1">--</span>
                  <span v-else>{{timeFormat(formInline.effectiveTime,1)}}</span>
                </el-form-item>
              </el-col>
            </el-row>
            <el-row>
              <el-col :span="8">
                <el-form-item label="提交时间:">
                  <span>{{timeFormat(formInline.submitTime,2)}}</span>
                </el-form-item>
              </el-col>
            </el-row>
            <el-row class="xw-yz-table" style="padding:0">
              <el-table border :data="formInline.matList" style="margin-top:20px;" max-height="785">
                <el-table-column align="center" label="物料分类" resizable :formatter="getMaterialType"></el-table-column>
                <el-table-column align="center" label="物料名称" resizable prop="materialName"></el-table-column>
                <el-table-column align="center" label="规格" resizable prop="materialSpec"></el-table-column>
                <el-table-column align="center" label="库存单位" resizable prop="stockUnit"></el-table-column>
                <el-table-column align="center" label="供货单位" resizable prop="adjustSupplyUnit" :formatter="supplyStr"></el-table-column>
                <el-table-column align="center" label="原供货单价（元）" resizable prop="originalSupplyPrice" :formatter="getPriceFM"></el-table-column>
                <el-table-column align="center" label="审核供货单价（元）" resizable prop="adjustSupplyPrice" :formatter="getPriceFM"></el-table-column>
                <el-table-column align="center" label="操作" resizable prop="verifyStatus" fixed="right" width="380">
                  <template slot-scope="scope" prop="rejectReason">
                    <div class="zfd-Examine-mian">
                      <el-form-item :prop="'matList.' + scope.$index  " :rules="rules.rejectReason">
                        <el-button size="mini" :class="scope.row.verifyStatus==1? 'zfd-Examine-btn-green' : ''" @click="verifyStatusBtn(scope.row,1)">通过</el-button>
                        <el-button size="mini" :class="scope.row.verifyStatus==2? 'zfd-Examine-btn-red' : ''" @click="verifyStatusBtn(scope.row,2)">驳回</el-button>
                        <el-input v-model="scope.row.rejectReason" placeholder="请输入驳回原因" :disabled="scope.row.verifyStatus==1" size="mini" class="el-input order-number" :maxlength="30"></el-input>
                      </el-form-item>
                    </div>

                  </template>
                </el-table-column>
              </el-table>
            </el-row>
            <el-row class="xw-yz-funbtn">
              <el-button type="primary" @click="verify('formInline')">确认</el-button>
            </el-row>
          </el-form>
        </el-row>
      </div>
    </el-main>
  </el-container>
</template>
<script>
import { format } from "date-fns";
import { getDic, findVal } from "@/tools/dictionary";
import { getXrOrigin } from "@/tools/users";
export default {
  props: ["yzEnable"],
  data() {
    return {
      formInline: {
        matList: []
      },
      adjustPriceId: "",
      height: 600,
      rules: {
        rejectReason: [
          {
            validator(rule, row, callback) {
              if (row.verifyStatus == 2) {
                if (row.rejectReason === "") {
                  callback("请输入驳回原因");
                }
                callback();
              } else {
                callback();
              }
            },
            required: true,
            trigger: "change"
          }
        ]
      }
    };
  },
  created() {
    this.adjustPriceId = this.$route.query.adjustPriceId;
  },
  mounted() {
    if (this.adjustPriceId) {
      this.loadData();
    }
    this.intoHeight();
  },
  methods: {
    verify(formName) {
      this.$refs[formName].validate(valid => {
        if (valid) {
          let detailData = [];
          this.formInline.matList.forEach(item => {
            detailData.push({
              rejectReason: item.rejectReason,
              verifyStatus: item.verifyStatus,
              pmId: item.pmId,
              adjustPriceMatId: item.adjustPriceMatId
            });
          });
          let params = {
            providerId: this.formInline.providerId,
            providerName: this.formInline.providerName,
            adjustPriceId: this.formInline.adjustPriceId,
            matList: detailData
          };
          this.post("/tiancang/api/v1/WhsAdjustPrice/verify", params).then(
            res => {
              if (res.data.status == 1) {
                this.$message.success("已确认");
                this.$router.push({
                  path: this.yzEnable
                    ? "/yz/gxQuotedPriceSchoolTabs/gxQuoteSchoolCtr"
                    : "/gxQuotedPriceSchoolTabs/gxQuoteSchoolCtr"
                });
              }
            }
          );
        } else {
          return false;
        }
      });
    },
    verifyStatusBtn(row, res) {
      let _field = this.$refs["formInline"].fields;
      _field.map(i => {
        if (i.prop === "matList." + row.indexID) {
          i.resetField();
          return false;
        }
      });

      if (res == 1) {
        row.verifyStatus = 1;
        this.isShowValid = true;
      } else {
        row.verifyStatus = 2;
      }
    },
    loadData() {
      let url = "/tiancang/api/v1/WhsAdjustPrice/detail";
      this.post(url, { adjustPriceId: this.adjustPriceId }).then(res => {
        if (res.data.status == 1) {
          this.formInline = res.data.data;
          this.formInlinesubmitNameTxt =
            res.data.data.submitName + "( " + res.data.data.submitCode + " )";
          this.formInline.matList.forEach((item, i) => {
            item.indexID = i;
            item.verifyStatus = item.verifyStatus || 1;
          });
          this.$nextTick(() => {
            this.intoHeight();
          });
        }
      });
    },
    setParams() {
      let params = {
        height: this.height,
        isLoading: this.isLoading
      };
      window.parent.postMessage(params, "*");
    },
    //嵌入高度
    intoHeight() {
      this.$nextTick(() => {
        this.height = document.getElementById("getMain").clientHeight + 60;
        this.isLoading = false;
        this.setParams();
      });
    },
    statusFM(val) {
      return findVal("gxQuoteStatus", val);
    },
    getPriceFM(row, col, val) {
      return val ? _.toNumber(val / 100).toFixed(2) : "--";
    },
    timeFormat(cellValue, res) {
      if (res == 1) {
        return cellValue == 0 ? "--" : format(cellValue, "YYYY-MM-DD");
      } else {
        return cellValue == 0 ? "--" : format(cellValue, "YYYY-MM-DD HH:mm:ss");
      }
    },
    getMaterialType(row) {
      if (row.typeThreeName) {
        return (
          row.typeOneName + "/" + row.typeTwoName + "/" + row.typeThreeName
        );
      } else if (row.typeTwoName) {
        return row.typeOneName + "/" + row.typeTwoName;
      } else {
        return row.typeOneName;
      }
    },
    supplyStr(row, column, val) {
      if (row.stockUnit != row.purchaseUnit) {
        return (
          val +
          "( 1" +
          val +
          "=" +
          row.adjustExchangeRate +
          row.stockUnit +
          " )"
        );
      } else {
        if (row.adjustExchangeRate > 1) {
          return (
            val +
            "( 1" +
            val +
            "=" +
            row.adjustExchangeRate +
            row.stockUnit +
            " )"
          );
        } else {
          return val;
        }
      }
    }
  }
};
</script>
<style lang="scss" scoped>
.zfd-Amount {
  text-align: right;
  font-size: 14px;
  font-weight: 700;
  .zfd-span-text {
    margin-left: 15px;
  }
}
.zfd-Detail-title {
  display: flex;
  justify-content: space-between;
  align-items: center;
}
</style>

<style lang="scss">
.zfd-Examine-mian {
  .el-form-item {
    margin-bottom: 30px !important;
  }
  .el-form-item__content {
    display: flex;
    top: 12px;
    height: 30px;
    align-items: center;
    .el-form-item__error {
      left: 132px !important;
    }
    .el-input__inner {
      margin-left: 10px !important;
      height: 30px !important;
      width: 160px !important;
    }
  }
}
.zfd-Examine-btn-green {
  background-color: rgba(0, 204, 153, 1) !important;
  color: #ffffff !important;
}
.zfd-Examine-btn-red {
  background-color: rgba(255, 0, 0, 1) !important;
  color: #ffffff !important;
}

.zfd-content-headline {
  height: 40px;
  line-height: 40px;
  display: flex;
  justify-content: space-between;
  padding-bottom: 16px;
  border-bottom: 1px solid #ebeef5;
  .slm-content-title {
    color: #606266;
    font-size: 18px;
  }
  .headline-tip {
    margin-left: 20px;
  }
  .slm-tip {
    display: inline-block;
    padding: 2px 9px;
    font-size: 12px;
    margin-left: 10px;
    line-height: normal;
    color: #e6a13d;
    border: 1px solid #e6a13d;
    background: #fff;
    border-radius: 16px;
    &.gray {
      color: #ced1d9;
      border: 1px solid #ced1d9;
    }
    &.red {
      color: #f56c6c;
      border: 1px solid #f56c6c;
    }
    &.green {
      color: #67c239;
      border: 1px solid #67c239;
    }
  }
  .tip-view {
    color: #20a0ff;
    text-decoration: underline;
    cursor: pointer;
  }
  // .handle-btn {
  //   color: #20a0ff;
  //   font-size: 14px;
  //   margin-left: 24px;
  //   cursor: pointer;
  //   .icon-export {
  //     display: inline-block;
  //     width: 20px;
  //     height: 19px;
  //     // background: url(../../../../../assets/ico_dc.png) no-repeat;
  //     background-size: 100% 100%;
  //     vertical-align: text-bottom;
  //     margin-right: 5px;
  //   }
  //   .icon-print {
  //     display: inline-block;
  //     width: 18px;
  //     height: 18px;
  //     // background: url(../../../../../assets/ico_print.png) no-repeat;
  //     background-size: 100% 100%;
  //     vertical-align: text-bottom;
  //     margin-right: 5px;
  //   }
  // }
}
</style>

