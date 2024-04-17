[TOC]



# 实战：智能 OJ (在线判题系统)

## 判题系统相关概念

## 架构设计

## 核心业务流程

## 表结构设计

> 若是对相关表需要有哪些字段不是很了解，可以参考市面上已有的系统来进行表结构的设计，或者也可使用 al 工具进行设计

### 一、题目表

- 题目标题：title

- 题目内容：content

- 题目标签：tags

  - 例如：难易度标签、题目属于哪一类数据结构的标签等
  - 使用 json 数组字符串进行存储，即将 json 数组转化为字符串存储在数据库

- 题目标准答案：answer

- 题目提交数：submitNum

- 题目通过数：acceptedNum

- 判题用例：judgeCase

  ```json
  {
  	input:"1 2",
  	output:"3 4"
  },
  {
  	input:"1 2",
  	output:"3 4"
  }
  ```

- 判题配置：judgeConfig

  - timeLimit：时间限制（ms）
  - memoryLimit：内存限制（kb）
- stackLimit：堆栈限制
  
  ```
  {
  	timeLimit:"1",
  	memoryLimit:"3",
  	stackLimit:"1"
  }
  ```

```sql
-- 题目表
create table if not exists question
(
    id              bigint auto_increment comment 'id' primary key,
    title           varchar(512)                       null comment '题目标题',
    content         text                               null comment '题目内容',
    tags            varchar(1024)                      null comment '题目标签列表（json 数组）',
    answer          text                               null comment '题目答案',
    submitNum       int      default 0                 not null comment '题目提交数',
    acceptedNum     int      default 0                 not null comment '题目通过数',
    judgeCase       text                               null comment '判题用例',
    judgeConfig     text                               null comment '判题配置',
    thumbNum        int      default 0                 not null comment '点赞数',
    favourNum       int      default 0                 not null comment '收藏数',
    creatorId       bigint                             not null comment '创建用户 id',
    createTime      datetime default CURRENT_TIMESTAMP not null comment '创建时间',
    updateTime      datetime default CURRENT_TIMESTAMP not null on update CURRENT_TIMESTAMP comment '更新时间',
    isDelete        tinyint  default 0                 not null comment '是否删除',
    index idx_creatorId (creatorId)
) comment '题目' collate = utf8mb4_unicode_ci;
```

### 二、题目提交表

> 用户提交的题目的资料

- 用户 id ：userId

- 题目 id：questionId

- 提交的编程语言：language

- 提交的代码：code 

- 题目提交后的判题状态：status

- 判题信息：judgeInfo

  ```json
  {
  	"message":"题目执行信息",
  	"time":"执行所花费的时间（ms）",
  	"memory":"执行所花费的内存（kb）"
  }
  ```

  > `message` 的枚举值
  >
  > - Accepted  成功
  > - Wrong Answer 答案错误
  > - Compile Error 编译错误
  > - memory limit exceeded 内存溢出
  > - time limit exceeded 超时
  > - presentation error 展示错误
  > - output limit exceeded 输出溢出
  > - waiting 等待中
  > - dangerous operation 危险操作
  > - runtime error 运行错误（用户程序运行出错）
  > - system error 系统错误（判题系统出错）

```sql
-- 题目提交表
create table if not exists question_submit
(
    id         bigint auto_increment comment 'id' primary key,
    questionId bigint                             not null comment '题目 id',
    language   varchar(256)                       not null comment '编程语言',
    code       text                               not null comment '提交代码',
    status     int      default 0                 not null comment '判题状态（0 - 待判题、1 - 判题中、2 - 成功、3 - 失败）',
    judgeInfo  text                               null comment '判题信息（存 json 字串）',
    userId     bigint                             not null comment '提交用户 id',
    createTime datetime default CURRENT_TIMESTAMP not null comment '创建时间',
    updateTime datetime default CURRENT_TIMESTAMP not null on update CURRENT_TIMESTAMP comment '更新时间',
    isDelete   tinyint  default 0                 not null comment '是否删除',
    index idx_questionId (questionId),
    index idx_userId (userId)
) comment '题目提交表';
```

### 三、什么情况下适合加索引，如何选择给那个字段加索引

> 首先从业务出发，无论是单个索引还是联合索引，都要从实际的业务查询语句、字段枚举值的区分度、字段的类型考虑（where 条件指定的字段）
>
> 原则上：
>
> - 能不用索引就不用索引
> - 能用单个索引就不用联合索引或多个索引
> - 不给区分度低的字段加索引（如：性别）
>
> 因为索引也是占用空间的



## 技术选型

- 前端
  - TypeScript
  - vueCli
  - arco.design
  - vuex

## 项目初始化

### 一、 前端项目初始化

- node 版本：18.12.0
- npm 版本：9.5.1

- 安装 vue-cli 脚手架

  ```baash
  npm install -g @vue/cli 
  ```

  - @vue/cli 5.0.8 
    - typescript
    - eslint
    - prettier

- 创建 vue 项目

  ```bash
  vue create 项目名
  ```

- IDE：webstorm

  - 前端工程化配置

    - 代码美化

      ![image-20231026230021531](.\README.assets\image-20231026230021531.png)

    - 语法校验

      ![image-20231026230336637](.\README.assets\image-20231026230336637.png)

- 引入组件 

  - [arco.design]: https://arco.design/vue	"组件库"

  - 安装

    ```bash
    npm install --save-dev @arco-design/web-vue
    ```


#### 1. 前端通用布局

1. 创建 layouts 目录 ，创建 BasicLayout.vue 文件

   - 方便多个布局的切换

   - 此后内容可在 BasicLayout.vue 中编写

#### 2. 动态路由跳转

- 步骤

  1. 提取通用路由文件 （route.ts）

  2. 菜单文件读取路由

     > export 导出的对象在 import 时需要 {} 进行解构导入
     >
     > export default 导出则不需要

  3. 绑定菜单组件跳转事件

  4. 同步路由到菜单项

     > 点击菜单项 => 跳转更新路由 => 跳转路由后同步更新菜单项被选中状态

     ```typescript
     <script setup lang="ts">
     import { routes } from "../router/routes";
     import { useRouter } from "vue-router";
     import { ref } from "vue";
     
     //默认选中的菜单
     const selectedKeys = ref(["/"]);
     
     //点击菜单项进行路由跳转
     const router = useRouter();
     const clickMenuItem = (key: string) => {
       router.push({ path: key });
     };
     
     //路由跳转后更新菜单选中状态
     router.afterEach((to) => {
       selectedKeys.value = [to.path];
     });
     </script>
     ```

#### 3. 使用 VUEX 进行全局状态管理

- 像用户数据此类数据可用于全局管理

#### 4. 页面菜单路由跳转权限管理

- 路由中的 meta 属性可用于保存权限信息

- 在路由跳转之前验证用户是否可路由该页面

  ```typescript
  import { useRouter } from "vue-router";
  import { useStore } from "vuex";
  
  const store = useStore();
  const router = useRouter();
  router.beforeEach((to, from, next) => {
    if (
      to.meta.access === "adminAccess" &&
      store.state.user?.loginUserInfo?.role !== "admin"
    ) {
      next("/no-auth");
    }
    next();
  });
  ```


#### 5. 隐藏需要权限的路由菜单

> 在v-for 中不要使用 v-if 去筛选需要的数据，这样会先循环所有的数据再进行判断，导致性能浪费
>
> 解决方案：可以先在 js/ts 中过滤出需要数据再使用 v-for 渲染数据

```typescript
const visibleRoutes = routes.filter((item) => {
  return !item.meta?.hiddenInMenu;
});
```

```html
        <a-menu-item v-for="item in visibleRoutes" :key="item.path"
          >{{ item.name }}
        </a-menu-item>
```

#### 6. 根据检查用户权限和路由所需要的权限判断是否需要隐藏路由菜单

1. 编写全局的检查用户权限的函数

   ```typescript
   import PERMISSION_ENUM from "@/access/permissionEnum";
   
   /**
    * 全局检查用户权限函数（根据需要的权限判断用户是否有权限）
    * @param user 用户信息
    * @param needPermission 需要的权限
    * @return boolean
    */
   const checkPermission = (
     user: any,
     needPermission: string = PERMISSION_ENUM.NOT_LOGIN
   ) => {
     // 获取用户权限
     const userPermission = user?.userRole ?? PERMISSION_ENUM.NOT_LOGIN;
     // 用户未登录但是需要的是 user 权限
     if (
       needPermission === PERMISSION_ENUM.USER &&
       userPermission === PERMISSION_ENUM.NOT_LOGIN
     ) {
       return false;
     }
     // 用户需要的是 admin 权限，但是用户不是 admin
     if (
       needPermission === PERMISSION_ENUM.ADMIN &&
       userPermission !== PERMISSION_ENUM.ADMIN
     ) {
       return false;
     }
     return true;
   };
   
   export default checkPermission;
   
   ```

2. 判断用户是否有权限显示菜单

   ```typescript
   //用于获取store中的全局状态数据
   const store = useStore();
   const visibleRoutes = computed(() => {
     return routes.filter((item) => {
       //根据用户的权限判断是否隐藏菜单
       if (
         !checkPermission(
           store.state.user?.loginUserInfo,
           item.meta?.access as string
         )
       ) {
         return false;
       }
       //根据路由的meta属性判断是否隐藏该菜单
       return !item.meta?.hiddenInMenu;
     });
   });
   ```

   > 使用计算属性可以检查检测用户信息的更改，触发菜单栏的重新渲染，可显示更新后的用户所具有权限的菜单项

#### 7. 全局入口函数

- 在 App.vue 文件中添加全局入口函数

  - 全局只调用一次的函数可以在该函数中进行调用

    ```typescript
    /**
     * 全局初始化函数
     */
    const doInit = () => {
      console.log("kkoj 系统欢迎你！");
    };
    
    onMounted(() => {
      doInit();
    });
    ```

### 二、后端项目初始化

1. 导入后端模板，修改后端模板文件项目名称

2. idea 打开，使用以下快捷键修改相关包名和文件夹名

   > ctrl shift f  全局搜索
   >
   > ctrl shift r  全局替换
   >
   > shift f6 修改文件名

3. 修改 application.yml 文件中的 MySQL 配置信息，并在本地创建相关数据库和数据表

4. 运行项目

   > 运行时遇到的问题
   >
   > 链接数据库时 MySQL 密码正确，但是报错 Access denied for user 'root'@'localhost' (using password: YES)，使用idea 中的 database 工具可连接上，项目运行时就是无法连接
   >
   > 解决方案：重新修改了个新的 MySQL 的密码，就可以连接上了
   >
   > ```sql
   > ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'new_password';
   > 
   > ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'new_password';
   > ```

5. swagger 相关组件

   > Knife4j-OpenAPI2-UI 是 Knife4j 项目中针对 OpenAPI 2.0 规范提供的一款增强型 Swagger 文档 UI 界面。Knife4j（前身是 Swagger-bootstrap-ui）是一个由国内开发者开发的开源工具，用于改善和增强 Java Spring Boot 应用程序中集成 Swagger 后生成的 API 文档界面。
   >
   > ------------------------------------
   >
   > Knife4j-OpenAPI2-spring-boot-starter 是一个用于 Spring Boot 项目的自动化配置starter模块，它简化了在基于 Spring Boot 的应用中集成 Swagger 以支持 OpenAPI 2.0 规范的过程。通过引入这个 starter，开发者可以方便快捷地为项目启用和配置 Swagger2，并使用 Knife4j 提供的增强型文档界面。 

   

### 三、前后端联调

- 前端使用 axios 发送请求

  ```bash
  # 安装 axios
  npm install axios
  ```

  > 传统情况下，前端需要为每个后端接口单独编写一个 axios 请求
  >
  > 现在可以使用 openapi 工具自动根据接口文档生成所有请求
  >
  > [ferdikoomen/openapi-typescript-codegen: NodeJS library that generates Typescript or Javascript clients based on the OpenAPI specification (github.com)](https://github.com/ferdikoomen/openapi-typescript-codegen)、
  >
  > ```bash
  > # 安装该工具
  > npm install openapi-typescript-codegen --save-dev
  > 
  > #根据接口文档生成 axios 请求
  > openapi --input http://localhost:8101/api/v2/api-docs --output ./generated --client axios
  > ```
  >
  > -----------
  >
  > 使用 services 目录下的代码即可发送 axios 请求

- 若需要自定义请求参数（如：baseurl 等）

  1. 可在 OpenApi.ts 中修改

     ```typescript
     export const OpenAPI: OpenAPIConfig = {
       BASE: "http://localhost:8101",
       VERSION: "1.0",
       WITH_CREDENTIALS: false,
       CREDENTIALS: "include",
       TOKEN: undefined,
       USERNAME: undefined,
       PASSWORD: undefined,
       HEADERS: undefined,
       ENCODE_PATH: undefined,
     };
     ```

  2. 可在 axios 的拦截器中配置 axios 的属性

     [拦截器 | Axios中文文档 | Axios中文网 (axios-http.cn)](https://www.axios-http.cn/docs/interceptors)

## 功能开发

### 一、用户登录

1. 前端实现全局的用户登录和路由跳转时权限的检查

   - 针对上面前端项目初始化的第四点做的优化

   > 思路：提取出全局登录和路由权限检查的逻辑，在 App.vue 或者 main.ts 或一些全局组件中引用此逻辑即可
   >
   > ----
   >
   > access 目录下的 index.ts 文件
   >
   > ```typescript
   > import store from "@/store";
   > import router from "@/router";
   > import PERMISSION_ENUM from "@/access/permissionEnum";
   > import checkPermission from "@/access/checkPermission";
   > 
   > //在路由跳转前执行
   > router.beforeEach(async (to, from, next) => {
   >   //自动登录逻辑
   > 
   >   if (store.state.user?.loginUserInfo.userRole == null) {
   >     await store.dispatch("user/getUserInfo");
   >   }
   > 
   >   const userinfo = store.state.user?.loginUserInfo;
   >   const userRole = userinfo.userRole;
   > 
   >   const needPermission = to.meta?.access ?? PERMISSION_ENUM.NOT_LOGIN;
   >   //页面只需要未登录权限则直接跳转
   >   if (needPermission == PERMISSION_ENUM.NOT_LOGIN) {
   >     next();
   >     return;
   >   }
   > 
   >   //页面需要登录权限
   >   //用户未登录
   >   if (!userRole || userRole == PERMISSION_ENUM.NOT_LOGIN) {
   >     //跳转到登录页，登录成功重定向会原页面
   >     next(`user/login?redirect:${to.fullPath}`);
   >     return;
   >   }
   >   //用户已登陆，先检查权限
   >   if (!checkPermission(userinfo, needPermission as string)) {
   >     next("/no-auth");
   >     return;
   >   }
   >   next();
   > });
   > 
   > ```
   >
   > 在 main.ts中引入该文件
   >
   > ```typescript
   > import "./access/index";
   > ```

2. 多页面布局区分

   在 vue.app 中进行区分

   ```vue
   <template>
     <div id="app">
       <template v-if="route.path.startsWith('/user')">
         <router-view />
       </template>
       <template v-else>
         <BasicLayout />
       </template>
     </div>
   </template>
   ```

3. 提交表单实现登录功能

   ```typescript
   /**
    * 提交登录表单
    */
   const handleSubmit = async () => {
     const response = await UserControllerService.userLoginUsingPost(form);
     if (response.code == 0) {
       message.success("登录成功");
   
       //更新全局的用户信息
       store.dispatch("user/getUserInfo");
   
       //跳转至首页
       router.push({
         path: "/",
         replace: true, //不可通过点击浏览器回退按钮返回至该页面
       });
     } else {
       message.error("登录失败");
     }
   };
   ```

   > 注意：
   >
   > http 是无状态的，在发送请求时需要携带 cookie ，服务器通过检查 Cookie 来识别已登录用户并维持其会话状态。
   >
   > 因此前端在发起请求时也需要设置 WITH_CREDENTIALS: true，这样浏览器才会在跨域请求中携带 Cookie
   >
   > 使用 openapi 自动生成请求方式的在 OpenApi.ts 设置 WITH_CREDENTIALS: true
   >
   > ```typescript
   > export const OpenAPI: OpenAPIConfig = {
   >   BASE: "http://localhost:8101",
   >   VERSION: "1.0",
   >   WITH_CREDENTIALS: true,
   >   CREDENTIALS: "include",
   >   TOKEN: undefined,
   >   USERNAME: undefined,
   >   PASSWORD: undefined,
   >   HEADERS: undefined,
   >   ENCODE_PATH: undefined,
   > };
   > ```
   >

   > 不是所有情况下前端请求都需要携带后端返回的cookie。前端是否需要携带cookie取决于具体的业务场景和安全策略：
   >
   > - 同源请求：当前端应用与后端服务位于同一域名下时，浏览器会自动在后续请求中携带相应的cookie，无需显式配置。
   > - 跨域请求：
   >   如果跨域请求需要携带cookie，比如进行身份验证或维持状态等，则需要满足以下条件：
   >   1. 后端服务器必须通过CORS（Cross-Origin Resource Sharing）设置`Access-Control-Allow-Credentials: true`响应头，允许跨域请求包含凭据（如cookie）。
   >      前端在发起请求时需设置withCredentials: true属性，以便AJAX请求或其他HTTP客户端库能携带cookie到服务器。
   >   2. 如果跨域请求不需要携带用户认证相关的cookie信息，或者采用其他方式进行身份验证（例如Bearer Token、JWT等），则无需携带后端返回的cookie。
   >
   > 因此，是否需要携带cookie取决于后端API设计以及应用所采用的身份验证和授权机制。如果确实需要前端在请求中携带cookie，则按照上述跨域规则进行配置即可。否则，在很多现代Web应用中，尤其是在前后端分离架构中，往往倾向于使用无状态的令牌机制来代替传统的session管理方式。



### 二、编写oj系统相关的 crud 接口

后端接口开发的基本流程

> 1. 根据功能设计数据库表
> 2. 自动生成对数据库基本的增删改查（插件：mybatis X）
> 3. 编写 controller，实现基本的增删改查和权限校验
> 4. 根据业务开发定制化的功能

2. 使用 `mybatis X`插件生成相关的`mapper`和`service`

   ![image-20240327203213423](E:%5Cstudy%5Cproject%5Ckkoj%5CREADME.assets%5Cimage-20240327203213423.png)

   ![image-20240327203718380](E:%5Cstudy%5Cproject%5Ckkoj%5CREADME.assets%5Cimage-20240327203718380.png)
   
2. 将生成的代码移到对应的包下

3. 编写 controller、service 逻辑以及相关实体的 dto、vo、枚举值

   > DTO：
   >
   > - 用于封装数据传输对象，可以将数据库中的数据转换为前端需要的格式，方便前后端之间的数据交互。
   >
   > VO：
   >
   > - 用于封装值对象，可以根据具体的需求来封装不同的数据属性，方便前端页面的显示和交互。
   >
   > - 可以节约网路传输大小、或者过滤字段（脱敏）、保证安全性

   > 可以给 JSON 类型的字段编写单独的实体类，更方便处理该 json 字段中的内容 
   >
   > 枚举：例如编程语言、判题状态等

#### 查询题目提交信息需求

- 需求
  - 可以根据用户id、题目id进行查询
  - 用户本人仅能查看自己（提交用户 id与登录用户 id 不同）提交的代码
  - 管理员可以看到所有用户提交的代码

- 实现方案：先查询，再根据用户是否为本人进行脱敏

  ```java
      public QuestionSubmitVO getQuestionSubmitVO(QuestionSubmit questionSubmit, User loginUser) {
          QuestionSubmitVO questionSubmitVO = QuestionSubmitVO.objToVo(questionSubmit);
          //题目提交用户id
          Long userId = questionSubmit.getUserId();
          //除管理员外，用户本人仅能查看自己提交的代码
          if (!userId.equals(loginUser.getId()) && !userService.isAdmin(loginUser)) {
              questionSubmitVO.setCode(null);
          }
          return questionSubmitVO;
      }
  ```

## IDEA 插件

- Mybatis X

  > 用于自动生成 crud

- Generate All Getter And Setter

  > 一键调用一个对象的所有的set方法,get方法等
  > 在方法上生成两个对象的转换

- GenerateSerialVersionUID

  > 生成序列化 uid

- RoboPOJOGenerator

  > 将 json 转化为实体类

- String Manipulation

  > 处理字符串格式 

## 编程经验

### 1. 主键 id 设置为自增容易被他人爬虫

> 为了防止他人根据 id 进行爬虫，可以将 id 生成策略设置为 `ASSIGN_ID`
>
> ```java
>     /**
>      * id
>      */
>     @TableId(type = IdType.ASSIGN_ID)
>     private Long id;
> ```
>
> 注解中的 `type = IdType.ASSIGN_ID` 表示主键 ID 的生成策略为系统自动生成
>
>  `type = IdType.AUTO ` 指定主键的生成策略为自动增长
>
> `type = IdType.ASSIGN_UUID` 表示主键的生成策略是使用UUID