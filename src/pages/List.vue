<template>
  <div>
    <v-layout>
      <v-flex md2></v-flex>
      <v-flex md9>
        <my-v-form class="row jr" :inline='true' v-model='filters.model' v-if="filters.fields" :fields='filters.fields' @submit='doSearch' submitButtonText='Search' submitButtonIcon='search'></my-v-form>
      </v-flex>
      <!-- dialog form start -->
      <v-flex md1>
        <v-dialog v-model="dialog" persistent dialog-transition max-width="500px">
          <v-btn v-show="options.create && options.open  === 'window'" fab dark color="indigo" @click="add()" slot="activator" class="mb-2" >
            <v-icon dark>add</v-icon></v-btn>

            <v-btn v-show="options.create && options.open  === 'page'" fab dark color="indigo" :to="{path:$route.path+'/create'}" slot="activator" class="mb-2" >
            <v-icon dark>add</v-icon></v-btn>
          <v-card>
          
            <my-form v-if="dialog" :dialog="true" :action="action" :items="this.currentItem" @close="close"></my-form>
          </v-card>
        </v-dialog>
      </v-flex>
    </v-layout>
     <!-- data list start -->
    <v-data-table 
    :headers="columns"
    v-model="selected"
    class="elevation-1" 
    select-all 
    :items='items' 
    :total-items="pagination.totalItems" 
    :pagination.sync="pagination" 
    :loading="loading">

    <!-- <template slot="headers" slot-scope="props">
      <tr>
        <th>
          <v-checkbox
            primary
            hide-details
            @click.native="toggleAll"
            :input-value="props.all"
            :indeterminate="props.indeterminate"
          ></v-checkbox>
        </th>
        <th
          v-for="header in props.headers"
          :key="header.text"
          :class="['column sortable', pagination.descending ? 'desc' : 'asc', header.value === pagination.sortBy ? 'active' : '']"
          @click="changeSort(header.value)"
        >
          <v-icon small>arrow_upward</v-icon>
          {{ header.text }}
        </th>
      </tr>
    </template> -->
      <template slot="items" slot-scope="props">
        <tr :active="props.selected" @click="props.selected = !props.selected">
        <td>
          <v-checkbox
            primary
            hide-details
            :input-value="props.selected"
          ></v-checkbox>
        </td>
        <td :class="column.left? '': 'text-xs-left'" v-for="(column, index) in columns" :key="column.text" v-if="index < columns.length - 1 " v-html="getColumnData(props.item, column)"></td>
       <!-- actions start -->
        <td v-if='actions !== false' width='200'>
          <v-btn v-if="actions.edit && options.open !== 'page' && options.edit !== false" icon class="mx-0" @click.native.stop="showEdit(props.item)">
            <v-icon color="teal">edit</v-icon></v-btn>
          <template v-for=" (value, action) in actions">
            <v-btn v-if="action != 'delete' && options.open === 'page' && options.delete !== false" icon :key="action+1" :to="{name: action, params: {resource,id:props.item.id}}">
              <v-icon>{{action.icon ? action.icon : action}}</v-icon></v-btn>
            <v-btn v-if="action == 'delete' && options.delete !== false" icon class="mx-0" :key="action" @click="deleteItem(props.index)">
              <v-icon color="pink">delete</v-icon></v-btn>
          </template>
          <template v-for="btns in actionBtn">
            <v-btn icon class="mx-0" :key="btns.name" @click.native.stop="btns.action(props.item)">
              <v-icon :color="btns.color">{{btns.icon}}</v-icon></v-btn>
          </template>
        </td>
        </tr>
      </template>
      <template slot="no-data">Sorry, nothing to display here :(</template></v-data-table>
  </div>
</template>

<script>
import actions from '@/actions'
const getDefaultData = () => {
  return {
    formHasErrors: false,
    valid: true,
    errorMessages: [],
    form: {
      model: {},
      fields: {},
      rules: {},
      messages: {},
      open: null
    },
    filters: {
      model: {},
      fields: null
    },
    otherMethod: {},
    loading: false,
    columns: [], // fetch grid setup params from server if `columns` is empty
    actions: {},
    actionBtn: {},
    options: {
      sort: 'id',
      create: false,
      update: true,
      delete: false
    },
    pagination: {
      page: 1,
      rowsPerPage: 10,
      sortBy: 'id',
      descending: true,
      totalItems: 0
    },
    isShowEdit: false,
    currentItem: false,
    items: [],
    dialog: false,
    item: {},
    selected: [],
    action: 'add',
    search: null
  }
}
export default {
  data: getDefaultData,
  computed: {
    resource () {
      return this.$route.params.resource
    },
    totalPages () {
      return Math.ceil(this.pagination.totalItems / this.pagination.rowsPerPage)
    }
  },
  created () {
    console.log(this.$route)
    this.initialize()
  },
  watch: {
    '$i18n.locale' (val) {
      this.fetchGrid()
    },
    'pagination.page' (val) {
      this.fetchData()
    },
    'pagination.rowsPerPage' (val) {
      this.fetchData()
    },
    'pagination.sortBy' (val) {
      this.fetchData()
    },
    'pagination.descending' (val) {
      this.fetchData()
    },
    '$route.params': 'refresh',
    '$route.query': 'updateRoute'
  },
  methods: {
    initialize () {
      this.fetchGrid().then(() => {})
      this.fetchData()
    },
    toggleAll () {
      if (this.selected.length) this.selected = []
      else this.selected = this.items.slice()
    },
    fetchData () {
      this.preFetch()
      this.loading = true
      this.$http.get(`${this.resource}`, {params: this.$route.query}).then((data) => {
        this.items = data.data
        // this.currentItem = this.items[0]
        this.pagination.totalItems = data.total
        this.loading = false
        this.actionBtn = actions[this.resource]
      })
    },
    fetch () {
      if (this.columns.length <= 0) {
        // fetch grid params from server: e.g. /crud/${resource}/grid
        this.fetchGrid()
      } else {
        // or define grid params manually
        this.fetchData()
      }
    },
    doSearch () {
      this.pagination.page = 1
      this.fetchData()
    },
    filter (val, search) {
      return true
      // this.search = search
      // this.fetchData()
    },
    getColumnData (row, field) {
      // process fields like `type.name`
      let [l1, l2] = field.value.split('.')
      let value = row[l1]
      let tag = null
      if (l2) {
        value = row[l1] ? row[l1][l2] : null
      }
      if (field.type === 'image') {
        tag = 'img'
      }
      if (tag) {
        value = `<${tag} src="${value}" class="crud-grid-thumb" controls />`
      }
      return value
    },
    fetchGrid () {
      return new Promise((resolve, reject) => {
        this.$http.get(`${this.resource}/grid`).then((data) => {
          for (let k in data.columns) {
            data.columns[k].text = this.$t(data.columns[k].text)
          }
          data.columns.push({text: '操作', value: '操作', sortable: false})
          this.columns = data.columns || {}
          this.actions = data.actions || {}
          this.filters = data.filters || {}
          this.options = data.options || {}
          this.act = data.act

          if (this.options && this.options.sort) {
            let sortData = this.options.sort.split('-')
            let desc = sortData.length > 1
            let sortField = sortData.pop()
            // if (sortField.indexOf('.') < 0) {
            //   sortField = sortField
            // }
            this.pagination.sort = sortField
            this.pagination.descending = desc
          }
          resolve()
        })
      })
    },
    preFetch () {
      let sort = this.pagination.sortBy
      if (this.pagination.descending) {
        sort = '-' + sort
      }
      this.$route.query.query = JSON.stringify(this.filters.model)
      this.$route.query.sort = sort
      this.$route.query.perPage = this.pagination.rowsPerPage
      this.$route.query.page = this.pagination.page
    },
    updateRoute () {
      console.log(33)
      this.$route.query.keepLayout = true
      console.log('update route')
      this.$router.push({
        path: this.$route.path,
        params: this.$route.params,
        query: this.$route.query
      })
    },
    refresh () {
      Object.assign(this.$data, getDefaultData())
      this.fetchGrid().then(() => {})
      this.fetchData()
    },
    editItem (item) {
      this.currentItem = item
      this.fetchForm(item)
      this.dialog = true
      this.isShowEdit = true
    },
    deleteItem (index) {
      // alert user for delete
      this.$http.delete(`${this.resource}/${this.items[index].id}`).then((data) => {
        console.log(data)
        if (data.result) {
          this.$http.open({
            body: data.messages || 'errors',
            color: 'success',
            timeout: 2000
          })
          this.items.splice(index, 1)
        } else {
          this.$http.open({
            body: data.messages,
            color: 'error',
            timeout: 8000
          })
        }
      })
    },

    close () {
      this.item.edit = false
      this.dialog = false
    },
    onSaveEdit (data) {
      if (data.id) {
        this.isShowEdit = false
        this.fetchData()
      }
    },
    showEdit (item) {
      this.currentItem = item
      this.dialog = true
      this.action = 'edit'
      this.isShowEdit = true
    },
    add () {
      this.action = 'add'
      this.edit = false
      this.isShowEdit = false
      // this.currentItem = false
    }
  },
  beforeRouteUpdate (to, from, next) {
  // just use `this`
    this.name = to.params.name
    console.log(this.name)
    next()
  }
}
</script>