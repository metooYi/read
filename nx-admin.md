# nx-admin 一个Vue自动化管理系统

## 简介

nx-admin是一个后台管理系统，基于vue和element，内置了很多具体的业务需求，包括有i18国际化解决方案、动态权限路由、图表数据可视化、富文本编辑器等，对我来说这是一套后台管理系统的模板比较合适，或许怎么使用这个库是其次的，我们主要来看他的源码，这就像 **”鱼“** 和 **”渔“** 的道理，只要明白的怎么实现该系统的原理，一通百通。

- [github地址](https://github.com/mgbq/nx-admin)
- [线上演示地址](https://mgbq.github.io/vue-permission)

## 计划&目标

nx-admin提供了很多的功能，最最值得学习的我认为有以下几个：

- 动态菜单(权限控制)
- vuex
- ...

## 开始

### 目录结构

|-- build		(webpack打包配置)
|-- config	(webpack运行打包配置)
|-- github
|-- src		核心
​	|-- api	项目所有网络请求
​	|-- assets 静态文件
​	|-- components	封装的组件(比较细小的组件：backToTop按钮、全屏按钮...)
​	|-- config	项目配置(开发环境，生产环境...)
​	|-- const		网站全局数据
​	|-- directive
​	|-- filters	vue过滤器
​	|-- global	全局主题样式相关
​	|-- icons 	导出所有svg图标
​	|-- lang 		国际化相关
​	|-- mock		模拟网络请求(main.js入口文件中引入，自动拦截Ajax请求，返回数据)
​	|-- router 	路由相关
​	|-- store		vuex状态管理
​	|-- style 		scss样式
​	|-- utils 		提取出来的工具方法
​	|-- vendor	excel表格、zip压缩文件功能相关
​	|-- views		大模块组件，我们所看到的页面
|-- static		静态文件

### 动态路由(addRouters)

核心文件

- src/router/index.js	基础文件，所有页面路由信息
- src/permission.js	   路由拦截
- src/store/modules/permission.js    路由表保存

#### 基础文件 /router/index.js

```javascript
// index.js
export const constantRouterMap = [
    // 不需要权限就可以访问的页面的路由信息
]

export default new Router({
    routers: constantRouterMap // 直接挂载不需要权限的页面
})

export const asyncRouterMap = [
    // 存放需要权限访问页面，路由元信息 meta 对象中的 roles 表示可以访问本页面的角色
]
```

#### 路由拦截 src/permission.js

这个文件我们看他的结构，主要是 router.beforeEach() 路由跳转之前做的一些条件判断

```javascript
router.beforeEach((to, from, next) => {
    // 路由跳转即触发，第一次进入网站，也就是登录页面也会触发，即这时候不存在token
    // 用户登录成功之后，commit提交状态，保存token，再跳转：this.$router.push({ path: '/dashboard/dashboard' })，再次触发beforeEach方法，这时有token，进入if
    if (hasToKen) {
        // 有token 已经登录
        // token是保存在cookie里面的，用户登录之后会保存token在cookie和vuex里面
        if (isLock) {
            // 页面已经锁住，无论用户想跳转到哪个页面都重定向到lock页面
        } else if (toLogin) {
            // 已经登录再跳转到login，重定向到首页
        } else {
            // 跳转到正常页面，首次登录跳转到首页，这时vuex中还没有用户信息
            if (roles === 0) {
                // vuex中没有用户信息
                // 通过dispatch方法触发GetInfo获取用户信息
                // 保存用户信息 commit('SET_ROLES', data.roles)
                // 获取用户角色之后，再次使用dispatch方法调用GenerateRoutes方法生成对应用户角色的路由表，保存再vuex中，此时首页还没有渲染，也就是侧边菜单栏还没有出来，有了路由表，我们就可以动态渲染出侧边菜单栏了
                // 根据用户角色，调用addRouters方法，加入用户可访问的页面
            } else {
                // 动态改变权限的附加需求...
                // 切换admin，editor，路由跳转的时候需要检测router.mate.roles 与用户角色是否匹配，如果不匹配就跳转到401页面，无访问权限
            }
        }
    } else {
        // 没有token
        if (inWhiteList) {
            // 在不重定向白名单里面，即不用登录也能访问页面的路由path
        } else {
            // 不在白名单里面，没有登录不能访问，跳转到login页面，让用户登录
        }
    }
})
```

路由拦截逻辑结构大概就是这样，其中最核心的是生成路由表的方法GenerateRoutes，这个方法在vuex保存路由表的逻辑里面，也就是下一节...

#### 路由表生成和保存 store/modules/permission.js

这个文件，我们按顺序解读

```javascript
// 首先，引入了基础文件，也就是所有的路由信息，游客页面和权限页面都有
import { asyncRouterMap, constantRouterMap } from '@/router' // 变量结构赋值
```

声明两个方法

- hasPermission 判断该具体的一项路由规则是否匹配role，如果匹配放回true，否则返回false；**注意：** 如果这项路由规则没有meta 路由元信息，也返回true；这是防止在一个一级菜单下面有多个二级菜单，这些二级菜单有的是权限页面有的不是，不是权限页面的就不带meta 或者 meta.roles = undefind
- filterAsyncRouter 递归过滤异步路由表，返回符合用户角色权限的路由表；该方法接收两个参数，路由表(数组)、用户角色(数组)，遍历路由表，调用hasPermission方法，把遍历的路由项和用户角色数组传进去，如果路由项包含子路由，递归调用，过滤完成就生成了权限路由表；(真香警告！！！算法真好用...)

```javascript
// 这里把hasPermission 方法和filterAsyncRouter 方法整合，更好理解，当然代码不美观...
function filterAsyncRouter (routerArray, roles) {
    // Array.prototype.filter 方法过滤数组，返回符合条件的项组成的新数组，不会改变原数组
    const addRouters = routerArray.filter(route => {
        // 判断当前遍历到的路由项是否有mate 或者 mate.roles
        if (route.mate && route.mate.roles) {
            // 是权限路由，判断用户角色能不能访问
            // Array.prototype.some 方法判断数组中是否有满足条件的项，有一项即以上放回true
            // 循环用户角色，判断是否存在当前路由项可以访问的角色数组中
            if (roles.some(role => { route.mate.roles.indexOf(role) >= 0 })) {
                // 存在，再次判断，当前路由项有没有子路由，递归调用
                if (route.children && route.children.length) {
                    // 存在子路由，把子路由当作第一个参数
                    router.children = filterAsyncRouter (router.children, roles)
                }
                // 忽略上一步判断子路由，确定当前路由项已匹配用户角色，返回true
                return true
            } else {
                // 没有匹配成功，返回false
                return false
            }
        } else {
            // 不是权限路由，可能是一个一级菜单下面二级菜单既有权限页面也有游客页面
            return true
        }
    })
    // 遍历完成，返回得到的权限路由
    return addRouters
}

// permission
{
    state: {
        routers: constantRouterMap; // 用户完整路由表，初始化为不用权限路由项
        addRouters: [] // 根据用户角色异步添加的路由项
    },
    mutations: {
        // commit 修改state；commit('SET_ROUTERS', params)
        SET_ROUTERS: (state, routers) => {
            state.addRouters = routers // 修改异步添加路由项的状态
            // 往完整路由添加权限路由，添加之前完整路由就是不用权限的路由
            state.routers = constantRouterMap.concat(routers)
        }
	},
    actions: {
        // 核心方法
        GenerateRoutes ({ commit }, data) {
            // data就是获取用户信息之后传过来的用户角色数组
            if (admin) {
                // 如果用户是管理员，最大权限，addRouters 就是全部异步路由
                addRouter = asyncRouterMap
            } else {
                // 用户不是管理员，调用filterAsyncRouter ，把全部异步路由项和用户角色数组传进去，得到addRouters
                addRouters = filterAsyncRouter(asyncRouterMap, roles)
            }
            // 最后commit 改变状态，得到完整路由表
            commit('SET_ROUTERS', addRouters)
        }
    }
}
```

#### 权限路由总结

这里过一遍实现流程

1. 定义同步路由表，即登录之后即可访问页面；定义异步路由表，即权限页面
2. 用户登录成功，保存token，跳转首页，获取用户信息，这时页面还未渲染
3. 获取到了用户角色，通过递归算法过滤比对用户角色对应的可以访问的权限页面，得到完整路由表，保存起来，首页再根据路由表渲染菜单

#### 菜单渲染小细节

核心文件 **src/views/layout/components/Sidebar/SidebarItem** 

父组件调用核心组件的时候把路由表传递过来了：

```html
<!-- permission_routers是从vuex中获取的 -->
<sidebar-item :routes="permission_routers"></sidebar-item>
```

页面结构

```html
<!-- 首先判断当前路由项是不是要显示的 -->
<!-- 没有子路由，即对应一个页面 -->
<router-link v-if=''></router-link>

<!-- 有子路由 -->
<el-submenu v-else>
	<template slot='title'>一级菜单标题</template>
    <!-- v-for遍历出二级菜单 -->
    <template v-for=''>
    	<!-- 判断该二级菜单下面是否还有三级菜单，如果有调用组件本身，把该二级菜单用数组包裹再传入组件，可以理解为组件的递归引用 -->
        <sidebar-item v-if='hasChildren'></sidebar-item>
        <router-link v-else></router-link>
    </template>
</el-submenu>
```

