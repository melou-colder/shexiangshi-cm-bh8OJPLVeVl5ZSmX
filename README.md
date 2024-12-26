
随着技术的成熟和 AI 的崛起，很多原本需要团队协作才能完成的工作现在都可以通过自动化和智能化的方式完成。于是乎，**单个开发者的能力得到了极大的提升** \- 借助各种工具，一个人就可以完成开发、测试、运维等整条链路上的工作，渡劫飞升成为真正的 “全干工程师”。


之前我们分享过一些入门级的 Hello World 教程。今天，我想通过一个实际的业务案例来展示 Devbox 并非只能开发玩具，而是一个真正的生产力工具。


[Sealos](https://github.com) 平台上有很多应用，其中很多管控层面的应用都是使用 Cursor \+ Go \+ Next.js 开发的。我们的开发环境直接使用 [Sealos Devbox](https://github.com)，上线也是通过 Devbox 一键完成。这种开发模式让我们团队拥有了非常高效的作战能力 \- **大部分重复性工作都通过自动化或 AI 完成，让开发者可以专注于核心业务逻辑**。


![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115803359-771121103.png)


以 Sealos 中的 [AI Proxy](https://github.com) 应用为例，这是一个典型的前后端分离架构的应用，主要由两部分组成：


1. 基于 Next.js 开发的前端应用和 BFF 层。BFF 层负责用户鉴权，并将经过验证的请求转发给后端服务。
2. 使用 Golang 开发的后端服务，负责核心业务逻辑，包括 token 存储、日志记录和请求转发等功能。


![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115804846-294253458.png)


接下来，我将详细介绍**如何高效地开发这样一个生产级别的系统**。


## Golang 后端


### 创建开发环境


首先在 [Sealos Cloud](https://github.com):[wgetCloud机场](https://tabijibiyori.org) 中打开 Debox 应用，创建一个新项目，选择 Go 作为运行环境，选择 1\.23 版本。


![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115806036-340212529.png)


Devbox 为开发者提供了几个非常实用的功能：


* **灵活的资源配置**：可以根据项目需求自由调整 CPU 和内存，选择合适配置既保证性能又能控制成本。
* **一键启用 HTTPS**：系统自动分配安全域名，再也不用为配置 SSL 证书发愁。
* **全自动域名管理**：从开发到测试环境，域名配置全程由系统处理，开发者可以专注于代码本身。


创建完成后，几秒钟即可启动开发环境。


环境准备好后，我们直接用 Cursor 连接开发环境。在操作选项中选择使用 Cursor 连接：


![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115807146-1919356021.png)


首次打开会提示安装 [Devbox 插件](https://github.com)，安装后即可自动连接开发环境。


![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115808073-964144479.png)


### 导入项目到 Cursor


首先 Fork [Sealos 源码](https://github.com)到自己的仓库，然后再将你自己的仓库克隆到 Devbox 开发环境：


![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115809229-168174455.png)


![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115810322-2120157871.png)


### 测试环境开发


在 Cursor 的面板中切换到 “Databse” 标签页，然后点击箭头指向的按钮，在浏览器中打开 Sealos 的数据库应用：


![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115811654-1157967612.png)


然后创建 PostgreSQL 和 Redis 实例。


![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115812756-931796782.png)


![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115814477-350967599.png)


回到 Cursor 面板的 “Database” 标签页，点击刷新即可看到刚创建的数据库实例，点击可复制连接信息：


![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115815461-1503324922.png)


在终端中启动服务：



```
export ADMIN_KEY=sealos-admin
export SQL_DSN=<复制的pgsql连接串>/postgres
export REDIS_CONN_STRING=<复制的redis连接串>
go run . --port 8080

```

提示 `server stared` 即为启动成功


![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115816361-1386262905.png)


在 Cursor 面板的 “Network” 标签页中，点击地址栏右侧的 🌐 按钮，然后在弹窗中选择 “Copy”，将地址复制到自己电脑上使用 curl 进行测试：



```
curl https://mmznjndvzdrv.sealoshzh.site/api/status -H "Authorization: sealos-admin"

```

![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115817240-1509871865.png)


接口返回没有问题。


### 优化数据库设计


在开发过程中，我们发现数据库中 Group 和 Token 之间的外键约束增加了系统维护的复杂度。为了简化这一关系，我们可以将外键约束改为程序层面的显式调用，这样可以让代码逻辑更加清晰和可控。


首先切换到 `fix-aiproxy` 分支：


![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115817929-366784844.png)


在 `sealos/service/aiproxy/model/group.go` 文件中，我们需要将 Group 结构体中一个外键约束改成在程序内显示调用更新和删除来降低维护心智。


这里我选择使用 Cursor 的 Chat 功能让 AI 自己写代码，最后生成的结果如下：


![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115818620-1957004520.png)


![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115819317-630720045.png)


这种实现方式的优势在于：当删除 Group 时，相关的 Token 删除操作会在同一个事务中完成。由于是在事务内进行，我们不需要担心删除失败或系统宕机导致的数据不一致问题。


我们可以通过一系列测试来验证这个优化是否达到预期效果。首先编译并运行服务：



```
go build . && ./aiproxy --port 8080

```

然后通过以下 API 调用来测试完整的 Group 和 Token 生命周期：



```
# 创建一个group
curl https://gawavirgsomu.sealosbja.site/api/group/ -H "Authorization: sealos-admin" -d '{
    "id": "ns-admin"
}'

# 创建一个token
curl https://gawavirgsomu.sealosbja.site/api/token/ns-admin -H "Authorization: sealos-admin" -d '{
    "name": "token 1"
}'

# 查询token
curl https://gawavirgsomu.sealosbja.site/api/tokens/ -H "Authorization: sealos-admin"

# 删除group
curl https://gawavirgsomu.sealosbja.site/api/group/ns-admin -H "Authorization: sealos-admin" -X DELETE

# 再次查询token
curl https://gawavirgsomu.sealosbja.site/api/tokens/ -H "Authorization: sealos-admin"

```

![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115820239-170020683.png)


测试结果符合预期，确认优化方案可行。接下来我们就可以提交这些更改并创建 Pull Request 了。


### 上线到生产环境


首先在 Cursor 目录顶层的 endpoint.sh 中设置启动命令，在文件中添加以下启动配置：



```
cd sealos/service/aiproxy
export ADMIN_KEY=sealos-admin
# 可以再创建一个单独的生产环境数据库，与开发环境隔离
export SQL_DSN=<复制的pgsql连接串>/postgres
export REDIS_CONN_STRING=<复制的redis连接串>
# 使用编译好的二进制文件
./aiproxy --port 8080

```

![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115820945-558622857.png)


然后来到 Devbox 发布页面发布版本：


![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115821693-1336718034.png)


点击发布按钮后，等待发布流程完成。发布成功后，点击 “上线” 按钮进入部署页面，然后点击 “部署应用” 即可：


![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115822596-567160715.png)


![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115823316-9357405.png)


部署完成后，进入应用的详情页面，等待应用变成 running 状态，然后复制公网地址：


![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115824027-118137723.png)


这个公网地址就是生产环境的域名，我们可以使用生产环境的域名进行测试：



```
# 这里使用的是生产环境的域名
curl https://jpesudzryuhp.sealosbja.site/api/tokens/ -H "Authorization: sealos-admin"

```

![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115824696-1336072587.png)


完美！


## Next.js 前端


### 前端项目搭建


前端环境的搭建与后端类似，具体步骤如下：


1. 在 Devbox 中创建一个 Node.js 环境，版本选择 20，端口改成 3000。由于 pnpm 安装依赖比较消耗资源，建议选择 `4c 16G` 的配置。然后克隆你自己 Fork 的 Sealos 仓库：`git clone https://github.com/xxx/sealos.git`。AI Proxy 的前端代码位于 `sealos/frontend/providers/aiproxy` 目录。
2. 切换到 `sealos/frontend` 目录，首先修改 `sealos/frontend/package.josn` 文件，去除 node 版本限制，直接删除 `"node": "20.4.0"` 和 `"pnpm": "8.9.0"` 这两行即可，**这一步很重要，不然代码构建依赖会不成功**。


![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115825625-1979230555.png)
3. 执行命令 `pnpm i` 安装依赖。


![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115826573-2018829057.png)
4. 执行命令 `pnpm -r --filter ./packages/client-sdk run build` 构建 client\-sdk 包。


![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115827294-1172430562.png)
5. 为了让 Cursor 的 i18n 插件正常工作，我们需要将项目根目录切换到 `sealos/frontend/providers/aiproxy`：


![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115828235-1059587408.png)


切换目录后，建议安装所有 @recommended 插件以获得最佳的开发体验：


![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115829184-691378787.png)
6. 之前只是构建出了 Sealos Desktop SDK，并没有安装 aiproxy 的依赖，aiproxy 的依赖需要在 aiproxy 工作目录下 `sealos/frontend/providers/aiproxy` 进行安装。直接执行命令 `pnpm i` 安装即可：


![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115830404-441409710.png)


### 对接后端环境


项目搭建完成后，我们需要配置环境变量来对接后端服务。在项目根目录创建一个 `.env` 文件，需要配置以下几个关键变量：



```
NEXT_PUBLIC_MOCK_USER=""
AI_PROXY_BACKEND_KEY=""
APP_TOKEN_JWT_KEY="test123"
AI_PROXY_BACKEND=""
AI_PROXY_BACKEND_INTERNAL=""
ADMIN_NAMESPACES=""

```

* `NEXT_PUBLIC_MOCK_USER`：由于 AI Proxy 是 Sealos Desktop 的一部分，用户认证通过 JWT Token 实现，AI Proxy 只做解析 Token，JWT Token 的签发由 Sealos Desktop 完成。在开发阶段，我们需要 mock 一个 JWT Token。NEXT\_PUBLIC\_MOCK\_USER 的值就是 mock 出来的 JWT Token。可以使用在线工具 **[https://www.lddgo.net/encrypt/jwt\-generate](https://github.com)** 生成。


mock 数据如下：



```
{
    "workspaceId" : "test"
}

```

![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115831355-20418334.png)
* `APP_TOKEN_JWT_KEY`：JWT Token 的密钥 (随便写)
* `AI_PROXY_BACKEND_KEY`：后端 API 的访问密钥 (也就是后端项目的 ADMIN\_KEY)
* `AI_PROXY_BACKEND`：后端服务的公网地址
* `AI_PROXY_BACKEND_INTERNAL`：后端服务的内网地址 (开发测试阶段可以不填)
* `ADMIN_NAMESPACES`：管理员用户名，开发时填 test 就行，和 token 中的 “workspaceId”：“test” 保持一致


环境变量配置完成后，运行 `pnpm dev` 即可启动开发服务器。项目的发布和部署流程与前面介绍的后端开发流程完全一致。


![](https://img2024.cnblogs.com/other/1737323/202412/1737323-20241226115832273-1286743379.png)


## 总结


Sealos AI Proxy 前端项目采用了经典的 Next.js App Router 架构，其中 `app/[lng]` 目录用于页面路由，`app/api` 目录则用于后端 API 路由。


在这个项目中，Next.js 的后端实际上是一个中间层，它主要负责用户认证相关的业务逻辑，并将经过认证的请求转发给真正的 Golang 后端服务。这种分层设计可以让 Golang 后端专注于核心业务逻辑，不需要关心认证等基础设施，从而提高了代码的灵活性和可移植性。


