<template src="./template.html"></template>

<style lang="scss" scoped>
  @mixin ellipsis() {
    display: inline-block;
    max-width: 100%;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    vertical-align: middle;
  }
  .multiple-select {
    font-size: 14px;
    height: 100%;
    width: 100%;
    @import "./style/controls.scss";
    @import "./style/preview.scss";
    @import "./style/header.scss";
    @import "./style/search.scss";

    & > .select-box {
      width: 100%;
      height: 100%;
      display: flex;
      .select-controls {
        flex: 1;
      }
    }
  }
</style>

<style lang="scss">
  .autocomplete-search {
    .name {
      b {
        color: #409eff;
      }
    }
  }
</style>

<script>
import _ from 'lodash';
import DB from '@fengqiaogang/dblist';

export default {
  data () {
    const db = new DB('select', [], this.primaryKey, this.foreignKey);
    return {
      list: db, // 所有数据, 
      showSelect: [].concat(this.topLevel),
      // 选中的数据
      checkList: [],
      // 半选状态
      indeterminate: {},
      // 搜索框内容
      searchValue: '',
      // 筛选后的选中数据
      checked: []
    }
  },
  props: {
    labelKey: {
      type: String,
      default: () => 'name',
    },
    // 主键
    primaryKey: {
      type: String,
      default: () => 'id',
    },
    foreignKey: {
      type: String,
      default: () => 'pid',
    },
    topLevel: {
      type: [String, Number],
      default: () => 0
    },
    title: {
      type: String,
    },
    preview: {
      type: Boolean,
      default: () => false
    },
    previewTitle: {
      type: String
    },
    childrenKey: {
      type: String,
      default: () => 'children'
    },
    /** 是否支持搜索 */
    search: {
      type: Boolean,
      default: () => false
    },
    /** 搜索框占位提示语 */
    placeholder: {
      type: String,
      default: () => '请输入需要搜索的内容'
    },
    /**
     * 搜索框与内容是否有间隔
     * @param 0 默认无间隔
     */
    searchGutter: {
      type: Number,
      default: () => 0,
      validator(value) {
        return value >= 0 ? true : false;
      }
    },
    /**
     * 获取搜索数据
     */
    getSearchServer: {
      type: Function,
      default() {
        return function() {
          return [];
        }
      }
    },
    /**
     * 获取列表数据
     */
    getColumnServer: {
      type: Function,
      default() {
        return function() {
          return [];
        }
      }
    }
  },
  async mounted() {
    const insert = '_insertData';
    const result = await this.getColumnData(this.topLevel);
    const array = this.setLevels(result, this.topLevel, 0);
    this[insert](array);
    this.$emit('load');
  },
  methods: {
    setAutoChecked(keys) {
      keys = _.flattenDeep([].concat(keys));
      const indeterminate = Object.assign({}, this.indeterminate);
      const checkList = _.map(this.checkList, value => [].concat(value));
      const app = (foreignKey) => {
        const db = this.list;
        const where = this.getForeignKeyWhere(foreignKey);
        const list = db.select(where);
        for(let i = 0, len = list.length; i < len; i++) {
          const item = list[i];
          const key = item[this.primaryKey];
          if (_.includes(keys, key)) {
            const childrens = db.childrenDeep(this.getPrimaryKeyWhere(key));
            _.each(childrens, data => {
              const level = data.level;
              const id = data[this.primaryKey];
              const parents = db.parentDeep(this.getPrimaryKeyWhere(id));
              _.each(parents, parent => {
                indeterminate[parent[this.primaryKey]] = true;
              });
              indeterminate[id] = false;
              checkList[level].push(id);
            })
          } else {
            app(key);
          }
        }
      }
      app(this.topLevel);
      this.checkList = checkList;
      this.indeterminate = indeterminate;
    },
    
    /**
     * 设置 level 级别
     */
    setLevels(list, foreignKey, level = 0) {
      const array = [];
      const db = new DB('leve', [], this.primaryKey, this.foreignKey);
      db.insert(db.flatten(list, this.childrenKey));
      const app = (key, levelValue) => {
        const where = this.getForeignKeyWhere(key);
        const list = db.select(where);
        for(let i = 0, len = list.length; i < len; i++) {
          const item = Object.assign({}, list[i], { level: levelValue });
          array.push(item);
          app(item[this.primaryKey], levelValue + 1);
        }
      }
      app(foreignKey, level);
      return array;
    },
    _insertData (array) {
      const checkList = [].concat(this.checkList);
      const db = this.list;
      const levels = new Set();
      for(let i = 0, len = array.length; i < len; i++) {
        const item = array[i];
        const level = item.level ?? 0;
        levels.add(level);
      }
      for(const index of levels) {
        const value = checkList[index];
        if (value) {
          continue;
        }
        checkList[index] = [];
      }
      this.checkList = checkList;
      db.insert(array);
    },
    // 查询外键列表数据
    getForeignKeyWhere(value) {
      const where = {};
      where[this.foreignKey] = value;
      return where;
    },
    // 查询主键列表数据
    getPrimaryKeyWhere(value) {
      const where = {};
      where[this.primaryKey] = value;
      return where;
    },
    // 查询当前列数据
    getColumnList(value) {
      const db = this.list;
      const where = this.getForeignKeyWhere(value);
      return db.select(where);
    },
    // 判断是否有子集
    isHaveSon(item) {
      const db = this.list;
      const where = this.getForeignKeyWhere(item[this.primaryKey]);
      const list = db.select(where);
      return list.length > 0 ? true : false;
    },
    // 维护展示的列表数据
    amendShowSelect(item, showNext) {
      const db = this.list;
      const primaryKey = item[this.primaryKey];
      // 从数据中获取最正确的数据
      item = db.selectOne(this.getPrimaryKeyWhere(primaryKey));
      // 获取数据链条
      const parentDeep = db.parentDeep(this.getPrimaryKeyWhere(primaryKey));
      const array = [].concat(parentDeep.map(data => data[this.primaryKey]), this.topLevel);
      // 设置每一栏的键值
      if (showNext) {
        this.showSelect = array.reverse();
      } else {
        this.showSelect = array.slice(1).reverse();
      }
    },
    // 点击元素文字
    async checkboxClick(item) {
      this.amendShowSelect(item, true);
      const primaryKey = item[this.primaryKey];
      // 处理子级数据
      const haveSon = this.isHaveSon(item);
      if (!haveSon) {
        const level = item.level;
        const list = await this.getColumnData(primaryKey);
        if (list) {
          // 设置 level 级别
          const arr = this.setLevels(list, primaryKey, level + 1);
          const insert = '_insertData';
          this[insert](arr);
        }
        const checkeds = this.checkList[level];
        if (_.includes(checkeds, primaryKey)) {
          this.checkboxChange(true, item);
        } else {
          this.checkboxChange(false, item);
        }
      }
    },
    // 复选框选中与取消
    checkboxChange(status, data) {
      const db = this.list;
      data = db.selectOne(this.getPrimaryKeyWhere(data[this.primaryKey]));
      // 半选数据
      let indeterminate = Object.assign({}, this.indeterminate);
      const checkList = [];
      // 取消自身半选状态
      indeterminate[data[this.primaryKey]] = false;
      // 设置子元素选中状态
      const childrenDeep = db.childrenDeep(this.getPrimaryKeyWhere(data[this.primaryKey]));
      _.each(childrenDeep, item => {
        const level = item.level;
        const value = item[this.primaryKey];
        indeterminate[value] = false;
        checkList[level] = [].concat(checkList[level], value);
      });
      // 选中
      if (status) {
        _.each(this.checkList, (column, index) => {
          const arr = [].concat(column || [], checkList[index] || []);
          checkList[index] = _.compact(_.uniq(arr));
        });
      } else {
        _.each(this.checkList, (column, index) => {
          const arr = _.difference(column || [], checkList[index] || []);
          checkList[index] = _.compact(arr);
        });
      }
      // 修正父级元素选中状态
      const prev = (id) => {
        const where = this.getPrimaryKeyWhere(id);
        const item = db.selectOne(where);
        if (!item) {
          return false;
        }
        const level = _.toInteger(item.level);
        if (level < 0 || level - 1 < 0) {
          
          return false;
        }
        // 查询当前列数据
        const foreignKey = item[this.foreignKey];
        const childrenlist = this.getColumnList(foreignKey);
        // 临时db
        const tempDb = new DB('temp', childrenlist, this.primaryKey, this.foreignKey);
        // 查询当前列已选中的数据
        const selectCheckboxValues = _.compact([].concat(checkList[level] || []));
        const childrenCheckendList = tempDb.select(this.getPrimaryKeyWhere(selectCheckboxValues));
        // 上一级 checkend 下标
        const prevIndex = level - 1;
        if (childrenCheckendList.length === childrenlist.length && childrenlist.length > 0) { // 如果当前列数据的个数与已选数据的个数相同
          indeterminate[foreignKey] = false;
          // 向父级数据添加一个数据
          const arr = [].concat(checkList[prevIndex] || [], foreignKey);
          // 去重去空
          checkList[prevIndex] = _.compact(_.uniq(arr));
        } else {
          // 获取父级数据
          let arr = [].concat(checkList[prevIndex] || []);
          // 从数组中删除某元素
          arr = _.difference(arr, [foreignKey]);
          // 去空
          checkList[prevIndex] = _.compact(arr);
          // 判断是否需要半选
          if (childrenCheckendList.length === 0) { // 已选数据等于0
            let flag = false;
            for(const temp of childrenlist) {
              const key = temp[this.primaryKey];
              if (indeterminate[key]) {
                flag = true;
                break;
              }
            }
            indeterminate[foreignKey] = flag;
          } else {
            indeterminate[foreignKey] = true;
          }
        }
        prev(foreignKey);
      };
      prev(data[this.primaryKey]);
      this.checkList = checkList;
      this.indeterminate = indeterminate;
      this.onInput();
    },
    onInput() {
      const db = this.list;
      // 所有已选中的数据
      const checkList = _.map(this.checkList, item => [].concat(item));
      // 需要删除的数据
      let difference = [];
      let result = _.flattenDeep(checkList);
      for(let i = 0, len = checkList.length; i < len - 1; i++) {
        const checkends = checkList[i];
        for(let j = 0, size = checkends.length; j < size; j++) {
          const key = checkends[j];
          if (_.includes(difference, key)) {
            break;
          }
          const list = db.childrenDeep(this.getForeignKeyWhere(key));
          const children = list.map(item => item[this.primaryKey]);
          difference = [].concat(difference, children);
        }
      }
      result = _.difference(result, difference);
      this.checked = result;
      this.$emit('change', result);
    },
    // 请求数据
    async getColumnData(id) {
      try {
        const queyr = { id };
        // 获取数据
        const result = await this.getColumnServer(queyr);
        // 整合数据格式
        const db = new DB('column', [], this.primaryKey, this.foreignKey);
        const list = db.flatten([].concat(result));
        // 处理数据
        if (id && id !== 0) {
          db.insert(list);
          // 维护返回数据中的第一层数据中的 foreignKey 关系
          const where = this.getForeignKeyWhere(0);
          const newData = this.getForeignKeyWhere(id);
          db.update(where, newData);
          return db.select();
        }
        return list;
      } catch (error) {
        return [];
      }
    }
  }
}
</script>