<!-- 通用树形组件 -->
<template>
  <div class="laws-tree" :class="{ 'disabledItem': disabled }">
    <div class="content_wrap">
      <div class="input-wrapper">
        <input type="text" :id="searchId" value=""  placeholder="节点快速查找" class="empty" v-if="filterNode" />
      </div>
      <div class="left">
        <ul :id="treeDivId" class="ztree"></ul>
      </div>
    </div>
    <div id="rMenu">
      <ul>
        <li id="m_add" @click="rightAdd">{{ addTitle }}</li>
        <li id="m_edit" @click="rightEdit" v-if="!isRoot">{{ editTitle }}</li>
        <li id="m_del" @click="rightRemove" v-if="!isRoot">{{ removeTitle }}</li>
      </ul>
    </div>
    <span v-if="loading">{{ loadingTips }}</span>
  </div>
</template>

<!--
  treeReady: 树结构加载完成, 返回值: 无
  treeEdit: 节点编辑, 返回值: treeId, treeNode
  treeClick: 节点点击, 返回值: treeId, treeNode
  treeRemove: 节点移除, 返回值: treeId, treeNode
  treeAdd: 节点新增, 返回值: treeId, treeNode
-->

<script>
import 'ztree/js/jquery.ztree.core'
import 'ztree/js/jquery.ztree.excheck.js'
import 'ztree/js/jquery.ztree.exedit.js'
import 'ztree/js/jquery.ztree.exhide.js'
export default {
  name: 'lawsTree',
  data () {
    return {
      treeId: '', // 当前操作树组件的id
      treeNode: '', // 当前操作的节点
      loading: false, // 树结构加载中
      zTreeObj: '', // 树实例对象
      openNodes: [], // 已经打开的节点
      currentNode: '', // 当前操作节点
      isRoot: false, // 该节点为根节点
      nodeName: '', // 节点快速查找
      zNodeBeforeSearch: [], // 搜索前的节点
      searchResult: [], // 快速查找结果
      originalNode: [] // 原始节点
    }
  },
  methods: {
    /**
       * @description: 树结构渲染
       * @author: chenxiaoxi
       * @date: 2018/10/22 16:44:38
       */
    lawsTreeInit () {
      // 设置title
      this.zNodes.map((node) => {
        node.title = node.oldname || node.name
      })
      if (this.searchResult.length) {
        this.searchNodeLazy($('#' + this.searchId).val())
      } else {
        return new Promise((resolve, reject) => {
          this.$nextTick(() => {
            if (!this.loading) {
              this.loading = true
            }
            let zNodes = JSON.parse(JSON.stringify(this.zNodes))
            if (this.openNodes.length > 0) {
              this.openNodes.map((openNode) => {
                zNodes.map((node) => {
                  if (openNode.id === node.id) {
                    node.open = true
                  }
                })
              })
            }
            $.fn.zTree.init($('#' + this.treeDivId), this.setting, zNodes)
            this.loading = false
            if (this.filterNode) {
              this.fuzzySearch(this.treeDivId, '#' + this.searchId, null, true)
            }
            // 绑定拖拽元素
            if (this.drag) {
              this.MoveTest.bindDom()
              this.MoveTest.updateType()
            }
            resolve()
          })
        })
      }
    },

    /**
       * @description: 右键新增
       * @author: chenxiaoxi
       * @date: 2018/10/23 10:18:57
       */
    rightAdd () {
      $('#rMenu').css({'visibility': 'hidden'})
      this.setOpenNodes()
      this.$emit('treeAdd', this.treeId, this.treeNode)
    },

    /**
       * @description: 右键编辑
       * @author: chenxiaoxi
       * @date: 2018/10/23 10:19:14
       */
    rightEdit () {
      $('#rMenu').css({'visibility': 'hidden'})
      this.setOpenNodes()
      this.$emit('treeEdit', this.treeId, this.treeNode)
    },

    /**
       * @description: 右键删除
       * @author: chenxiaoxi
       * @date: 2018/10/23 10:19:24
       */
    rightRemove () {
      $('#rMenu').css({'visibility': 'hidden'})
      this.setOpenNodes()
      this.$emit('treeRemove', this.treeId, this.treeNode)
    },

    /**
       * @description: 节点移除
       * @author: chenxiaoxi
       * @date: 2018/10/29 14:47:49
       */
    removeNode (treeNode) {
      this.zNodes.map((item, i) => {
        if (treeNode.id === item.id) {
          let zTree = $.fn.zTree.getZTreeObj(this.treeDivId)
          zTree.removeNode(treeNode)
        }
      })
    },

    /**
       * @description: 节点添加
       * @author: chenxiaoxi
       * @date: 2018/10/29 14:53:01
       */
    addNode (treeNode) {
      this.zNodes.push(treeNode)
    },

    /**
       * @description: 节点取消选中
       * @author: chenxiaoxi
       * @date: 2018/10/29 16:09:11
       */
    cancelSelectedNode (treeNode) {
      let zTree = $.fn.zTree.getZTreeObj(this.treeDivId)
      if (treeNode !== '' && treeNode !== undefined) {
        zTree.cancelSelectedNode(treeNode)
      } else {
        zTree.cancelSelectedNode()
      }
    },

    /**
       * @description: 节点选中
       * @author: chenxiaoxi
       * @date: 2018/10/29 16:14:23
       */
    selectNode (treeNode) {
      this.$nextTick(() => {
        if (treeNode !== '' && treeNode !== undefined) {
          this.lawsTreeInit()
        }
      })
    },

    /**
       * @description: 获取所选节点
       * @author: chenxiaoxi
       * @date: 2018/10/29 16:14:23
       */
    getSelectNode (idList) {
      let selectNode = []
      idList.map((id, index) => {
        this.zNodes.map((zNodesItem) => {
          if (zNodesItem.id === id) {
            selectNode.push(zNodesItem)
          }
        })
      })
      return selectNode
    },

    /**
       * @description: 获取被勾选的节点集合
       * @author: chenxiaoxi
       * @date: 2018/11/08 19:47:10
       */
    getCheckNodes () {
      let treeObj = $.fn.zTree.getZTreeObj(this.treeDivId)
      let checkNodes = treeObj.getCheckedNodes(true)
      return checkNodes
    },

    /**
       * @description: 获取所有节点
       * @author: chenxiaoxi
       * @date: 2018/11/08 19:47:10
       */
    transformToArray () {
      let treeObj = $.fn.zTree.getZTreeObj(this.treeDivId)
      let treeNodes = treeObj.transformToArray(treeObj.getNodes())
      return treeNodes
    },

    /**
       * @description: 保存展开的节点
       * @author: chenxiaoxi
       * @date: 2018/11/09 16:29:18
       */
    setOpenNodes () {
      let treeObj = $.fn.zTree.getZTreeObj(this.treeDivId)
      let node = treeObj.getNodes()
      let nodes = treeObj.transformToArray(node)
      let openNodes = nodes.filter((item, index) => {
        return item.open === true
      })
      this.openNodes = openNodes
    },

    /**
       * @description: 获取下一个节点
       * @author: chenxiaoxi
       * @date: 2018/11/10 10:51:48
       */
    getNextNode () {
      let treeObj = $.fn.zTree.getZTreeObj(this.treeDivId)
      let sNodes = treeObj.getSelectedNodes()
      if (sNodes.length > 0) {
        return sNodes[0].getNextNode()
      } else {
        return ''
      }
    },

    /**
       * @description: 选取根节点
       * @author: chenxiaoxi
       * @date: 2018/11/10 12:10:31
       */
    selectRootNode () {
      let treeObj = $.fn.zTree.getZTreeObj(this.treeDivId)
      let nodes = treeObj.getNodes()
      if (nodes.length > 0) {
        treeObj.selectNode(nodes[0])
      }
    },

    /**
       * @description: 节点快速查找
       * @author: chenxiaoxi
       * @date: 2018/11/23 14:44:38
       */
    /* eslint-disable */
      fuzzySearch (zTreeId, searchField, isHighLight, isExpand) {
        const _this = this
        var zTreeObj = $.fn.zTree.getZTreeObj(zTreeId)
        var nameKey = zTreeObj.setting.data.key.name
        isHighLight = isHighLight !== false
        isExpand = !!isExpand
        zTreeObj.setting.view.nameIsHTML = isHighLight

        var metaChar = '[\\[\\]\\\\\^\\$\\.\\|\\?\\*\\+\\(\\)]'
        var rexMeta = new RegExp(metaChar, 'gi')

        // keywords filter function
        function ztreeFilter (zTreeObj, _keywords, callBackFunc) {
          if (!_keywords) {
            _keywords = ''
          }
          function filterFunc (node) {
            if (node && node.oldname && node.oldname.length > 0) {
              node[nameKey] = node.oldname
            }
            zTreeObj.updateNode(node)
            if (_keywords.length === 0) {
              zTreeObj.expandNode(node, false)
              return true
            }
            // transform node name and keywords to lowercase
            // 当前节点与已选中的节点做对比
            function isSplitNode (node) {
              if (!_this.spliceNode) {
                return true
              } else {
                // 默认可以展示,如果找到了,就不展示
                let flag = true
                _this.spliceNode.map((spNode) => {
                  if (spNode.id === node.id) {
                    flag = false
                  }
                })
                return flag
              }
            }
            if (node[nameKey] && node[nameKey].toLowerCase().indexOf(_keywords.toLowerCase()) !== -1 && isSplitNode(node)) {
              if (isHighLight) {
                var newKeywords = _keywords.replace(rexMeta, function (matchStr) {
                  return '\\' + matchStr
                })
                node.oldname = node[nameKey]
                var rexGlobal = new RegExp(newKeywords, 'gi')
                node[nameKey] = node.oldname.replace(rexGlobal, function (originalText) {
                  var highLightText =
                    '<span style="color: whitesmoke;background-color: darkred;">' +
                    originalText +
                    '</span>'
                  return 	highLightText
                })
                zTreeObj.updateNode(node)
              }
              zTreeObj.showNode(node)
              return true
            }
            zTreeObj.hideNode(node)
            return false
          }

          var nodesShow = zTreeObj.getNodesByFilter(filterFunc)
          processShowNodes(nodesShow, _keywords)
        }
        // 展示前处理
        function processShowNodes (nodesShow, _keywords) {
          if (nodesShow && nodesShow.length > 0) {
            if (_keywords.length > 0) {
              _this.searchResult = nodesShow
              $.each(nodesShow, function (n, obj) {
                var pathOfOne = obj.getPath()
                if (pathOfOne && pathOfOne.length > 0) {

                  for (var i = 0; i < pathOfOne.length - 1; i++) {
                    zTreeObj.showNode(pathOfOne[i])
                    zTreeObj.expandNode(pathOfOne[i], true)
                  }
                }
              })
            } else {
              _this.searchResult = []
              // 重新加载所有的节点
              let zNodes = JSON.parse(JSON.stringify(_this.zNodes))
              $.fn.zTree.init($('#' + _this.treeDivId), _this.setting, zNodes)
              let zTree = $.fn.zTree.getZTreeObj(_this.treeDivId);
              // 选中根节点第一个节点
              // if (!_this.spliceNode) {
              //   let nodes = zTree.getNodes()
              //   if (nodes.length>0) {
              //     zTree.selectNode(nodes[0])
              //   }
              // }
            }
          }
        }

        // listen to change in input element
        $(searchField).bind('input propertychange', function() {
          var _keywords = $(this).val();
          searchNodeLazy(_keywords); //call lazy load
        });

        var timeoutId = null
        // excute lazy load once after input change, the last pending task will be cancled
        function searchNodeLazy (_keywords) {
          if (timeoutId) {
            // clear pending task
            clearTimeout(timeoutId)
          }
          timeoutId = setTimeout(function () {
            ztreeFilter(zTreeObj, _keywords) // lazy load ztreeFilter function
            $(searchField).focus()// focus input field again after filtering
          }, 50)
        }
        _this.searchNodeLazy = function (_keywords) {
          return searchNodeLazy(_keywords)
        }
      },
      // 过滤输入框清空
      clearSearch () {
        $('#' + this.searchId).val('')
      }
    },
    components: {},
    props: {
      // 新增按钮title
      addTitle: {
        type: String,
        default: '节点新增'
      },
      // 编辑按钮title
      editTitle: {
        type: String,
        default: '节点维护'
      },
      // 移除按钮title
      removeTitle: {
        type: String,
        default: '节点删除'
      },
      // 是否显示删除按钮
      showRemoveBtn: {
        type: Boolean
      },
      // 显示编辑按钮
      showEditBtn: {
        type: Boolean
      },
      // 树加载中的提示
      loadingTips: {
        type: String,
        default: '节点加载中'
      },
      // 是否显示右键菜单
      showRMenu: {
        type: Boolean,
        default: false
      },
      // 树dom唯一标识
      treeDivId: {
        type: String,
        required: true
      },
      // 是否禁用
      disabled: {
        type: Boolean,
        default: false
      },
      // 是否可进行操作
      editable: {
        type: Boolean,
        default: true
      },
      // 机构是否可被选择
      deptSelect: {
        type: Boolean,
        default: false
      },
      // 树结构加载完成
      treeReady: {
        type: Boolean,
        default: false
      },
      // 自动选择根节点
      autoSelect: {
        type: Boolean,
        default: false
      },
      // 是否启用checkbox
      checkEnable: {
        type: Boolean,
        default: false
      },
      // 新增权限id
      addId: {
        type: String,
        default: ''
      },
      // 编辑权限id
      editId: {
        type: String,
        default: '123'
      },
      // 删除权限id
      removeId: {
        type: String,
        default: ''
      },
      // 是否开启拖拽
      drag: {
        type: Boolean,
        default: false
      },
      // 拖拽绑定区域
      dragBind: {
        type: String,
        required: this.drag
      },
      // 拖拽节点dom
      dragDom: {
        type: String,
        required: this.drag
      },
      // 是否开启节点过滤
      filterNode: {
        type: Boolean,
        default: true
      },
      // 已经选了的节点
      spliceNode: {
        type: Array
      },
      // 树结构
      zNodes: {
        type: Array,
        required: true
      }
    },
    computed: {
      searchId () {
        return this.treeDivId + 'key'
      }
    },
    watch: {
      zNodes: {
        deep: true,
        handler (val, oldVal) {
          this.setOpenNodes()
          this.lawsTreeInit()
          this.$nextTick(() => {
            let zTree = $.fn.zTree.getZTreeObj(this.treeDivId)
            if (this.autoSelect && this.treeNode === '' && !this.spliceNode) {
              // 选中根节点第一个节点
              let nodes = zTree.getNodes()
              if (nodes.length > 0) {
                zTree.selectNode(nodes[0])
              }
            } else {
              // zTree.selectNode(this.treeNode)
              if (!this.treeNode.isParent && this.treeNode !== '') {
                let copyNode = JSON.parse(JSON.stringify(val))
                copyNode.map((node) => {
                  if (node.id === this.treeNode.pId) {
                    node.open = true
                    let setParentNode = function (node) {
                      for (let i = 0; i < copyNode.length; i++) {
                        if (copyNode[i].id === node.pId) {
                          copyNode[i].open = true
                          setParentNode(copyNode[i])
                        }
                      }
                    }
                    setParentNode(node)
                    $.fn.zTree.init($('#' + this.treeDivId), this.setting, copyNode)
                    if (!this.spliceNode && this.autoSelect) {
                      let zTree = $.fn.zTree.getZTreeObj(this.treeDivId)
                      let nodes = zTree.getNodes()
                      zTree.selectNode(nodes[0])
                    }
                  }
                })
              }
              // dom加载完成后取消选中的样式
              this.$nextTick(() => {
                $('#' + this.treeDivId + ' .curSelectedNode').removeClass('curSelectedNode')
              })
            }
          })
          if (this.spliceNode) {
            // 节点添加的操作, 用于流程右移操作, 设置treeNode, 在右移后立刻可以看到移动的节点
            for (let i = 0; i < val.length; i ++) {
              let flag = false
              for (let j = 0; j < oldVal.legnth; j ++) {
                if (val[i].id === oldVal[i].id) {
                  flag = true
                }
              }
              if (!flag) {
                this.treeNode = val[i]
              }
            }
          }
        }
      }
    },
    mounted () {
      let _this = this
      this.loading = true
      this.lawsTreeInit()
      var MoveTest = {
        errorMsg: '放错了...请选择正确的类别！',
        curTarget: null,
        curTmpTarget: null,
        dragId: null,
        noSel: function () {
          try {
            window.getSelection ? window.getSelection().removeAllRanges() : document.selection.empty()
          } catch (e) {}
        },
        dragTree2Dom: function (treeId, treeNodes) {
          return !treeNodes
        },
        prevTree: function (treeId, treeNodes, targetNode) {
          return !targetNode.isParent && targetNode.parentTId === treeNodes[0].parentTId
        },
        nextTree: function (treeId, treeNodes, targetNode) {
          return !targetNode.isParent && targetNode.parentTId === treeNodes[0].parentTId
        },
        innerTree: function (treeId, treeNodes, targetNode) {
          return targetNode != null && targetNode.isParent && targetNode.tId === treeNodes[0].parentTId
        },
        dragMove: function (e, treeId, treeNodes) {
          let p = null
          let pId = 'dom_' + treeNodes[0].pId
          if (e.target.id === pId) {
            p = $(e.target)
          } else {
            p = $(e.target).parent('#' + pId)
            if (!p.get(0)) {
              p = null
            }
          }

          $('.domBtnDiv .active').removeClass('active')
          if (p) {
            p.addClass('active')
          }
        },
        dropTree2Dom: function (e, treeId, treeNodes, targetNode, moveType) {
          var domId = 'dom_' + treeNodes[0].getParentNode().id
          if (moveType === null && (domId === e.target.id || $(e.target).parents('#' + domId).length > 0)) {
            var zTree = $.fn.zTree.getZTreeObj('treeDemo')
            zTree.removeNode(treeNodes[0])

            var newDom = $('span[domId=' + treeNodes[0].id + ']')
            if (newDom.length > 0) {
              newDom.removeClass('domBtn_Disabled')
              // newDom.addClass('domBtn')
            } else {
              $('#' + domId).append("<span class='domBtn' domId='" + treeNodes[0].id + "'>" + treeNodes[0].name + '</span>')
            }
            MoveTest.updateType()
          } else if ($(e.target).parents('.domBtnDiv').length > 0) {
            // alert(MoveTest.errorMsg)
          }
        },
        dom2Tree: function (e, treeId, treeNode) {
          if (MoveTest.dragId !== null) {
            let dragId = []
            dragId.push(MoveTest.dragId)
            _this.$emit('drag', treeNode, dragId)
            MoveTest.dragId = null
          }
        },
        updateType: function () {
          const zTree = $.fn.zTree.getZTreeObj(_this.treeDivId)
          let nodes = zTree.getNodes()
          for (var i = 0, l = nodes.length; i < l; i++) {
            var num = nodes[i].children ? nodes[i].children.length : 0
            nodes[i].name = nodes[i].name.replace(/ \(.*\)/gi, '') + ' (' + num + ')'
            zTree.updateNode(nodes[i])
          }
        },
        bindDom: function () {
          $(_this.dragBind).bind('mousedown', MoveTest.bindMouseDown)
        },
        bindMouseDown: function (e) {
          var target = e.target
          // 阻止元素触发事件
          const prevent = ['input', 'a', 'span', 'li']
          const targetTagName = target.localName
          if (prevent.indexOf(targetTagName) !== -1) { return }
          if (target !== null && $(target).parents('tr').hasClass(_this.dragDom)) {
            var doc = $(document)
            target = $(target).parents('tr')
            let className = target[0].className
            let id = className.substring(className.indexOf('id=') + 3)
            id = id.substring(0, id.indexOf(' '))
            MoveTest.dragId = id
            let docScrollTop = doc.scrollTop()
            let docScrollLeft = doc.scrollLeft()
            // target.addClass('domBtn_Disabled')
            target.removeClass(_this.dragDom)
            const curDom = $("<span class='dom_tmp domBtn iconfont'>&#xe64b;</span>")
            curDom.appendTo('body')

            curDom.css({
              'top': (e.clientY + docScrollTop + 3) + 'px',
              'left': (e.clientX + docScrollLeft + 3) + 'px'
            })
            MoveTest.curTarget = target
            MoveTest.curTmpTarget = curDom

            doc.bind('mousemove', MoveTest.bindMouseMove)
            doc.bind('mouseup', MoveTest.bindMouseUp)
            doc.bind('selectstart', MoveTest.docSelect)
          }
          if (e.preventDefault) {
            e.preventDefault()
          }
        },
        bindMouseMove: function (e) {
          MoveTest.noSel()
          const doc = $(document)
          let docScrollTop = doc.scrollTop()
          let docScrollLeft = doc.scrollLeft()
          let tmpTarget = MoveTest.curTmpTarget
          if (tmpTarget) {
            tmpTarget.css({
              'top': (e.clientY + docScrollTop + 3) + 'px',
              'left': (e.clientX + docScrollLeft + 3) + 'px'
            })
          }
          return false
        },
        bindMouseUp: function (e) {
          var doc = $(document)
          doc.unbind('mousemove', MoveTest.bindMouseMove)
          doc.unbind('mouseup', MoveTest.bindMouseUp)
          doc.unbind('selectstart', MoveTest.docSelect)

          let target = MoveTest.curTarget
          let tmpTarget = MoveTest.curTmpTarget
          if (tmpTarget) tmpTarget.remove()

          if ($(e.target).parents(_this.treeDivId).length === 0) {
            if (target) {
              target.removeClass('domBtn_Disabled')
            }
            MoveTest.curTarget = null
            MoveTest.curTmpTarget = null
          }
        },
        bindSelect: function () {
          return false
        }
      }
      this.MoveTest = MoveTest
      var setting = {
        view: {
          addHoverDom: addHoverDom,
          removeHoverDom: removeHoverDom,
          selectedMulti: false,
          dblClickExpand: false,
          nameIsHTML: true // 允许name支持html
        },
        edit: {
          enable: _this.editable,
          editNameSelectAll: true,
          showRemoveBtn: _this.showRemoveBtn || showRemoveBtn,
          showRenameBtn: _this.showEditBtn || showRenameBtn,
          renameTitle: _this.editTitle,
          removeTitle: _this.removeTitle,
          drag: {
            prev: false,
            next: true,
            inner: true
          }
        },
        data: {
          simpleData: {
            enable: true
          },
          key: {
            title: "title"
          }
        },
        callback: {
          beforeEditName: beforeEditName,
          beforeRemove: beforeRemove,
          beforeClick: beforeClick,
          beforeDblClick: beforeDblClick,
          onClick: handleClick,
          onDblClick: handleDblClick,
          onRightClick: this.showRMenu ? OnRightClick : '',
          beforeDrag: MoveTest.dragTree2Dom,
          onDrop: MoveTest.dropTree2Dom,
          onDragMove: MoveTest.dragMove,
          onMouseUp: MoveTest.dom2Tree
        },
        check: {
          enable: this.checkEnable,
          chkboxType: {
            'Y': 's', 'N': 's'
          }
        }
      }
      this.setting = setting
      let rMenu = $('#rMenu')
      let className = 'dark'
      function beforeEditName (treeId, treeNode) {
        className = (className === 'dark' ? '' : 'dark')
        let zTree = $.fn.zTree.getZTreeObj(_this.treeDivId)
        zTree.selectNode(treeNode)
        this.setOpenNodes()
        _this.$emit('treeEdit', treeId, treeNode)
        return false
      }

      /**
       * @description: 树节点点击
       * @author: chenxiaoxi
       * @date: 2018-09-25 16:13:25
       */
      function handleClick (event, treeId, treeNode) {
        let treeNodeOld = JSON.parse(JSON.stringify(treeNode))
        if (treeNodeOld.oldname) {
          treeNodeOld.name = treeNodeOld.oldname
        }
        _this.treeNode = treeNode
        _this.$emit('treeClick', treeId, treeNodeOld)
      }

      /**
       * @description: 树节点双击
       * @author: chenxiaoxi
       * @date: 2018/10/24 15:38:14
       */
      function handleDblClick (event, treeId, treeNode) {
        let dblTreeNode = JSON.parse(JSON.stringify(treeNode))
        if (dblTreeNode.oldname) {
          dblTreeNode.name = dblTreeNode.oldname
        }
        _this.$emit('treeDblClick', treeId, dblTreeNode)
      }

      /**
       * @description: 树节点删除
       * @author: chenxiaoxi
       * @date: 2018-09-25 14:05:25
       */
      function beforeRemove (treeId, treeNode) {
        className = (className === 'dark' ? '' : 'dark')
        let zTree = $.fn.zTree.getZTreeObj(_this.treeDivId)
        zTree.selectNode(treeNode)
        this.setOpenNodes()
        _this.$emit('treeRemove', treeId, treeNode)
        return false
      }

      /**
       * @description: 判断哪些节点可被选中
       * @author: chenxiaoxi
       * @date: 2018/10/12 09:51:18
       */
      function beforeClick (treeId, treeNode, clickFlag) {
        return (!treeNode.isParent && !_this.disabled) || (_this.deptSelect && !_this.disabled)
      }

      /**
       * @description: 判断哪些节点可被选中
       * @author: chenxiaoxi
       * @date: 2018/10/25 10:12:39
       */
      function beforeDblClick (treeId, treeNode) {
        if (treeNode.isParent !== undefined && treeNode.isParent !== null && treeNode.isParent !== 'null') {
          return !treeNode.isParent || _this.deptSelect
        } else {
          return _this.deptSelect
        }
      }
      // 是否显示移除按钮
      function showRemoveBtn (treeId, treeNode) {
        return treeNode.pId !== null && !_this.showRMenu && !_this.disabled && _this.editable
      }
      function showRenameBtn (treeId, treeNode) {
        return treeNode.pId !== null && !_this.showRMenu && !_this.disabled && _this.editable
      }
      function addHoverDom (treeId, treeNode) {
        if (_this.showRMenu || _this.disabled || !_this.editable) { return }
        var sObj = $('#' + treeNode.tId + '_span')
        if (treeNode.editNameFlag || $('#addBtn_' + treeNode.tId).length > 0 || treeNode.id === '404' || treeNode.name === '未分配人员') return
        var addStr = "<span class='button add' id='addBtn_" + treeNode.tId +
          "' title='节点新增' onfocus='this.blur();'></span>"
        sObj.after(addStr)
        var btn = $('#addBtn_' + treeNode.tId)
        if (btn) {
          btn.bind('click', function () {
            className = (className === 'dark' ? '' : 'dark')
            var zTree = $.fn.zTree.getZTreeObj(_this.treeDivId)
            this.zTree = zTree
            zTree.selectNode(treeNode)
            _this.setOpenNodes()
            _this.$emit('treeAdd', treeId, treeNode)
            return false
          })
        }
      };
      function removeHoverDom (treeId, treeNode) {
        $('#addBtn_' + treeNode.tId).unbind().remove()
      }
      function OnRightClick (event, treeId, treeNode) {
        var zTree = $.fn.zTree.getZTreeObj(_this.treeDivId)
        // 把treeId, treeNode存储到data中
        _this.treeId = treeId
        _this.treeNode = treeNode
        if (!treeNode && event.target.tagName.toLowerCase() !== 'button' && $(event.target).parents('a').length === 0) {
          zTree.cancelSelectedNode()
          showRMenu('root', event.clientX, event.clientY, treeId, treeNode)
          _this.$emit('treeRightClick', treeId, treeNode)
        } else if (treeNode && !treeNode.noR) {
          zTree.selectNode(treeNode)
          showRMenu('node', event.clientX, event.clientY, treeId, treeNode)
          _this.$emit('treeRightClick', treeId, treeNode)
        }
      }

      function showRMenu (type, x, y, treeId, treeNode) {
        if (treeNode.pId !== null && _this.showRMenu && !_this.disabled && _this.editable) {
          _this.isRoot = false
          $('#rMenu ul').show()
          if (type === 'root') {
            $('#m_add').hide()
            $('#m_edit').hide()
            $('#m_del').hide()
          } else {
            $('#m_add').show()
            $('#m_edit').show()
            $('#m_del').show()
          }
        } else {
          _this.isRoot = true
        }
        y += document.body.scrollTop
        x += document.body.scrollLeft
        rMenu.css({'top': y + 'px', 'left': x + 'px', 'visibility': 'visible'})
        $('body').bind('mousedown', onBodyMouseDown)
      }
      /* eslint-disable */
      function hideRMenu () {
        if (rMenu) rMenu.css({'visibility': 'hidden'})
        $('body').unbind('mousedown', onBodyMouseDown)
      }$
      /* eslint-disable */
      function onBodyMouseDown (event) {
        if (!(event.target.id === 'rMenu' || $(event.target).parents('#rMenu').length > 0)) {
          rMenu.css({'visibility': 'hidden'})
        }
      }
    }
  }
</script>

<style lang="less">
  @import 'ztree/css/zTreeStyle/zTreeStyle.css';
  .laws-tree{
    position: relative;
    width: 100%;
    height: 100%;
    .content_wrap{
      width: 100%;
      height: 100%;
      .input-wrapper{
        width: 100%;
        input{
          display: inline-block;
          width: 100%;
          height: 32px;
          line-height: 1.5;
          padding: 4px 7px;
          font-size: 12px;
          border-radius: 4px;
          color: #515a6e;
          background-color: #fff;
          background-image: none;
          position: relative;
          cursor: text;
          font-family: inherit;
          border: 1px solid #DDD;
          box-sizing: border-box;
        }
      }
    }
    .ztree li span.button.add {margin-left:2px; margin-right: -1px; background-position:-144px 0; vertical-align:top; *vertical-align:middle}
    div#rMenu {position:absolute; visibility:hidden; top:0;text-align: left;padding: 2px;}
    div#rMenu ul {
      li{
        padding: 0 15px;
        cursor: pointer;
        list-style: none;
        border: 1px solid #888;
        border-bottom: 1px solid transparent;
        background: #FFEBC0;
        &:last-child{
          border-bottom: 1px solid #888;
        }
        &:hover{
          background: #FFC774;
        }
      }
    }
    #rMenu{
      position: fixed !important;
    }
    .ztree li span.button.edit:hover{
      background-position: -110px -48px;
    }
    .ztree li span.button.remove:hover{
      background-position: -110px -64px !important;
    }
    .left{
      width: 100%;
      height: calc(~'100% - 32px');
      overflow-y: auto;
      .ztree{
        width: 100% !important;
        height: 100% !important;
        box-sizing: border-box;
      }
    }
  }
  .dom_line {margin:2px;border-bottom:1px gray dotted;height:1px}
  .domBtnDiv {display:block;padding:2px;border:1px gray dotted;background-color:powderblue}
  .categoryDiv {display:inline-block; width:335px}
  .domBtn {display:inline-block;cursor:pointer}
  .domBtn_Disabled {background-color:#DFDFDF;color:#999999}
  .dom_tmp {position:absolute;font-size:50px;}
</style>
