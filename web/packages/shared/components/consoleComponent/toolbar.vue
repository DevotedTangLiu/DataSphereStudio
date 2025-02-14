<template>
  <div
    class="we-toolbar-wrap">
    <ul
      ref="toolbar"
      class="we-toolbar">
      <li @click="changeViewType('table')" :class="{'we-toolbar-active': activeTool === 'table'}">
        <Icon type="ios-grid" :size="iconSize"/>
        <span v-if="isIconLabelShow" :title="$t('message.common.toolbar.analysis')" class="v-toolbar-icon">{{$t('message.common.toolbar.resultGroup')}}</span>
      </li>
      <template v-for="(comp, index) in extComponents">
        <component
          :key="comp.name + index"
          :is="comp.component"
          :script="script"
          :class="{'we-toolbar-active': activeTool === comp.name}"
          @event-from-ext="eventFromExt"
        />
      </template>
      <li v-if="toolbarShow.download" :style="{cursor: rsDownload ? 'pointer': 'not-allowed'}">
        <Poptip
          :transfer="true"
          :width="200"
          v-model="popup.download"
          placement="right"
          popper-class="we-poptip">
          <div @click.stop="openPopup('download')">
            <SvgIcon :style="{ 'font-size': '20px' }" icon-class="downLoad" color="#515a6e" />
            <span v-if="isIconLabelShow" :title="$t('message.common.download')" class="v-toolbar-icon">{{ $t('message.common.download') }}</span>
          </div>
          <div slot="content">
            <div>
              <Row>
                {{ $t('message.common.toolbar.format') }}
              </Row>
              <Row>
                <RadioGroup v-model="download.format">
                  <Col span="10">
                    <Radio label="1">CSV</Radio>
                  </Col>
                  <Col
                    v-if="resultType === '2'"
                    span="10"
                    offset="4">
                    <Radio label="2">Excel</Radio>
                  </Col>
                </RadioGroup>
              </Row>
            </div>
            <div>
              <Row>
                {{ $t('message.common.toolbar.coding') }}
              </Row>
              <Row>
                <RadioGroup v-model="download.coding">
                  <Col span="10">
                    <Radio label="1">UTF-8</Radio>
                  </Col>
                  <Col
                    span="10"
                    offset="4">
                    <Radio label="2">GBK</Radio>
                  </Col>
                </RadioGroup>
              </Row>
            </div>
            <div>
              <Row>
                {{$t('message.common.toolbar.replace')}}
              </Row>
              <Row>
                <RadioGroup v-model="download.nullValue">
                  <Col span="10">
                    <Radio label="1">NULL</Radio>
                  </Col>
                  <Col
                    span="10"
                    offset="4">
                    <Radio label="2">{{$t('message.common.toolbar.emptyString')}}</Radio>
                  </Col>
                </RadioGroup>
              </Row>
            </div>
            <div v-if="isAll">
              <Row>
                {{$t('message.common.toolbar.downloadMode')}}
              </Row>
              <Row>
                <Checkbox v-model="allDownload">{{$t('message.common.toolbar.all')}}</Checkbox>
              </Row>
            </div>
            <Row v-if="download.format == 2">
              <Checkbox v-model="autoFormat">{{$t('message.common.toolbar.autoformat')}}</Checkbox>
            </Row>
            <Row class="confirm">
              <Col span="10">
                <Button @click="cancelPopup('download')">{{$t('message.common.cancel')}}</Button>
              </Col>
              <Col
                span="10"
                offset="4">
                <Button
                  type="primary"
                  @click="confirm('download')">{{ $t('message.common.submit') }}</Button>
              </Col>
            </Row>
          </div>
        </Poptip>
      </li>
      <li
        @click="openPopup('export')"
        v-if="toolbarShow.export">
        <SvgIcon :style="{ 'font-size': '20px' }" icon-class="export" color="#515a6e"/>
        <span
          class="v-toolbar-icon"
          v-if="isIconLabelShow">{{ $t('message.common.toolbar.export') }}</span>
      </li>
      <li
        @click="openPopup('rowView')"
        v-if="row"
        :title="$t('message.common.toolbar.rowToColumnTitle')">
        <SvgIcon :style="{ 'font-size': '20px' }" icon-class="transform" color="#515a6e"/>
        <span v-if="isIconLabelShow" class="v-toolbar-icon">{{$t('message.common.toolbar.rowToColumn')}}</span>
      </li>
      <li
        @click="openPopup('filter')"
        :title="$t('message.common.toolbar.resultGroupLineFilter')"
        v-if="showFilter">
        <SvgIcon :style="{ 'font-size': '20px' }" icon-class="export" color="#515a6e"/>
        <span
          class="v-toolbar-icon"
          v-if="isIconLabelShow">{{$t('message.common.toolbar.lineFilter')}}</span>
      </li>
    </ul>
    <results-export
      ref="exportToShare"
      :script="script"
      :dispatch="dispatch"
      :current-path="currentPath">
    </results-export>
    <table-row
      ref="tableRow"
      :baseinfo="baseinfo"
      :row="row">
    </table-row>
  </div>
</template>
<script>
import moment from 'moment';
import elementResizeEvent from '@dataspherestudio/shared/common/helper/elementResizeEvent';
import resultsExport from './resultsExport.vue';
import tableRow from './tableRow.vue';
import mixin from '@dataspherestudio/shared/common/service/mixin';
import api from '@dataspherestudio/shared/common/service/api';
import storage from '@dataspherestudio/shared/common/helper/storage';
import plugin from '@dataspherestudio/shared/common/util/plugin'

const extComponents = plugin.emitHook('script_result_toolbar') || []
export default {
  name: 'toolbar',
  components: {
    resultsExport,
    tableRow,
  },
  mixins: [mixin],
  props: {
    currentPath: {
      type: String,
      default: '',
    },
    showFilter: {
      type: Boolean,
      default: false
    },
    script: [Object],
    row: {
      type: Array,
      default: () => [],
    },
    dispatch: {
      type: Function,
      required: true
    },
    getResultUrl: {
      type: String,
      defalut: `filesystem`
    },
    comData: {
      type: Object
    },
    resultType: {
      type: String,
      defalut: ''
    }
  },
  data() {
    // let appItem = 'linkis'
    // if (this.$route.name === 'Workflow') {
    //   appItem = 'workflow'
    // } else if(this.$route.name === 'ServicesExecute') {
    //   appItem = 'apiService'
    // }
    let rsDownload =  true // todo
    return {
      activeTool: 'table',
      rsDownload,
      popup: {
        download: false,
        export: false,
        rowView: false,
      },
      baseinfo: {},
      download: {
        format: '1',
        coding: '1',
        nullValue: '1',
      },
      isIconLabelShow: true,
      iconSize: 14,
      allDownload: false, // 是否下载全部结果集
      autoFormat: false,
      extComponents
    };
  },
  computed: {
    isAll() {
      return ['hql', 'sql'].includes(this.script.runType) && this.download.format === '2';
    },
    toolbarShow() {
      let isScriptis = this.$route.name === 'Home' || (this.$route.name === 'results' && this.$route.query.from === 'Home')
      return  {
        export: this.baseinfo.exportResEnable !== false && isScriptis && this.activeTool === 'table' && this.resultType === '2',
        download: this.activeTool === 'table' && this.rsDownload,
      }
    }
  },
  mounted() {
    this.baseinfo = storage.get("baseInfo", "local") || {}
    elementResizeEvent.bind(this.$el, this.resize);
  },
  methods: {
    openPopup(type) {
      if (type === 'download') {
        this.download = {
          format: '1',
          coding: '1',
          nullValue: '1',
        }
      }
      if (type === 'export') {
        this.$refs.exportToShare.open();
      } else if (type === 'rowView') {
        this.$refs.tableRow.open();
      } else if (type === 'filter') {
        this.$emit('on-filter');
      } else {
        this.popup[type] = true;
      }
    },
    cancelPopup(type) {
      this.popup[type] = false;
    },
    confirm(type) {
      this[`${type}Confirm`]();
      this.cancelPopup(type);
    },
    downloadConfirm() {
      this.$Modal.confirm({
        title: '提示',
        content: '当前进行数据下载，可能涉及敏感数据，请保障数据的安全',
        onOk: async () => {
          const splitor = this.download.format === '1' ? 'csv' : 'xlsx';
          const charset = this.download.coding === '1' ? 'utf-8' : 'gbk';
          const nullValue = this.download.nullValue === '1' ? 'NULL' : 'BLANK';
          const timestamp = moment.unix(moment().unix()).format('MMDDHHmm');
          let fileName = ''
          if ( this.script.fileName && this.script.fileName !== 'undefined') {
            fileName = this.script.fileName
          } else if(this.script.title && this.script.title !== 'undefined') {
            fileName = this.script.title
          }
          const filename = `Result_${fileName}_${timestamp}`;
          let temPath = this.currentPath;
          // api执行下载的结果集路径不一样
          let apiPath = `${this.getResultUrl}/resultsetToExcel`;
          if (this.isAll && this.allDownload) {
            temPath = temPath.substring(0, temPath.lastIndexOf('/'));
            apiPath = `${this.getResultUrl}/resultsetsToExcel`
          }
          let querys = 'path=' + temPath + '&charset=' + charset + '&outputFileType=' + splitor + '&nullValue=' + nullValue + '&outputFileName=' + filename + '&autoFormat=' + this.autoFormat;
          // 如果是api执行页获取结果集，需要带上taskId
          if(this.getResultUrl !== 'filesystem') {
            querys += `&taskId=${this.comData.taskID}`
          }
          let url = `${window.location.protocol}//${window.location.host}/api/rest_j/v1/${apiPath}?` + querys

          window.console.log(url, 'DOWNLOD-URL')

          // 下载之前条用心跳接口确认是否登录
          await api.fetch('/user/heartbeat', 'get');
          // 下载记录日志
          api.fetch('/dss/framework/admin/audit/script/download/save', {
            creator: this.baseinfo.username,
            tenant: this.baseinfo.proxyUserName,
            path: temPath,
            sql: this.script.executionCode || '',
            createTime: Date.now()
          }, 'post').catch(()=>{});
          const link = document.createElement('a');
          link.setAttribute('href', url);
          link.setAttribute('download', '');
          const evObj = document.createEvent('MouseEvents');
          evObj.initMouseEvent('click', true, true, window, 0, 0, 0, 0, 0, false, false, true, false, 0, null);
          const flag = link.dispatchEvent(evObj);
          this.$nextTick(() => {
            if (flag) {
              this.$Message.success(this.$t('message.common.toolbar.success.download'));
            }
          });
        }
      });
    },
    resize() {
      const height = window.getComputedStyle(this.$refs.toolbar).height;
      if (parseInt(height) <= 190) {
        this.isIconLabelShow = false;
        this.iconSize = 20;
      } else {
        this.isIconLabelShow = true;
        this.iconSize = 14;
      }
    },
    changeViewType(type, viewType = 'table') {
      if (type !== this.activeTool) {
        this.activeTool = type
      }
      this.$emit('change-view-type', viewType)
    },
    eventFromExt(evt) {
      if (evt && evt.callFn && typeof this[evt.callFn] === 'function') {
        this[evt.callFn](...evt.params)
      }
    }
  },
  beforeDestroy() {
    elementResizeEvent.unbind(this.$el, this.resize);
  },
};
</script>
<style lang="scss" scoped>
@import '@dataspherestudio/shared/common/style/variables.scss';
  .we-toolbar-wrap {
    width: $toolbarWidth;
    height: 100%;
    word-break: break-all;
    position: $absolute;
    left: 40px;
    margin-left: -$toolbarWidth;
    @include bg-color($light-base-color, $dark-menu-base-color);
    .we-poptip {
      padding: 12px;
      line-height: 28px;
      .confirm {
        margin-top: 10px;
      }
      .title {
        margin: 10px 0;
        text-align: center;
      }
    }
    .we-toolbar {
      width: 100%;
      height: 100%;
      border-right: 1px solid #dcdee2;
      @include border-color($border-color-base, $dark-menu-base-color);
      padding-top: 10px;
      li {
        padding-bottom: 20px;
        text-align: center;
        cursor: pointer;
      }
      span {
        display: block;
        line-height: 24px
      }
    }
    .we-toolbar-active {
      i,span {
        @include font-color($primary-color, $dark-primary-color);
      }
    }
  }
</style>

