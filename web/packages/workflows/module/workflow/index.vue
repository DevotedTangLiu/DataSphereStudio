<template>
  <div class="page-bgc">
    <div style="width: 100%; height: 100%">
      <template v-if="!topTapList || topTapList.length === 0">
        <Void
          :addProject="createProjectHandler"
          :addWrokFlow="ProjectMergeAdd"
          :goto="gotoLastWorkflow"
        />
      </template>
    </div>
    <Modal
      v-model="deleteProjectShow"
      :title="$t('message.orchestratorModes.deleteOrchestrator')"
      @on-ok="deleteProjectConfirm"
    >{{
      `${$t("message.orchestratorModes.confirmDeleteOrchestrator")}${
        deleteProjectItem.orchestratorName
      } ？`
    }}</Modal
    >
    <!--修改编排-->
    <WorkflowForm
      ref="workflowForm"
      :workflow-data="currentOrchetratorData"
      :classify-list="cacheData"
      :add-project-show="ProjectShow"
      :action-type="actionType"
      :projectData="projectData"
      :orchestratorModeList="orchestratorModeList"
      :selectOrchestratorList="selectOrchestratorList"
      @show="ProjectShowAction"
      @confirm="ProjectConfirm"
    ></WorkflowForm>
    <VersionDetail
      :version-detail-show="versionDetailShow"
      :version-data="versionData"
      @rollback="rollback"
      @versionDetailShow="versionDetailAction"
      @goto="versionGotoWorkflow"
    ></VersionDetail>
    <!--新增和导入 二合一弹窗-->
    <Modal v-model="mergeModalShow" :footer-hide="true" title="添加工作流">
      <WorkflowFormNew
        v-if="mergeModalShow"
        :workflow-data="currentOrchetratorData"
        :orchestratorModeList="orchestratorModeList"
        :selectOrchestratorList="selectOrchestratorList"
        :projectNameList="formatProjectNameList"
        @cancel="ProjectMergeCancel"
        @confirm="ProjectMergeConfirm"
      >
      </WorkflowFormNew>
      <!-- <Tab-pane label="导入编排" name="upload">
          <Upload
            ref="uploadJson"
            type="drag"
            :data="uploadData"
            :before-upload="handleUpload"
            :on-success="handleSuccess"
            :on-error="handleError"
            :format="['zip']"
            :max-size="2001000"
            :action="uploadUrl">
            <div class="upload-box">
              <Icon
                type="ios-cloud-upload"
                size="52"
                style="color: #3399ff"></Icon>
              <p>{{ $t('message.orchestratorModes.clickOrDragFile') }}</p>
            </div>
          </Upload>
        </Tab-pane> -->
    </Modal>
    <Spin v-if="loading" size="large" fix />
  </div>
</template>
<script>
import WorkflowForm from "./module/workflowForm.vue";
import WorkflowFormNew from "./module/workflowFormNew.vue";
import VersionDetail from "./module/versionDetail.vue";
import Void from "../common/voidPage/void.vue";
import storage from '@dataspherestudio/shared/common/helper/storage';
import api from '@dataspherestudio/shared/common/service/api';
import { GetAreaMap } from '@dataspherestudio/shared/common/service/apiCommonMethod.js';
import axios from "axios";
export default {
  props: {
    topTapList: {
      type: Array,
      default: () => []
    },
    projectData: {
      type: Object,
      default: () => {}
    },
    orchestratorModeList: {
      type: Object,
      default: () => {}
    },
    currentMode: {
      type: String,
      default: null
    },
    selectOrchestratorList: {
      type: Array,
      default: () => []
    },
    refreshFlow: {
      type: Boolean,
      default: false
    },
    createProjectHandler: {
      type: Function,
      default: () => {}
    },
    projectsTree: {
      type: Array,
      default: () => []
    }
  },
  components: {
    // projectContentItem: commonModule.component.workflowContentItem,
    WorkflowForm,
    WorkflowFormNew,
    VersionDetail,
    Void
  },
  data() {
    return {
      mergeModalShow: false, // 二合一弹窗展示
      ProjectShow: false, // 添加工作流展示
      versionDetailShow: false,
      deleteProjectShow: false, // 删除工作流弹窗展示
      deleteProjectItem: "", // 删除的工作流项
      isRootFlow: false,
      actionType: "", // add || modify
      loading: false,
      cacheData: [], // 工作流初始树结构，用于搜索过滤
      params: "",
      versionData: [], // 工作流版本信息
      currentOrchetratorData: null,
      applicationAreaMap: [],
      importModal: false,
      uploadUrl: ``,
      uploadData: null,
    };
  },
  computed: {
    checkEditable() {
      if (this.projectData.editUsers && this.projectData.editUsers.length > 0) {
        return this.projectData.editUsers.some(e => e === this.getUserName());
      } else {
        return false;
      }
    },
    // 整理项目列表
    formatProjectNameList() {
      let res = [];
      if( this.projectsTree.length > 0 ) {
        this.projectsTree.forEach(item => {
          let name = item.name;
          let id = item.id + '';
          res.push({name, id})
        })
      }
      return res;
    },
  },
  watch: {
    isRootFlow(val) {
      this.getParams(val);
      this.getFlowData(this.params);
    },
    $route() {
      this.fetchFlowData();
    },
    currentMode(val) {
      if (val) {
        this.fetchFlowData();
      }
    },
    refreshFlow(val) {
      if (val) {
        // 收到通知刷新flow
        this.fetchFlowData();
      }
    }
  },
  created() {
    this.fetchFlowData();
    GetAreaMap().then(res => {
      this.applicationAreaMap = res.applicationAreas;
    });
  },
  mounted() {

  },
  methods: {
    rollback(row) {
      api
        .fetch(
          `${this.$API_PATH.ORCHESTRATOR_PATH}rollbackOrchestrator`,
          {
            orchestratorId: row.orchestratorId,
            version: row.version,
            projectName: this.$route.query.projectName,
            projectId: row.projectId,
            labels: {
              route: this.getCurrentDsslabels()
            }
          },
          "post"
        )
        .then(() => {
          this.$Message.success('操作成功');
          this.$emit('rollback', row.orchestratorId)
          this.versionDetail()
        })
        .catch(() => {
        });
    },
    fetchFlowData() {
      this.getParams();
      this.getFlowData(this.params);
    },
    // 获取工作流的参数
    getParams() {
      this.params = {
        workspaceId: this.$route.query.workspaceId,
        projectId: this.$route.query.projectID,
        orchestratorMode: this.currentMode
      };
    },
    // 获取所有分类和工作流
    getFlowData(params = {}) {
      if (!params.projectId) return;
      this.loading = true;
      api
        .fetch(
          `${this.$API_PATH.ORCHESTRATOR_PATH}getAllOrchestrator`,
          params,
          "post"
        )
        .then(res => {
          this.dataList[0].dwsFlowList = res.page;
          this.cacheData = this.dataList;
          this.loading = false;
        })
        .catch(() => {
          this.loading = false;
        });
    },
    ProjectMergeAdd() {
      if(!this.projectsTree.length) {
        this.$Message.warning(this.$t("message.orchestratorModes.noProjectWarning"));
      } else {
        this.mergeModalShow = true;
        this.init();
        this.initUpload();
      }
    },
    ProjectMergeCancel() {
      this.mergeModalShow = false;
    },
    ProjectMergeConfirm({data, cb}) {
      data.dssLabels = [this.getCurrentDsslabels()];
      data.labels = { route: this.getCurrentDsslabels() };
      api
        .fetch(
          `${this.$API_PATH.ORCHESTRATOR_PATH}createOrchestrator`,
          data,
          "post"
        )
        .then(() => {
          this.$Message.success(this.$t("message.workflow.createdSuccess"));
          this.ProjectMergeCancel();
          // 更新左侧tree，同时父组件会通知刷新flow
          this.noticeParent(data.projectId);
          if (cb) {
            cb()
          }
          this.init()
        });
    },
    noticeParent(projectId) {
      // 更新左侧tree，同时父组件会通知刷新flow
      this.$emit("on-tree-modal-confirm", {
        id: projectId
      });
    },
    // 新增工作流
    addProject() {
      this.ProjectShow = true;
      this.actionType = "add";
      this.init();
    },
    // 确认新增工作流
    ProjectConfirm(orchestratorData) {
      orchestratorData.labels = { route: this.getCurrentDsslabels() };
      this.loading = true;
      if (this.actionType === "add") {
        api
          .fetch(
            `${this.$API_PATH.ORCHESTRATOR_PATH}createOrchestrator`,
            orchestratorData,
            "post"
          )
          .then(() => {
            this.$Message.success(this.$t("message.workflow.createdSuccess"));
            this.getParams();
            this.getFlowData(this.params);
          })
          .catch(() => {
            this.loading = false;
          });
      } else {
        const {
          workspaceId,
          projectId,
          orchestratorName,
          // orchestratorId,
          orchestratorMode,
          orchestratorWays,
          id,
          description,
          uses,
          labels
        } = orchestratorData;
        api
          .fetch(
            `${this.$API_PATH.ORCHESTRATOR_PATH}modifyOrchestrator`,
            {
              workspaceId,
              projectId,
              orchestratorName,
              // orchestratorId,
              orchestratorMode,
              orchestratorWays,
              id,
              description,
              uses,
              labels
            },
            "post"
          )
          .then(() => {
            this.loading = false;
            this.$Message.success(
              this.$t("message.workflow.eitorSuccess", {
                name: orchestratorData.orchestratorName
              })
            );
            this.getParams();
            this.getFlowData(this.params);
            setTimeout(() => {
              this.$router.go(0);
            }, 1500);
          })
          .catch(() => {
            this.loading = false;
            this.currentOrchetratorData.uses = this.$refs.workflowForm.originBusiness;
          });
      }
    },
    // 删除单项工作流
    deleteWorkflow(params) {
      if (params) {
        this.deleteProjectShow = true;
        this.deleteProjectItem = {...params};
      } else {
        console.error('parmas emtpty', params)
      }
    },
    // 确认删除单项工作流
    deleteProjectConfirm() {
      // 调用删除接口
      this.loading = true;
      const params = {
        id: this.deleteProjectItem.id,
        projectId: this.deleteProjectItem.projectId,
        workspaceId: this.deleteProjectItem.workspaceId,
        labels: { route: this.getCurrentDsslabels() }
      };
      api
        .fetch(
          `${this.$API_PATH.ORCHESTRATOR_PATH}deleteOrchestrator`,
          params,
          "post"
        )
        .then(() => {
          this.loading = false;
          this.$emit('delete-workflow', this.deleteProjectItem)
          this.$Message.success(
            this.$t("message.workflow.deleteSuccessName", {
              name: this.deleteProjectItem.name
            })
          );
        })
        .catch(() => {
          this.loading = false;
        });
    },
    init() {
      this.currentOrchetratorData = {
        orchestratorName: "",
        description: "",
        uses: "",
        orchestratorMode: "",
        orchestratorWays: null,
        projectId: this.$route.query.projectID,
        workspaceId: this.$route.query.workspaceId
      };
    },
    // 修改编排
    projectModify(classifyId, project) {
      this.init();
      this.ProjectShow = true;
      this.actionType = "modify";
      this.currentOrchetratorData = project;
      this.currentOrchetratorData.taxonomyID = classifyId;
    },
    ProjectShowAction(val) {
      this.ProjectShow = val;
    },
    // 最近打开点击入口
    gotoLastWorkflow(workflow) {
      let that = this;
      const params = {
        workspaceId: this.$route.query.workspaceId,
        projectId:
          (workflow.query && workflow.query.projectID) ||
          this.$route.query.projectID,
        orchestratorMode: this.currentMode
      };
      if (!params.projectId) return;
      this.loading = true;
      api
        .fetch(
          `${this.$API_PATH.ORCHESTRATOR_PATH}getAllOrchestrator`,
          params,
          "post"
        )
        .then(res => {
          let dwsFlowList = res.page;
          const { id } = workflow;
          let subItem = null;
          dwsFlowList.forEach( item => {
            if( item.orchestratorId == id ) {
              subItem = item
            }
          })
          // 这里的一定是最新版本
          that.$emit("open-workflow", {
            workspaceId: params.workspaceId,
            projectID: params.projectID,
            projectName: workflow.query && workflow.query.projectName,
            id: subItem.orchestratorId,
            name: subItem.orchestratorName,
            version: String(subItem.orchestratorVersionId),
            orchestratorMode: subItem.orchestratorMode,
            releasable: subItem.releasable, // 可发布权限字段
            editable: subItem.editable,
            priv: subItem.priv, // 权限字段
            lastedNode: true,
          });
          that.loading = false;
        })
        .catch(() => {
          this.loading = false;
        });
    },
    // 查看版本详情
    versionDetail(classifyId, project = this.currentOrchetratorData) {
      this.versionDetailShow = true;
      this.currentOrchetratorData = project;
      const prams = {
        orchestratorId: project.orchestratorId
      };
      api
        .fetch(
          `${this.$API_PATH.ORCHESTRATOR_PATH}getVersionByOrchestratorId`,
          prams,
          "post"
        )
        .then(res => {
          // 新加tab功能需要工作流名称
          this.versionData = res.list || [];
        });
    },
    versionDetailAction(val) {
      this.versionDetailShow = val;
    },
    // 点击版本的打开跳转工作流编辑
    versionGotoWorkflow(row) {
      this.$emit("open-workflow", {
        ...this.$route.query,
        id: row.id,
        orchestratorId: row.orchestratorId,
        version: String(row.id), // 在编排列表返回的是版本id
        name:
          row._index === 0
            ? `${this.currentOrchetratorData.orchestratorName}`
            : `${this.currentOrchetratorData.orchestratorName}(${this.$t(
              "message.workflow.historicalVersion"
            )})`,
        priv: this.currentOrchetratorData.priv,
        orchestratorMode: this.currentOrchetratorData.orchestratorMode,
        readonly: row._index === 0 ? "false" : "true",
        appId: row.appId,
        orchestratorVersionId: row.id,
        releasable: this.currentOrchetratorData.releasable,
        editable: this.currentOrchetratorData.editable
      });
    },
    goBack() {
      this.$router.go(-1);
    },
    charCompare(charA, charB) {
      // 谁为非法值谁在前面
      if (
        charA === undefined ||
        charA === null ||
        charA === "" ||
        charA === " " ||
        charA === "　"
      ) {
        return -1;
      }
      if (
        charB === undefined ||
        charB === null ||
        charB === "" ||
        charB === " " ||
        charB === "　"
      ) {
        return 1;
      }
      return charA.localeCompare(charB);
    },
    // 退出到合作项目页
    gotoBack() {
      let workspaceId = this.$route.query.workspaceId;
      this.$router.push({
        path: `/workspaceHome?workspaceId=${workspaceId}`
      });
    },
    // 编排的导出
    orchestraorExport(orchestrator) {
      const params = {
        workspaceId: +this.$route.query.workspaceId,
        projectId: +this.$route.query.projectID,
        projectName: this.$route.query.projectName,
        orchestratorId: orchestrator.orchestratorId,
        orcVersionId: orchestrator.orchestratorVersionId,
        addOrcVersion: false,
        dssLabels: this.getCurrentDsslabels(),
        labels: {
          route: this.getCurrentDsslabels()
        }
      };
      // 返回结构不一样
      axios
        .get(
          `${location.protocol}//${window.location.host}/api/rest_j/v1/dss/framework/orchestrator/exportOrchestrator`,
          {
            params,
            responseType: "arraybuffer"
          }
        )
        .then(res => {
          const blob = new Blob([res.data], { type: "application/zip" });
          const url = URL.createObjectURL(blob);
          const a = document.createElement("a");
          a.style.display = "none";
          a.href = url;
          a.setAttribute(
            "download",
            `${this.$route.query.projectName}_${orchestrator.orchestratorName}`
          );
          const evObj = document.createEvent("MouseEvents");
          evObj.initMouseEvent(
            "click",
            true,
            true,
            window,
            0,
            0,
            0,
            0,
            0,
            false,
            false,
            true,
            false,
            0,
            null
          );
          a.dispatchEvent(evObj);
          // const flag = a.dispatchEvent(evObj);
          // this.$nextTick(() => {
          //   if (flag) {
          //     this.$Message.success(this.$t('message.orchestratorModes.exportSuccess'));
          //   }
          // });
        })
        .catch(() => {
          this.$Message.error(this.$t("message.orchestratorModes.exportFail"));
        });
    },
    initUpload() {
      this.uploadUrl = `/api/rest_j/v1/dss/framework/orchestrator/importOrchestratorFile?labels=dev`;
      this.uploadData = {
        projectName: this.$route.query.projectName,
        projectID: +this.$route.query.projectID,
        dssLabels: "dev"
      };
      // this.$refs.uploadJson.clearFiles();
    },

    orchestratorImport() {
      this.uploadUrl = `/api/rest_j/v1/dss/framework/orchestrator/importOrchestratorFile`;
      this.uploadData = {
        projectName: this.$route.query.projectName,
        projectID: +this.$route.query.projectID,
        labels: JSON.stringify({
          route: "dev"
        })
      };
      this.importModal = true;
    },
    // 手动上传
    handleUpload(file) {
      if (file.name.indexOf(".zip") === -1) {
        this.$Message.warning(this.$t("message.orchestratorModes.selectZip"));
        return false;
      }
    },
    handleSuccess(response, file) {
      if (response.status === 0) {
        this.$Message.success(
          `${file.name} ${this.$t("message.orchestratorModes.importSuccess")}`
        );
        // 更新左侧tree，同时父组件会通知刷新flow
        this.noticeParent();
      }
      this.ProjectMergeCancel();
    },
    handleError() {
      this.$Message.error(this.$t("message.orchestratorModes.importFail"));
      this.ProjectMergeCancel();
    },
    // 获取当前编排的环境
    getCurrentDsslabels() {
      return (
        this.$route.query.label ||
        (storage.get("currentDssLabels")
          ? storage.get("currentDssLabels")
          : null)
      );
    }
  }
};
</script>
<style lang="scss" scoped src="./index.scss"></style>
