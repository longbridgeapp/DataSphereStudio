<template>
  <div>
    <div class="layout-header">
      <div class="layout-header-menu-icon">
        <div
          style="display:inline-block;"
          v-show="isNavShowMenu"
          @mouseleave="mouseleave"
          @click="mouseover"
        >
          <SvgIcon style="font-size: 28px;" icon-class="menu" color="#00FFFF" />
        </div>
        <div class="logo">
          <img
            @click.stop="goHome"
            class="logo-img"
            :style="{ cursor: isAdmin ? 'pointer' : 'default' }"
            src="../../assets/images/dssLogo5_1.png"
            :alt="$t('message.common.logoName')"
          />
          <span class="version">{{sysVersion}}</span>
        </div>
      </div>
      <!-- <span v-if="showWorkspaceNav && !currentProject.id" class="workspace-icon" :title="$t('message.common.project')">
        <SvgIcon style="font-size: 2rem;" class="icon selected workspace-svg" icon-class="project" color="#ffffff"/>
      </span>-->
      <Dropdown class="project-dro" v-if="showWorkspaceNav && !currentProject.id">
        <div class="project">
          {{ currentWorkspace.name }}
          <Icon type="ios-arrow-down" style="margin-left: 5px"></Icon>
        </div>
        <DropdownMenu slot="list" class="proj-list">
          <div class="proj-name">工作空间列表</div>
          <div class="name-bar">
            <span
              v-for="p in workspaceList"
              :key="p.id"
              :class="{'active':p.id == currentWorkspace.id}"
              class="proj-item"
              @click="changeWorkspace(p)"
            >{{ p.name}}</span>
          </div>
        </DropdownMenu>
      </Dropdown>
      <!-- <span v-if="currentProject.id" class="proj-select" :title="$t('message.common.project')">
        <SvgIcon style="font-size: 2rem;" icon-class="file" color="#1d9cf0"/>
      </span>&nbsp;-->
      <Dropdown class="project-dro" v-if="currentProject.id">
        <div class="project">
          {{ currentProject.name }}
          <Icon type="ios-arrow-down" style="margin-left: 5px"></Icon>
        </div>
        <DropdownMenu slot="list" class="proj-list">
          <div v-for="proj in projectList" :key="proj.id">
            <div class="proj-name">{{ proj.name }}</div>
            <div class="name-bar">
              <span
                v-for="p in proj.dwsProjectList"
                @click="changeProj(proj,p)"
                :key="proj.id+p.id"
                :class="{'active':p.id == currentProject.id}"
                class="proj-item"
              >{{ p.name }}</span>
            </div>
          </div>
        </DropdownMenu>
      </Dropdown>
      <div
        v-clickoutside="handleOutsideClick"
        :class="{'selected': isUserMenuShow}"
        class="user"
        @click="handleUserClick"
      >
        <span>{{ userName || 'Null' }}</span>
        <Icon v-show="!isUserMenuShow" type="ios-arrow-down" class="user-icon" />
        <Icon v-show="isUserMenuShow" type="ios-arrow-up" class="user-icon" />
        <userMenu v-show="isUserMenuShow" @clear-session="clearSession" />
      </div>
      <ul class="menu">
        <li
          v-if="$route.path !== '/newhome' && $route.path !== '/bankhome' && $route.query.workspaceId"
          class="menu-item"
          @click="goSpaceHome"
        >{{$t("message.common.home")}}</li>
        <li
          class="menu-item"
          v-if="homeRoles && $route.query.workspaceId"
          @click="goRolesPath"
        >{{ homeRoles.name }}</li>
        <li
          class="menu-item"
          v-if="$route.query.workspaceId"
          @click="goConsole"
        >{{$t("message.common.management")}}</li>
      </ul>
      <div class="icon-group">
        <Icon
          v-if="isSandbox"
          :title="$t('message.common.home')"
          class="book"
          type="ios-chatboxes"
          @click="linkTo('freedback')"
        ></Icon>
      </div>
    </div>
    <nav-menu
      v-show="isNavMenuShow"
      :menuList="menuList"
      @mouseover.native.capture="mouseover"
      @mouseleave.native="mouseleave"
      @click="handleNavMenuClose"
    ></nav-menu>
  </div>
</template>
<script>
import { isEmpty } from 'lodash'
import storage from '@/common/helper/storage'
import userMenu from './userMenu.vue'
import clickoutside from '@/common/helper/clickoutside'
import navMenu from './navMenu/index.vue'
import mixin from '@/common/service/mixin'
import util from '@/common/util'
import {
  GetBaseInfo,
  GetWorkspaceApplications,
  GetWorkspaceList,
  GetWorkspaceBaseInfo
} from '@/common/service/apiCommonMethod.js'
export default {
  directives: {
    clickoutside
  },
  components: {
    userMenu,
    navMenu
  },
  data() {
    return {
      // 是否开启logo跳转，默认不开启(管理员才开启)
      isAdmin: false,
      // 根据接口返回渲染快捷入口
      homeRoles: null,
      sysVersion: process.env.VUE_APP_VERSION,
      isUserMenuShow: false,
      userName: '',
      isNavMenuShow: false,
      currentProject: {},
      projectList: [],
      isSandbox: process.env.NODE_ENV === 'sandbox',
      workspaceList: [],
      currentWorkspace: {},
      menuList: []
    }
  },
  mixins: [mixin],
  created() {
    this.getWorkspacesRoles()
      .then(res => {
        // cookies改变最新后再执行其他方法
        if (res) {
          this.init()
          this.getApplications()
          this.getWorkspaces()
        }
      })
      .catch(err => {
        console.error(err)
      })
  },
  mounted() {
    this.getCurrentProject()
  },
  computed: {
    isNavShowMenu() {
      return !!this.$route.query.workspaceId
    },
    moudleName() {
      let moudleNameMaps = {
        '/home': 'Scriptis',
        '/console': 'Console',
        '/workflow': 'Workflow',
        '/process': 'Workflow'
      }
      let moudleName
      Object.keys(moudleNameMaps).forEach(k => {
        if (this.$route.path.indexOf(k) > -1) {
          moudleName = moudleNameMaps[k]
        }
      })
      return moudleName
    },
    showWorkspaceNav() {
      return (
        this.$route.path.indexOf('/workspaceHome') !== -1 ||
        this.$route.path === '/project' ||
        this.$route.path === '/workspace'
      )
    }
  },
  watch: {
    '$route.query.projectID'(newValue) {
      this.projectID = newValue
      this.getCurrentProject()
    },
    // 将以前的'$route.query.workspaceId'监听换成'$route'，之前的无法监听组件之间的切换触发
    $route(v) {
      // 设定条件只有切换在工作空间首页时才触发
      if (v.name === 'workspaceHome') {
        this.getWorkspacesRoles()
          .then(res => {
            // cookies改变最新后再执行其他方法
            if (res) {
              this.getApplications()
              this.getWorkspaces()
            }
          })
          .catch(err => {
            console.error(err)
          })
      }
    }
  },
  methods: {
    // 获取进入工作空间的用户权限
    getWorkspacesRoles() {
      // 如果下面接口不通时，默认显示用户名
      if (storage.get('baseInfo', 'local')) {
        this.userName = storage.get('baseInfo', 'local').username
      }
      return new Promise((resolve, reject) => {
        if (this.$route.query.workspaceId) {
          GetWorkspaceBaseInfo({
            workspaceId: this.$route.query.workspaceId || ''
          })
            .then(res => {
              // 缓存数据，供其他页面判断使用
              storage.set(`workspaceRoles`, res.roles, 'session')
              // 获取顶部的快捷入口
              this.homeRoles = {
                name: res.topName,
                path: res.topUrl,
                id: res.workspaceId
              }
              // 同步改变cookies在请求中的附带
              resolve(true)
              // 改变了cookies通知其他地方
              this.$router.app.$emit('getChangeCookies')
            })
            .catch(err => {
              reject(err)
            })
        } else {
          resolve(false)
        }
      })
    },
    isShowWorkspaceMenu() {
      return (
        !this.currentProject.id &&
        (this.$route.query && this.$route.query.workspaceId) &&
        this.$route.path.indexOf('workflow') === -1
      )
    },
    getApplications() {
      if (this.$route.query.workspaceId) {
        GetWorkspaceApplications(this.$route.query.workspaceId).then(data => {
          this.menuList = data.applications || []
        })
      }
    },
    getWorkspaces() {
      GetWorkspaceList({}, 'get')
        .then(res => {
          this.workspaceList = res.workspaces
          this.getCurrentWorkspace()
        })
        .catch(() => {})
    },
    init() {
      GetBaseInfo().then(rst => {
        if (!isEmpty(rst)) {
          this.userName = rst.username
          // 根据权限来判断是否开启logo跳转(管理员才会开启)
          this.isAdmin = rst.isAdmin
          storage.set('baseInfo', rst, 'local')
          this.$router.app.$emit('username', rst.username)
          this.$emit('set-init')
        }
      })
    },
    goto(name) {
      this.$router.push({
        name: name
      })
    },
    getClass(path) {
      let arr = []
      if (this.$route.name === path || this.$route.meta.parent === path) {
        arr.push('selected')
      }
      return arr
    },
    handleOutsideClick() {
      this.isUserMenuShow = false
    },
    handleUserClick() {
      this.isUserMenuShow = !this.isUserMenuShow
    },
    clearSession() {
      this.$emit('clear-session')
    },
    handleNavMenuClose() {
      this.isNavMenuShow = false
    },
    mouseover() {
      this.isNavMenuShow =
        this.$route.path != '/newhome' && this.$route.query.workspaceId
    },
    mouseleave() {
      this.isNavMenuShow = false
    },
    getCurrentWorkspace() {
      let workspaceId = +this.$route.query.workspaceId
      if (workspaceId) {
        this.workspaceList.forEach(item => {
          if (item.id === workspaceId) {
            this.currentWorkspace = item
            storage.set('currentWorkspace', item)
          }
        })
      }
    },
    getCurrentProject() {
      let projList = storage.get('projectList', 'local')
      this.projectList = projList
      let projId = this.$route.query.projectID
      let proj = {}
      if (projId && this.projectList) {
        this.projectList.forEach(item => {
          if (item.dwsProjectList) {
            item.dwsProjectList.forEach(p => {
              if (p.id == projId) {
                proj = { ...p }
              }
            })
          }
        })
      }
      this.currentProject = proj
    },
    // 切换工作空间
    changeWorkspace(workspace) {
      if (workspace.id === this.currentWorkspace.id) return
      // 得考虑在流程图页面和知画的情况, 在此情况下跳转到工程页
      if (['/process'].includes(this.$route.path)) {
        this.$router.replace({ path: '/workspace' })
      } else {
        this.$router.replace({
          path: this.$route.path,
          query: {
            ...this.$route.query,
            workspaceId: workspace.id
          }
        })
      }
    },
    changeProj(proj, p) {
      if (
        p.id == this.currentProject.id &&
        proj.id == this.$route.query.projectTaxonomyID
      )
        return
      // 得考虑在流程图页面和知画的情况, 在此情况下跳转到工程页
      if (['/process'].includes(this.$route.path)) {
        this.$router.replace({ path: '/project' })
      } else {
        this.$router.replace({
          path: this.$route.path,
          query: {
            ...this.$route.query,
            projectTaxonomyID: proj.id,
            projectID: p.id,
            projectName: p.name,
            notPublish: p.notPublish
          }
        })
      }
    },
    goHome() {
      if (this.isAdmin) this.$router.push('/newhome')
    },
    linkTo(type) {
      let url = ''
      if (type === 'book') {
        url = `https://github.com/WeBankFinTech/DataSphereStudio/blob/master/docs/zh_CN/ch3/DSS_User_Manual.md`
      } else if (type === 'github') {
        url = `https://github.com/WeBankFinTech/DataSphereStudio`
      } else if (type === 'freedback') {
        url = 'https://wj.qq.com/s2/4943071/c037/ '
        if (localStorage.getItem('locale') === 'en') {
          url = 'https://wj.qq.com/s2/4943706/5a8b'
        }
      }
      util.windowOpen(url)
    },
    goSpaceHome() {
      let workspaceId = this.$route.query.workspaceId
      if (!workspaceId) return this.goHome()
      this.$router.push({ path: '/workspaceHome', query: { workspaceId } })
      this.currentProject = {}
    },
    goConsole() {
      const url =
        location.origin + '/dss/linkis?noHeader=1&noFooter=1#/console' 
      this.$router.push({
        name: 'commonIframe',
        query: {
          workspaceId: this.$route.query.workspaceId,
          url
        }
      })
    },
    goRolesPath() {
      // 根据接口getWorkspaceBaseInfo渲染跳转不同路径
      this.$router.push({
        path: this.homeRoles.path,
        query: Object.assign({}, this.$route.query)
      })
    }
  }
}
</script>
<style lang="scss" scoped src="./index.scss"></style>
