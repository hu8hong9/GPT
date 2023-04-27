<template>
  <el-container>
    <el-header v-if="groupTabs?.length > 0" class="header-tabs">
      <el-tabs type="card" size="small" v-model="groupTabName" @tab-change="groupTabChange">
        <template v-for="(item, index) in groupTabs" :key="index">
          <el-tab-pane :label="item.label" :name="item.name"></el-tab-pane>
        </template>
      </el-tabs>
    </el-header>
    <el-main class="nopadding">
      <el-container v-loading="loading" class="wode-table">
        <el-header>
          <wode-tool-panel>
            <template v-slot:left>
              <slot name="top-form"></slot>
            </template>
            <template v-slot:right>
              <!--日期查询控件-->
              <wode-date-picker v-if="showDatePicker" v-model="searchForm.date_picker" v-bind="datePickerConfig" @change="this.research"></wode-date-picker>
              <!--按钮-->
              <el-input v-model="searchForm.keyword" :placeholder="keywordInputPlaceholder">
                <template #prepend>
                  <el-tooltip placement="top" content="新增">
                    <el-button icon="el-icon-plus" type="primary" @click="$emit('add')"></el-button>
                  </el-tooltip>
                </template>
                <template #append>
                  <el-button-group>
                    <el-tooltip placement="top" content="查询">
                      <el-button icon="el-icon-search" @click="research"></el-button>
                    </el-tooltip>
                    <el-tooltip placement="top" content="重置">
                      <el-button icon="el-icon-refresh-left" type="warning" @click="reset"></el-button>
                    </el-tooltip>
                  </el-button-group>
                </template>
              </el-input>
              <el-tooltip placement="top" content="是否开启远程筛选">
                <el-switch v-if="hideRemoteFilterSwitch" v-model="remoteFilterSwitch" active-color="#13ce66" inactive-color="#ff4949">
                </el-switch>
              </el-tooltip>
            </template>
          </wode-tool-panel>
        </el-header>
        <el-main class="nopadding">
          <el-table :ref="tableName" :data="tableData" v-bind="$attrs" @sort-change="sortChange" @filter-change="filterChange" height="100%">
            <!--表格列配置-->
            <template v-for="(item, index) in userViewColumn" :key="index">
              <el-table-column v-if="(typeof item.vIf === 'function') ? item.vIf() : item.vIf ?? true" :column-key="item.prop" :label="item.label" :prop="item.prop"
                               :width="item.width" :sortable="item.sortable" :fixed="item.fixed" :filters="item.filters" :filter-multiple="item.filterMultiple"
                               :filter-method="remoteFilter||!item.filters?null:filterHandler" :filtered-value="item.filteredValue"
                               :show-overflow-tooltip="item.showOverflowTooltip" :min-width="item.miniWidth">
                <template #default="scope">
                  <!--如果配置了呈现属性，根据呈现类型，展示不同的控件-->
                  <template v-if="item.render">
                    <!--【函数呈现】-->
                    <template v-if="typeof item.render === 'function'">
                      <span v-html="item.render(scope.row)"></span>
                    </template>
                    <!--【组件呈现】-->
                    <template v-else>
                      <!--链接组件-->
                      <el-link v-if="item.render?.component === 'el-link'" target="_blank"
                               :href="(typeof item.render.href === 'function') ? item.render.href(scope.row) : scope.row[item.render.href]">
                        {{scope.row[item.render.label]}}</el-link>
                      <!--时间显示-->
                      <template v-else-if="item.render?.component === 'date-time'">
                        <!-- 函数呈现-->
                        <span>{{ this.dateTimeFormat(scope.row[item.prop] ,item.render.format) }}</span>
                      </template>
                      <!--图片组件-->
                      <template v-else-if="item.render?.component === 'el-link-image'">
                        <el-link target="_blank" :href="(typeof item.render.href === 'function') ? item.render.href(scope.row) : scope.row[item.render.href]">
                          <el-image :src="scope.row[item.render.src]" :fit="item.render.fit" :lazy="item.render.lazy" :style="item.render.style"></el-image>
                        </el-link>
                      </template>
                      <el-image v-else-if="item.render?.component === 'el-image'" :src="scope.row[item.render.src]" :fit="item.render.fit" :lazy="item.render.lazy"
                                :style="item.render.style"></el-image>
                      <!--图标组件-->
                      <template v-else-if="item.render?.component === 'icon'">
                        <el-icon>
                          <component :is="(typeof item.render.icon === 'function') ? item.render.icon(scope.row) : scope.row[item.render.icon]" />
                        </el-icon>
                      </template>
                      <!--文本查看-->
                      <el-input v-else-if="item.render?.component === 'el-input'" :type="item.render.type" :autosize="item.render.autosize"
                                :disabled="item.render.disabled" :style="item.render.style" v-model="scope.row[item.render.label]"></el-input>
                      <!--按钮组件-->
                      <el-button v-else-if="item.render?.component === 'el-button'" :type="item.render.type ?? 'danger'" :size="item.render.size ?? 'small'"
                                 :link="item.render.link ?? true" @click="item.render.click(scope.row)">
                        {{scope.row[item.prop]}}
                      </el-button>
                      <!--文本与标签组件-->
                      <template v-else-if="item.render?.component === 'el-tag'">
                        <el-tag :type="(typeof item.render.type === 'function') ?item.render.type(scope.row) : '' ">
                          {{ (typeof item.render.showValue === 'function') ? item.render.showValue(scope.row) : scope.row[item.prop]}}
                        </el-tag>
                      </template>
                      <template v-else-if="item.render?.component === 'el-text'">
                        <el-text :type="(typeof item.render.type === 'function') ?item.render.type(scope.row) : '' ">
                          {{ (typeof item.render.showValue === 'function') ? item.render.showValue(scope.row) : scope.row[item.prop]}}
                        </el-text>
                      </template>
                      <!--其他组件-->
                      <component v-else :is="item.render?.component" v-model="scope.row[item.prop]" :original="scope.row" v-bind="item.render?.vBind"
                                 v-on="item.render?.vOn">
                        {{ (typeof item.render.showValue === 'function') ? item.render.showValue(scope.row) : scope.row[item.prop]}}
                      </component>
                    </template>
                  </template>
                  <!--默认展示字段内容-->
                  <template v-else>
                    <slot :name="item.prop" v-bind="scope">
                      {{scope.row[item.prop]}}
                    </slot>
                  </template>
                </template>
              </el-table-column>
            </template>
            <slot name="column-handle"></slot>
          </el-table>
        </el-main>
        <el-footer>
          <wode-tool-panel>
            <template v-slot:left>
              <el-button type="primary" icon="el-icon-download" @click="download" v-if="showDownload">下载</el-button>
            </template>
            <template v-slot:right>
              <wodo-table-pagination v-model="pagination" v-show="showPagination"></wodo-table-pagination>
              <el-button-group v-if="hideFooterColumnConfig">
                <el-popover placement="top" title="列设置" :width="500" trigger="click" :hide-after="0" @show="customColumnShow=true"
                            @after-leave="customColumnShow=false">
                  <template #reference>
                    <el-button icon="el-icon-setting" style="margin-left:15px">列设置</el-button>
                  </template>
                  <column-setting v-if="customColumnShow" ref="columnSetting" @userChange="columnSettingChange" @save="columnSettingSave" @back="columnSettingBack"
                                  :column="userColumn"></column-setting>
                </el-popover>
                <el-button icon="el-icon-set-up">筛选</el-button>
              </el-button-group>
            </template>
          </wode-tool-panel>
        </el-footer>
      </el-container>
    </el-main>
  </el-container>
</template>
<script>
import WodoTablePagination from './components/TablePagination'
import ColumnSetting from './components/TableColumnSetting'
import db from '@/mixins/db'
import _ from 'lodash'
import dayjs from 'dayjs'

export default {
  emits: ['select-over', 'filter-change'],
  name: 'wode-table',
  mixins: [db],
  setup(props) {
    //:>[wode-table-setup],表格属性:|props
    {
      props
    }
  },
  components: {
    WodoTablePagination,
    ColumnSetting,
  },
  props: {
    //表格配置信息
    wOption: {
      type: Object,
      default: () => {
        return {
          //+.布局配置
          layout: {},
          //+.查询配置
          select: {
            //是否加载时查询数据
            loadSelect: false,
            //是否远程排序
            remoteSort: true,
            //是否远程过滤
            remoteFilter: true,
          },
          //+.分组查询
          groupTabDefaultName: '',
          groupTabs: [],
          //+.列配置
          columns: {},
          refs: {},
          //+.API地址
          api: {},
          //+.表格名称
          tableName: '',
        }
      },
    },
  },
  watch: {
    // 监控翻页组件改变，刷新页面
    'pagination.current': {
      handler() {
        //:>[table],改变当前页面|this.pagination.current
        //this.select()
      },
      deep: true,
    },
    'pagination.size': {
      handler() {
        //:>[table],改变页面大小|this.pagination.size
        //this.select()
      },
      deep: true,
    },
  },
  //!.=================计算属性=====================================================
  computed: {
    //+.顶部分组赛选配置
    groupTabDefaultName() {
      return this.wOption?.groupTabDefaultName ?? 'out'
    },
    groupTabs() {
      return this.wOption?.groupTabs ?? []
    },
    //+.refs配置
    tableName() {
      return this.wOption?.tableName ?? ''
    },
    //+.api
    api() {
      return this.wOption?.api ?? null
    },
    //+.表格列配置，如果设置此属性，以此为准，否则通过API接口获取
    tableColumn() {
      return this.wOption?.columns ?? null
    },
    //+.分页大小
    pageSize() {
      return this.wOption?.pageSize ?? 15
    },
    //+.查询相关配置
    /**
     * 是否加载时查询数据
     */
    loadSelect() {
      return this.wOption?.select?.loadSelect ?? false
    },
    /**
     * 是否远程排序
     */
    remoteSort() {
      return this.wOption?.select?.remoteSort ?? true
    },
    /**
     * 是否远程过滤
     */
    remoteFilter() {
      //:>是否远程过滤|this.wOption?.select?.remoteFilter
      return this.wOption?.select?.remoteFilter ?? true
    },
    //+.布局与工具栏相关配置
    //是否显示翻页组件，默认显示
    showPagination() {
      return _.isNil(this.wOption?.layout?.hidePaginstion) ? true : !this.wOption?.layout?.hidePaginstion
    },
    //是否显示日期组件，默认显示
    showDatePicker() {
      return _.isNil(this.wOption?.layout?.hideDatePicker) ? true : !this.wOption?.layout?.hideDatePicker
    },
    //!.DatePicker配置
    datePickerConfig() {
      return _.isNil(this.wOption?.layout?.datePickerConfig) ? true : !this.wOption?.layout?.datePickerConfig
    },
    //是否显示下载按钮，默认显示
    showDownload() {
      return _.isNil(this.wOption?.layout?.hideDownload) ? true : !this.wOption?.layout?.hideDownload
    },
    //是否显示远程过滤切换按钮，默认显示
    showRemoteFilterSwitch() {
      return _.isNil(this.wOption?.layout?.hideRemoteFilterSwitch) ? true : !this.wOption?.layout?.hideRemoteFilterSwitch
    },
    //是否显示底部列配置按钮
    hideFooterColumnConfig() {
      return _.isNil(this.wOption?.layout?.hideFooterColumnConfig) ? true : !this.wOption?.layout?.hideFooterColumnConfig
    },
    //关键词搜索框提示文本
    keywordInputPlaceholder() {
      return _.isNil(this.wOption?.layout?.keywordInputPlaceholder) ? '' : this.wOption?.layout?.keywordInputPlaceholder
    },
  },
  data() {
    //:>[table-data]查看数据：|this.wOption
    return {
      //表格加载状态
      loading: true,
      // 翻页基本配置
      pagination: { current: 1, size: 15, total: 0 },
      //隐藏表单，不受重置参数影响
      hideForm: {},
      // 查询表单
      searchForm: {
        group: this.groupTabDefaultName,
        keyword: '',
        date_picker: null,
      },
      //+.分组信息
      groupTabName: this.wOption?.groupTabDefaultName ?? '',
      //+.排序信息：
      selectOrder: {},
      //远程过滤开关
      remoteFilterSwitch: true,
      //过滤信息
      filterValues: {},
      // 表格状态
      status: {
        // 是否加载数据
        isLoadingData: false,
      },
      // 默认视图
      defaultViewColumn: [],
      // 用户视图
      userViewColumn: [],
      // 表格数据
      tableData: [],
      tableConfig: false,
    }
  },
  async created() {
    const end = new Date()
    const start = new Date()
    start.setTime(start.setHours(0, 0, 0, 0))
    end.setTime(end.setHours(23, 59, 59, 999))
    this.searchForm.date_picker = [start, end]
  },
  //!.在实例挂载完成后被调用
  async mounted() {
    this.remoteFilterSwitch = this.remoteFilter
    this.defaultViewColumn = this.tableColumn
    this.pagination = { current: 1, size: this.pageSize, total: 0 }
    //获取当前列默认设置
    await this.loadColumnConfig()
    if (this.loadSelect) {
      //:)[table],加载时查询数据
      await this.select()
    } else {
      //:([table],加载时不查询数据
    }
    //:)[table],实例挂载完成
  },
  methods: {
    /* 分组查询数据 */
    groupTabChange() {
      //:)[table],分组查询更改
      this.upHideForm({ group: this.groupTabName })
      this.select()
    },
    dateTimeFormat(date, format) {
      //:<时间格式
      return date === null ? '-' : dayjs(date).format(format ?? 'YYYY-MM-DD')
    },
    // 加载列模式
    async loadColumnConfig() {
      //:)[table],加载表格列配置
      if (_.isNil(this.defaultViewColumn)) {
        //->[table],没有配置列信息，通过API获取表格列属性
        let columnData = await this.api.column.request()
        if (_.isNil(columnData)) {
          this.$message.warning('获取表格列配置失败！！')
          return false
        }
        //:>[table],返回列数据：|columnData
        this.defaultViewColumn = columnData
      }
      //:>[table],默认视图：|this.defaultViewColumn

      let saveViewColumn = this.userTableDataGet().userViewColumn || null
      //:>[table],读取用户保存列数据：|saveViewColumn

      if (_.isNil(saveViewColumn)) {
        this.userViewColumn = this.defaultViewColumn
      }
      //:>[table],原始视图与用户视图合并：|this.userViewColumn
    },
    // 查询数据
    async select(form = {}) {
      this.loading = true
      //+.重新查询，页面回到第一页，需要判断参数是否更新
      //this.pagination.current = 1
      //0、开始生成查询请求数据
      //1、先合并表单数据
      Object.assign(this.searchForm, this.hideForm)
      Object.assign(this.searchForm, form)
      //2、生成标准的查询格式
      // let selectReqData = {
      //   //查询表单信息
      //   search_form: this.searchForm,
      //   //翻页数据
      //   pagination: this.pagination,
      //   //排序信息
      //   order: {},
      // }

      //:>[table-select],查询数据：|this.searchForm
      let selectData = await this.api.select.request({ search_form: this.searchForm, pagination: this.pagination, order: this.selectOrder })
      //:>[table-select],查询返回数据：|selectData
      if (selectData.total >= 0) {
        if (selectData.total === 0) {
          this.$message.warning('没有相关数据！！')
          this.tableData = []
          this.loading = false
          return false
        }
        //记录条数
        this.pagination.total = selectData.total
        //记录数据
        this.tableData = selectData.list
        //数据处理
        this.$emit('select-over', this.tableData)
      } else {
        this.tableData = selectData
      }
      //:)数据加载完成：|this.loading
      this.loading = false
    },
    // 更新隐藏表单
    upHideForm(data = {}) {
      Object.assign(this.hideForm, data)
    },
    // 刷新数据
    research() {
      //:)[table-select],刷新数据
      this.pagination.current = 1
      this.select()
    },
    // 重置查询
    reset() {
      //:)[table-select],重置查询
      //用于多选表格，清空用户的选择
      this.$refs[this.tableName].clearSelection()
      //用于清空排序条件，数据会恢复成未排序的状态
      this.$refs[this.tableName].clearSort()
      //不传入参数时用于清空所有过滤条件，数据会恢复成未过滤的状态，也可传入由columnKey组成的数组以清除指定列的过滤条件
      this.$refs[this.tableName].clearFilter()

      //清空查询条件
      this.searchForm = {}
      //查询数据
      this.select()
    },

    //+.===========排序与过滤================================
    //排序事件
    sortChange(obj) {
      if (obj.column && obj.prop) {
        this.selectOrder = {
          field: obj.prop,
          order: obj.order,
        }
      } else {
        this.sortable = {}
      }
      this.select()
    },
    //本地过滤
    filterHandler(value, row, column) {
      //:>本地过滤事件：|value
      const property = column.property
      return row[property] === value
    },
    //过滤事件
    filterChange(filters) {
      this.filterValues = filters
      //:>远程过滤事件：|filters,this.remoteFilter
      if (!this.remoteFilterSwitch) {
        return false
      }
      // Object.keys(filters).forEach((key) => {
      //   filters[key] = filters[key].join(',')
      // })
      //:>过滤事件：|filters
      this.select(filters)
    },
    //+.数据增、删、改等相关操作
    // 数据增、删、改、查
    add() {},
    edit() {},
    del() {},

    setTableColumnFilteredValue(prop, value) {
      let column = this.$refs[this.tableName].tableColumn
      //:>企业收藏状态更改：|column
      column.forEach((element) => {
        if (element.prop === prop) {
          element.filteredValue = value
        }
      })
      return []
    },
  },
}
</script>