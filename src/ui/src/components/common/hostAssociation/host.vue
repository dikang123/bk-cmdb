<template>
    <div>
        <input class="bk-form-input selected-host" type="text" readonly :value="localSelected.join(',')" @click="isSelectBoxShow = !isSelectBoxShow">
        <i class="bk-icon icon-close bk-selector-icon clear-icon" @click.stop="clear" v-show="localSelected.length"></i>
        <div class="selectbox-wrapper" v-show="isSelectBoxShow" @click.self="handleCancel">
            <div class="selectbox-box">
                <div class="top-box">
                    <p class="content-title">变更关联</p>
                    <div class="content-box">
                        <template>
                            <div class="operation-group clearfix">
                                <bk-button type="primary" :disabled="!ready" @click.stop="setCurrentPage(1)">刷新查询</bk-button>
                                <div class="fr">
                                    <bk-button type="default" class="btn-small" @click.stop="resetFilterParams">
                                        <i class="icon icon-cc-clear"></i>清空
                                    </bk-button>
                                    <bk-button type="default" class="btn-small" hidden>
                                        <i class="icon icon-cc-setting"></i>列表
                                    </bk-button>
                                </div>
                            </div>
                            <div class="slide-wrapper">
                                <v-filter
                                    ref="hostFilter"
                                    :queryColumns="filter.queryColumns"
                                    :attribute="attribute"
                                    @filterChange="setFilterParams">
                                </v-filter>
                                <v-table
                                    :tableHeader="table.header"
                                    :tableList="table.list"
                                    :defaultSort="table.defaultSort"
                                    :pagination="table.pagination"
                                    :chooseId.sync="table.chooseId"
                                    :isLoading="table.isLoading"
                                    :maxHeight="'auto'"
                                    :multipleCheck="multiple"
                                    @handlePageTurning="setCurrentPage"
                                    @handlePageSizeChange="setCurrentSize"
                                    @handleTableSortClick="setCurrentSort"
                                    @handleTableAllCheck="checkAllHost">
                                    <template v-for="({id, name, property}, index) in table.header" :slot="id" slot-scope="{ item }">
                                        <td v-if="id === 'bk_host_id'" style="width: 50px;" class="checkbox-wrapper">
                                            <label class="bk-form-checkbox bk-checkbox-small" @click.stop>
                                                <input type="checkbox" 
                                                    :value="item['host']['bk_host_id']"
                                                    :checked="table.chooseId.indexOf(item['host']['bk_host_id']) !== -1"
                                                    @change="setChoose(item['host']['bk_host_id'])">
                                            </label>
                                        </td>
                                        <td v-else>{{getCellValue(property, item)}}</td>
                                    </template>
                                </v-table>
                            </div>
                        </template>
                    </div>
                </div>
                <div class="bottom-box">
                    <div class="btn-group">
                        <bk-button type="primary" class="btn" @click="handleConfirm">确认</bk-button>
                        <bk-button type="default" class="btn" @click="handleCancel">取消</bk-button>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>
<script>
    import vTable from '@/components/table/table'
    import vFilter from './children/filter'
    import { mapGetters } from 'vuex'
    export default {
        props: {
            selected: {
                required: true,
                validator: (selected) => {
                    return Array.isArray(selected) || typeof selected === 'string' || typeof selected === 'undefined' || selected === null
                }
            },
            multiple: Boolean
        },
        data () {
            return {
                ready: false,
                localSelected: [],
                isSelectBoxShow: false,
                table: {
                    header: [],
                    list: [],
                    defaultSort: 'bk_host_id',
                    sort: 'bk_host_id',
                    pagination: {
                        current: 1,
                        size: 10,
                        count: 0
                    },
                    chooseId: [],
                    allHost: null,
                    isLoading: false
                },
                attribute: [{
                    'bk_obj_id': 'host',
                    'bk_obj_name': '主机',
                    'properties': []
                }, {
                    'bk_obj_id': 'module',
                    'bk_obj_name': '模块',
                    'properties': []
                }, {
                    'bk_obj_id': 'set',
                    'bk_obj_name': '集群',
                    'properties': []
                }, {
                    'bk_obj_id': 'biz',
                    'bk_obj_name': '业务',
                    'properties': []
                }],
                filter: {
                    queryColumns: [],
                    params: {}
                }
            }
        },
        computed: {
            ...mapGetters(['bkSupplierAccount']),
            bkBizId () {
                return this.filter.params.bk_biz_id
            },
            allProperties () {
                let allProperties = []
                this.attribute.forEach(({properties}) => {
                    allProperties = [...allProperties, ...properties]
                })
                return allProperties
            },
            defaultTableHeader () {
                let reservedProperty = ['bk_os_type', 'bk_cloud_id', 'bk_host_name', 'bk_host_innerip']
                return this.allProperties.filter(({bk_property_id: bkPropertyId, isrequired}) => {
                    return reservedProperty.indexOf(bkPropertyId) !== -1
                }).slice(0, 7).sort((propertyA, propertyB) => {
                    return reservedProperty.indexOf(propertyB['bk_property_id']) - reservedProperty.indexOf(propertyA['bk_property_id'])
                })
            }
        },
        watch: {
            async bkBizId (bkBizId, oldBkBizId) {
                this.table.chooseId = []
                if (!oldBkBizId || oldBkBizId === -1) {
                    await this.getAllAttribute()
                    await this.getUserCustomColumn()
                    this.setQueryColumns()
                    this.ready = true
                }
                await this.getAllHost()
                this.setCurrentPage(1)
            },
            isSelectBoxShow (isSelectBoxShow) {
                if (isSelectBoxShow) {
                    this.initChoosed()
                } else {
                    this.table.chooseId = []
                }
            },
            selected () {
                this.initLocalSelected()
            }
        },
        created () {
            this.initLocalSelected()
        },
        methods: {
            initLocalSelected () {
                if (Array.isArray(this.selected)) {
                    let availableSelected = this.selected.filter(({id}) => id !== '')
                    this.localSelected = availableSelected.map(({bk_inst_name: bkInstName}) => bkInstName)
                }
            },
            initChoosed () {
                if (Array.isArray(this.selected)) {
                    this.table.chooseId = this.selected.map(({bk_inst_id: bkInstId}) => bkInstId)
                    this.localSelected = this.selected.map(({bk_inst_name: bkInstName}) => bkInstName)
                } else if (typeof this.selected === 'undefined' || this.selected === '' || this.selected === null) {
                    this.table.chooseId = []
                } else {
                    this.table.chooseId = this.selected.split(',').map(bkHostId => parseInt(bkHostId))
                }
            },
            getAllAttribute () {
                return this.$Axios.all(this.attribute.map(({bk_obj_id: bkObjId}, index) => {
                    return this.getAttribute(bkObjId, index)
                }))
            },
            getAttribute (bkObjId, index) {
                let params = {
                    'bk_supplier_account': this.bkSupplierAccount,
                    'bk_obj_id': bkObjId,
                    page: {
                        sort: 'bk_property_name'
                    }
                }
                return this.$axios.post('object/attr/search', params).then(res => {
                    if (res.result) {
                        this.attribute[index]['properties'] = res.data
                    } else {
                        this.$alertMsg(res['bk_error_msg'])
                    }
                })
            },
            getUserCustomColumn () {
                return this.$axios.post('usercustom/user/search', {}).then(res => {
                    if (res.result) {
                        let hostAssociationColumns = res.data['host_association_column']
                        if (Array.isArray(hostAssociationColumns) && hostAssociationColumns.length) {
                            hostAssociationColumns = hostAssociationColumns.filter(({bk_property_id: bkPropertyId, bk_obj_id: bkObjId}) => {
                                return this.getColumnProperty(bkPropertyId, bkObjId)
                            })
                            hostAssociationColumns.length ? this.setTableHeader(hostAssociationColumns) : this.setTableHeader(this.defaultTableHeader)
                        } else {
                            this.setTableHeader(this.defaultTableHeader)
                        }
                    } else {
                        this.$alertMsg(res['bk_error_msg'])
                    }
                })
            },
            getColumnProperty (columnPropertyId, columnObjId) {
                return this.allProperties.find(({bk_property_id: bkPropertyId, bk_obj_id: bkObjId}) => {
                    return columnPropertyId === bkPropertyId && columnObjId === bkObjId
                })
            },
            setTableHeader (columns) {
                this.table.header = [{
                    id: 'bk_host_id',
                    name: 'bk_host_id',
                    type: 'checkbox'
                }].concat(columns.map(column => {
                    return {
                        id: column['bk_property_id'],
                        name: column['bk_property_name'],
                        property: column
                    }
                }))
            },
            setQueryColumns () {
                this.filter.queryColumns = [{
                    bk_property_id: 'bk_host_name',
                    bk_property_name: '主机名称',
                    bk_property_type: 'singlechar',
                    bk_obj_id: 'host'
                }, {
                    bk_option: '[{"name":"Linux", "type":"text"},{"name":"Windows", "type":"text"}]',
                    bk_property_id: 'bk_os_type',
                    bk_property_name: '操作系统类型',
                    bk_property_type: 'enum',
                    bk_obj_id: 'host'
                }]
            },
            setFilterParams (params) {
                this.filter.params = params
            },
            setCurrentPage (current) {
                this.table.pagination.current = current
                this.getTableList()
            },
            setCurrentSize (size) {
                this.table.pagination.size = size
                this.setCurrentPage(1)
            },
            setCurrentSort (sort) {
                this.table.sort = sort
                this.setCurrentPage(1)
            },
            setChoose (bkHostId) {
                let chooseId = this.table.chooseId
                if (this.multiple) {
                    let index = chooseId.indexOf(bkHostId)
                    if (index === -1) {
                        chooseId.push(bkHostId)
                    } else {
                        chooseId.splice(index, 1)
                    }
                } else {
                    if (chooseId.indexOf(bkHostId) === -1) {
                        chooseId = [bkHostId]
                    } else {
                        chooseId = []
                    }
                }
                this.table.chooseId = chooseId
            },
            getAllHost () {
                let searchParams = this.$deepClone(this.filter.params)
                searchParams.page = {}
                searchParams.condition.map(({bk_obj_id: bkObjId, fields}) => {
                    if (bkObjId === 'host') {
                        fields.push('bk_host_id')
                        fields.push('bk_host_innerip')
                    }
                })
                this.table.isLoading = true
                this.$axios.post('hosts/search', searchParams).then(res => {
                    if (res.result) {
                        this.table.allHost = res.data.info.map(item => {
                            return {
                                'bk_host_id': item['host']['bk_host_id'],
                                'bk_host_innerip': item['host']['bk_host_innerip']
                            }
                        })
                    } else {
                        this.$alertMsg(res['bk_error_msg'])
                    }
                    this.table.isLoading = false
                }).catch(() => {
                    this.table.isLoading = false
                })
            },
            getTableList () {
                this.table.isLoading = true
                this.$axios.post('hosts/search', this.filter.params).then(res => {
                    this.table.isLoading = false
                    if (res.result) {
                        this.table.pagination.count = res.data.count
                        this.table.list = res.data.info
                    } else {
                        this.$alertMsg(res['bk_error_msg'])
                    }
                }).catch(() => {
                    this.table.isLoading = false
                })
            },
            getCellValue (property, item) {
                if (!property) {
                    return null
                }
                let bkObjId = property['bk_obj_id']
                let value = item[bkObjId][property['bk_property_id']]
                if (property['bk_asst_obj_id'] && Array.isArray(value)) {
                    let tempValue = []
                    value.map(({id, bk_inst_name: bkInstName}) => {
                        if (id !== '' && bkInstName) {
                            tempValue.push(bkInstName)
                        }
                    })
                    value = tempValue.join(',')
                }
                return value
            },
            checkAllHost (isCheck) {
                if (isCheck) {
                    this.table.chooseId = this.table.allHost.map(({bk_host_id: bkHostId}) => bkHostId)
                } else {
                    this.table.chooseId = []
                }
            },
            resetFilterParams () {
                this.$refs.hostFilter.resetQueryColumnData()
                this.$nextTick(() => {
                    this.setCurrentPage(1)
                })
            },
            handleConfirm () {
                this.isSelectBoxShow = false
                this.setLocalSelected()
                let availableId = this.table.chooseId.filter(id => {
                    return !!this.table.allHost.find(({bk_host_id: bkHostId}) => bkHostId === id)
                })
                this.$emit('update:selected', availableId.join(','))
            },
            setLocalSelected () {
                let selectedHost = this.table.allHost.filter(({bk_host_id: bkHostId}) => this.table.chooseId.indexOf(bkHostId) !== -1)
                this.localSelected = selectedHost.map(({bk_host_innerip: bkHostInnerip}) => bkHostInnerip)
            },
            handleCancel () {
                this.isSelectBoxShow = false
                this.initLocalSelected()
            },
            clear () {
                this.table.chooseId = []
                this.handleConfirm()
            }
        },
        components: {
            vTable,
            vFilter
        }
    }
</script>

<style lang="scss" scoped>
    $white: #fff;
    $gray: #fafbfd;
    $textColor: #737987;
    $lineColor: #e5e5e5;
    .btn-small{
        padding: 0 10px;
    }
    .bk-form-input.selected-host[readonly]{
        background-color: #fff;
    }
    .selectbox-wrapper{
        position: fixed;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        z-index: 999;
        .selectbox-box{
            position: absolute;
            right: 32px;
            top: 50%;
            width: 735px;
            height: 526px;
            transform: translate(0, -50%);
            box-shadow: 0 2px 9.6px 0.4px rgba(0, 0, 0, .4);
            background: $white;
        }
        .top-box{
            padding: 30px 20px;
            height: calc(100% - 60px);
            .content-title{
                margin: 0;
                color: $textColor;
                font-size: 14px;
                font-weight: bold;
            }
            .content-box{
                position: relative;
                height: 100%;
                margin: 20px 15px 20px;
                .slide-wrapper{
                    margin-top: 15px;
                    height: calc(100% - 87px);
                    overflow-y: auto;
                    @include scrollbar;
                }
                .operation-group{
                    padding-top: 10px;
                    font-size: 0;
                    .btn-clear{
                        margin-right: 10px;
                    }
                }
                .icon{
                    position: relative;
                    top: -1px;
                    margin-right: 4px;
                    vertical-align: middle;
                }
            }
        }
        .bottom-box{
            height: 60px;
            padding-right: 36px;
            background: $gray;
            .btn-group{
                line-height: 60px;
                text-align: right;
                font-size: 0;
                .btn{
                    width: 88px;
                    text-align: center;
                    &:first-child{
                        margin-right: 10px;
                    }
                }
            }
        }
    }
</style>