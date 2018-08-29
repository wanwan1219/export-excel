<template>
    <div class="page-wrap">
        <filtrate :type="false"></filtrate>
        <div class="total-num">共有
            <span class="num">{{consumeData.totalNum}}</span>条记录</div>
        <div class="page-content">
            <el-table :data="consumeData.items"
                      border
                      v-loading="loading"
                      style="width: 100%">
                <el-table-column prop="studentId"
                                 align="center"
                                 label="学员编号"
                                 show-overflow-tooltip
                                 min-width="156">
                </el-table-column>
                <el-table-column prop="name"
                                 align="center"
                                 label="姓名"
                                 show-overflow-tooltip
                                 min-width="90">
                </el-table-column>

                <el-table-column label="导出日期"
                                 min-width="205"
                                 align="center"
                                 header-align="center">
                    <template slot-scope="scope">
                        <el-button type="text"
                                   v-for="item , i in nearThreeMounth"
                                   @click="exportExcel(item.value , scope.row.studentId, scope.row.name)"
                                   :key="i">{{item.name}}</el-button>
                    </template>
                </el-table-column>
            </el-table>
        </div>
        <div class="pagination-wrap"
             v-if="consumeData.totalNum >10 ">
            <el-pagination background
                           layout="prev, pager, next"
                           :current-page="consumeData.currentPage"
                           @current-change="changePage"
                           :total="consumeData.totalNum">
            </el-pagination>
        </div>
    </div>
</template>

<script>
import filtrate from "./components/filtrate";
import pagination from "@/mixin/pagination.js";
export default {
    data() {
        return {
            consumeData: "",
            loading: false,
            nearThreeMounth: [],
            serverTimestamp: "",
            firstIn: true
        };
    },
    metaInfo: {
        title: "课耗导出(学员)"
    },
    components: {
        filtrate
    },
    mixins: [pagination],
    mounted() {
        this.init();
        this.getMonthList();
    },
    watch: {
        $route() {
            this.init();
        }
    },
    methods: {
        init() {
            let query = this.$route.query;
            this.loading = true;
            this.$Http
                .get(`/service/student/consume/index`, { params: query })
                .then(res => {
                    if (res.status == 200) {
                        this.consumeData = res.content;
                        this.loading = false;
                        this.serverTimestamp = res.serverTimestamp;
                        if (this.firstIn && this.serverTimestamp) {
                            this.firstIn = false;
                            this.getMonthList();
                        }
                    } else {
                        this.$message.error(res.message);
                        this.loading = false;
                    }
                });
        },
        exportExcel(date, studentId,studentName) {
            this.exportLists({
                type:'GET',
                url:`/api/service/student/consume/export-excel?studentId=${studentId}&dateMonth=${date}`,
                name:'test.xlsx',
                data:{},
                studentName,
                date
            })
        },
        exportLists(model) {
            // 二进制流下载 axios由于封装 会把数据转换成string 所以采用原生xhr
            var xhr=null;
            if (window.XMLHttpRequest) {//Mozilla 浏览器
                xhr = new XMLHttpRequest();
            }else {
                if (window.ActiveXObject) {//IE 浏览器
                    try {
                        xhr = new ActiveXObject("Microsoft.XMLHTTP");
                    }
                    catch (e) {
                        try {//IE 浏览器
                            xhr = new ActiveXObject("Msxml2.XMLHTTP");
                        }
                        catch (e) {
                        }
                    }
                }
            }
            xhr.open(model.type, model.url, true);
            xhr.responseType = "blob";
            xhr.setRequestHeader("X-Authorization", this.$store.getters.token);
            if(model.type=='post'){
                xhr.setRequestHeader("Content-type","application/json");
            }
            let self = this;
            xhr.onload = function() {
                if (this.status == 200) {
                	try {
                		var fileName = this.getResponseHeader("content-disposition");
                        var res = this.response;
                        if(!!window.ActiveXObject || "ActiveXObject" in window) {
                            fileName = `${model.studentName}-${model.date}.xlsx`;
                        }else {
                            fileName = `${model.studentName}-${model.date}`;
                        }
	                    var blob = new Blob([res], {
	                            type: "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet;charset=UTF-8"
                            });
	                    downFile(blob, fileName);
	                    function downFile(blob, fileName) {
	                        if (window.navigator.msSaveOrOpenBlob) {
	                            navigator.msSaveBlob(blob, fileName);
	                        } else {
	                            var link = document.createElement("a");
	                            link.href = window.URL.createObjectURL(blob);
	                            link.download = fileName;
	                            link.click();
	                            window.URL.revokeObjectURL(link.href);
	                        }
	                    }
                	}
                    catch (e) {
                        if(navigator.appName == "Microsoft Internet Explorer" && navigator.appVersion.match(/9./i)=="9.") {
                            self.$message.error('IE9暂不支持此功能，请升级或更换浏览器后重试！')
                        }else {
                            self.$message.error('导出错误')
                        }
                    }
                }else{
                    self.$message.error('导出错误')
                }
            }
            xhr.send(model.data);
        },
        getMonthList() {
            if (!this.serverTimestamp) {
                return;
            }
            let date = new Date(Number(this.serverTimestamp));
            // let date = new Date();
            let nowYear = date.getFullYear();
            let nowMounth = date.getMonth();
            let monthOfYear = [
                "一月",
                "二月",
                "三月",
                "四月",
                "五月",
                "六月",
                "七月",
                "八月",
                "九月",
                "十月",
                "十一月",
                "十二月"
            ];
            let nearThreeMounth = [];
            let listValue = [];
            for (let i = 0; i < 3; i++) {
                date.setMonth(nowMounth - i);
                if (date.getMonth() + 1 < 10) {
                    listValue.unshift(
                        date.getFullYear() + "0" + (date.getMonth() + 1)
                    );
                } else {
                    listValue.unshift(
                        date.getFullYear() + "" + (date.getMonth() + 1)
                    );
                }
                nearThreeMounth.unshift(monthOfYear[date.getMonth()]);
            }
            for (let i = 0; i < 3; i++) {
                let mounthObj = {};
                mounthObj["name"] = nearThreeMounth[i];
                mounthObj["value"] = listValue[i];
                this.nearThreeMounth.push(mounthObj);
            }
        }
    }
};
</script>

<style lang="scss" scoped>
</style>