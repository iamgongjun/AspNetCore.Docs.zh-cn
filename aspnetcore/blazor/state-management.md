---
title: ASP.NET Core Blazor 状态管理
author: guardrex
description: 了解如何在 Blazor 服务器端应用中持久保存状态。
monikerRange: '>= aspnetcore-3.0'
ms.author: riande
ms.custom: mvc
ms.date: 08/13/2019
uid: blazor/state-management
ms.openlocfilehash: af040635302fbf2dae8192dcf37d55bfcfedfcec
ms.sourcegitcommit: f5f0ff65d4e2a961939762fb00e654491a2c772a
ms.translationtype: MT
ms.contentlocale: zh-CN
ms.lasthandoff: 08/15/2019
ms.locfileid: "69030377"
---
# <a name="aspnet-core-blazor-state-management"></a><span data-ttu-id="8d3f9-103">ASP.NET Core Blazor 状态管理</span><span class="sxs-lookup"><span data-stu-id="8d3f9-103">ASP.NET Core Blazor state management</span></span>

<span data-ttu-id="8d3f9-104">作者：[Steve Sanderson](https://github.com/SteveSandersonMS)</span><span class="sxs-lookup"><span data-stu-id="8d3f9-104">By [Steve Sanderson](https://github.com/SteveSandersonMS)</span></span>

<span data-ttu-id="8d3f9-105">Blazor 服务器端是有状态的应用程序框架。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-105">Blazor server-side is a stateful app framework.</span></span> <span data-ttu-id="8d3f9-106">大多数情况下, 应用保持与服务器的持续连接。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-106">Most of the time, the app maintains an ongoing connection to the server.</span></span> <span data-ttu-id="8d3f9-107">用户处于*线路*中的服务器内存中。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-107">The user's state is held in the server's memory in a *circuit*.</span></span> 

<span data-ttu-id="8d3f9-108">用户线路的状态的示例包括:</span><span class="sxs-lookup"><span data-stu-id="8d3f9-108">Examples of state held for a user's circuit include:</span></span>

* <span data-ttu-id="8d3f9-109">呈现的 UI&mdash;, 组件实例的层次结构及其最新的呈现输出。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-109">The rendered UI&mdash;the hierarchy of component instances and their most recent render output.</span></span>
* <span data-ttu-id="8d3f9-110">组件实例中的任何字段和属性的值。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-110">The values of any fields and properties in component instances.</span></span>
* <span data-ttu-id="8d3f9-111">依赖于线路的[依赖关系注入 (DI)](xref:fundamentals/dependency-injection)服务实例中保存的数据。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-111">Data held in [dependency injection (DI)](xref:fundamentals/dependency-injection) service instances that are scoped to the circuit.</span></span>

> [!NOTE]
> <span data-ttu-id="8d3f9-112">本文介绍 Blazor 服务器端应用中的状态暂留。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-112">This article addresses state persistence in Blazor server-side apps.</span></span> <span data-ttu-id="8d3f9-113">Blazor 客户端应用程序可以[在浏览器中利用客户端状态持久性, 但在](#client-side-in-the-browser)本文讨论范围之外需要自定义解决方案或第三方包。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-113">Blazor client-side apps can take advantage of [client-side state persistence in the browser](#client-side-in-the-browser) but require custom solutions or 3rd party packages beyond the scope of this article.</span></span>

## <a name="blazor-circuits"></a><span data-ttu-id="8d3f9-114">Blazor 电路</span><span class="sxs-lookup"><span data-stu-id="8d3f9-114">Blazor circuits</span></span>

<span data-ttu-id="8d3f9-115">如果用户遇到暂时的网络连接丢失, Blazor 会尝试将用户重新连接到其原始线路, 以便他们可以继续使用该应用。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-115">If a user experiences a temporary network connection loss, Blazor attempts to reconnect the user to their original circuit so they can continue to use the app.</span></span> <span data-ttu-id="8d3f9-116">但是, 并不总是能够将用户重新连接到服务器内存中的原始线路:</span><span class="sxs-lookup"><span data-stu-id="8d3f9-116">However, reconnecting a user to their original circuit in the server's memory isn't always possible:</span></span>

* <span data-ttu-id="8d3f9-117">服务器不能永久保留断开连接的线路。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-117">The server can't retain a disconnected circuit forever.</span></span> <span data-ttu-id="8d3f9-118">在超时或服务器处于内存压力下时, 服务器必须释放断开连接的线路。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-118">The server must release a disconnected circuit after a timeout or when the server is under memory pressure.</span></span>
* <span data-ttu-id="8d3f9-119">在多服务器、负载平衡的部署环境中, 任何服务器处理请求在任何给定时间都可能会变得不可用。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-119">In multiserver, load-balanced deployment environments, any server processing requests may become unavailable at any given time.</span></span> <span data-ttu-id="8d3f9-120">当不再需要处理请求的总数量时, 单独的服务器可能会失败或自动删除。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-120">Individual servers may fail or be automatically removed when no longer required to handle the overall volume of requests.</span></span> <span data-ttu-id="8d3f9-121">当用户尝试重新连接时, 原始服务器可能不可用。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-121">The original server may not be available when the user attempts to reconnect.</span></span>
* <span data-ttu-id="8d3f9-122">用户可能会关闭并重新打开其浏览器或重新加载页面, 这会删除浏览器内存中保存的任何状态。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-122">The user might close and re-open their browser or reload the page, which removes any state held in the browser's memory.</span></span> <span data-ttu-id="8d3f9-123">例如, 通过 JavaScript 互操作调用设置的值将丢失。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-123">For example, values set through JavaScript interop calls are lost.</span></span>

<span data-ttu-id="8d3f9-124">当无法将用户重新连接到其原始线路时, 用户将收到一个具有空状态的新线路。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-124">When a user can't be reconnected to their original circuit, the user receives a new circuit with an empty state.</span></span> <span data-ttu-id="8d3f9-125">这等效于关闭和重新打开桌面应用程序。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-125">This is equivalent to closing and re-opening a desktop app.</span></span>

## <a name="preserve-state-across-circuits"></a><span data-ttu-id="8d3f9-126">跨线路保留状态</span><span class="sxs-lookup"><span data-stu-id="8d3f9-126">Preserve state across circuits</span></span>

<span data-ttu-id="8d3f9-127">在某些情况下, 需要保留线路上的状态。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-127">In some scenarios, preserving state across circuits is desirable.</span></span> <span data-ttu-id="8d3f9-128">如果出现以下情况, 应用可以为用户保留重要数据:</span><span class="sxs-lookup"><span data-stu-id="8d3f9-128">An app can retain important data for a user if:</span></span>

* <span data-ttu-id="8d3f9-129">Web 服务器不可用。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-129">The web server becomes unavailable.</span></span>
* <span data-ttu-id="8d3f9-130">用户的浏览器将强制使用新的 web 服务器启动新线路。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-130">The user's browser is forced to start a new circuit with a new web server.</span></span>

<span data-ttu-id="8d3f9-131">通常情况下, 跨线路维护状态适用于用户主动创建数据的情况, 不只是读取已存在的数据。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-131">In general, maintaining state across circuits applies to scenarios where users are actively creating data, not simply reading data that already exists.</span></span>

<span data-ttu-id="8d3f9-132">若要将状态保留在单个线路以外, 请*不要只将数据存储在服务器的内存中*。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-132">To preserve state beyond a single circuit, *don't merely store the data in the server's memory*.</span></span> <span data-ttu-id="8d3f9-133">应用必须将数据保存到其他某个存储位置。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-133">The app must persist the data to some other storage location.</span></span> <span data-ttu-id="8d3f9-134">状态持久性不是&mdash;自动的, 你必须在开发应用程序时采取措施来实现有状态数据持久性。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-134">State persistence isn't automatic&mdash;you must take steps when developing the app to implement stateful data persistence.</span></span>

<span data-ttu-id="8d3f9-135">数据暂留通常只需要用于用户需要花费大量精力才能创建的高值状态。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-135">Data persistence is typically only required for high-value state that users have expended effort to create.</span></span> <span data-ttu-id="8d3f9-136">在下面的示例中, 保存状态可节省商业活动的时间或辅助:</span><span class="sxs-lookup"><span data-stu-id="8d3f9-136">In the following examples, persisting state either saves time or aids in commercial activities:</span></span>

* <span data-ttu-id="8d3f9-137">多步骤 webform &ndash; , 如果用户的状态丢失, 则用户需要为多步骤进程的几个已完成步骤重新输入数据。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-137">Multistep webform &ndash; It's time-consuming for a user to re-enter data for several completed steps of a multistep process if their state is lost.</span></span> <span data-ttu-id="8d3f9-138">如果用户离开多步骤窗体并稍后返回窗体, 则该用户将失去状态。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-138">A user loses state in this scenario if they navigate away from the multistep form and return to the form later.</span></span>
* <span data-ttu-id="8d3f9-139">购物车&ndash;可以维持任何代表潜在收入的应用的商业重要组件。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-139">Shopping cart &ndash; Any commercially important component of an app that represents potential revenue can be maintained.</span></span> <span data-ttu-id="8d3f9-140">如果用户的购物车丢失了其状态, 因此他们的购物车以后可能会购买更少的产品或服务。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-140">A user who loses their state, and thus their shopping cart, may purchase fewer products or services when they return to the site later.</span></span>

<span data-ttu-id="8d3f9-141">通常不需要保留轻松重新创建的状态, 例如输入到未提交的登录对话框中的用户名。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-141">It's usually not necessary to preserve easily-recreated state, such as the username entered into a sign-in dialog that hasn't been submitted.</span></span>

> [!IMPORTANT]
> <span data-ttu-id="8d3f9-142">应用只能保留*应用状态*。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-142">An app can only persist *app state*.</span></span> <span data-ttu-id="8d3f9-143">不能持久保存 Ui, 如组件实例及其呈现树。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-143">UIs can't be persisted, such as component instances and their render trees.</span></span> <span data-ttu-id="8d3f9-144">组件和呈现树通常不能序列化。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-144">Components and render trees aren't generally serializable.</span></span> <span data-ttu-id="8d3f9-145">若要保存类似于 UI 状态的内容 (如 TreeView 的扩展节点), 应用必须具有自定义代码, 以便将行为建模为可序列化应用状态。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-145">To persist something similar to UI state, such as the expanded nodes of a TreeView, the app must have custom code to model the behavior as serializable app state.</span></span>

## <a name="where-to-persist-state"></a><span data-ttu-id="8d3f9-146">保留状态的位置</span><span class="sxs-lookup"><span data-stu-id="8d3f9-146">Where to persist state</span></span>

<span data-ttu-id="8d3f9-147">有三个常见位置用于在 Blazor 服务器端应用程序中保持状态。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-147">Three common locations exist for persisting state in a Blazor server-side app.</span></span> <span data-ttu-id="8d3f9-148">每种方法最适合于不同的方案, 并且有不同的注意事项:</span><span class="sxs-lookup"><span data-stu-id="8d3f9-148">Each approach is best suited to different scenarios and has different caveats:</span></span>

* [<span data-ttu-id="8d3f9-149">数据库中的服务器端</span><span class="sxs-lookup"><span data-stu-id="8d3f9-149">Server-side in a database</span></span>](#server-side-in-a-database)
* [<span data-ttu-id="8d3f9-150">URL</span><span class="sxs-lookup"><span data-stu-id="8d3f9-150">URL</span></span>](#url)
* [<span data-ttu-id="8d3f9-151">浏览器中的客户端</span><span class="sxs-lookup"><span data-stu-id="8d3f9-151">Client-side in the browser</span></span>](#client-side-in-the-browser)

### <a name="server-side-in-a-database"></a><span data-ttu-id="8d3f9-152">数据库中的服务器端</span><span class="sxs-lookup"><span data-stu-id="8d3f9-152">Server-side in a database</span></span>

<span data-ttu-id="8d3f9-153">对于永久数据持久性或必须跨多个用户或设备的任何数据, 一个独立的服务器端数据库几乎无疑是最佳选择。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-153">For permanent data persistence or for any data that must span multiple users or devices, an independent server-side database is almost certainly the best choice.</span></span> <span data-ttu-id="8d3f9-154">选项包括：</span><span class="sxs-lookup"><span data-stu-id="8d3f9-154">Options include:</span></span>

* <span data-ttu-id="8d3f9-155">关系 SQL 数据库</span><span class="sxs-lookup"><span data-stu-id="8d3f9-155">Relational SQL database</span></span>
* <span data-ttu-id="8d3f9-156">键-值存储</span><span class="sxs-lookup"><span data-stu-id="8d3f9-156">Key-value store</span></span>
* <span data-ttu-id="8d3f9-157">Blob 存储区</span><span class="sxs-lookup"><span data-stu-id="8d3f9-157">Blob store</span></span>
* <span data-ttu-id="8d3f9-158">表存储</span><span class="sxs-lookup"><span data-stu-id="8d3f9-158">Table store</span></span>

<span data-ttu-id="8d3f9-159">将数据保存到数据库后, 用户可以随时启动新线路。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-159">After data is saved in the database, a new circuit can be started by a user at any time.</span></span> <span data-ttu-id="8d3f9-160">用户的数据会保留在任何新线路中并可用。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-160">The user's data is retained and available in any new circuit.</span></span>

<span data-ttu-id="8d3f9-161">有关 Azure 数据存储选项的详细信息, 请参阅[Azure 存储空间文档](/azure/storage/)和[azure 数据库](https://azure.microsoft.com/product-categories/databases/)。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-161">For more information on Azure data storage options, see the [Azure Storage Documentation](/azure/storage/) and [Azure Databases](https://azure.microsoft.com/product-categories/databases/).</span></span>

### <a name="url"></a><span data-ttu-id="8d3f9-162">URL</span><span class="sxs-lookup"><span data-stu-id="8d3f9-162">URL</span></span>

<span data-ttu-id="8d3f9-163">对于表示导航状态的暂时性数据, 请将数据作为 URL 的一部分进行建模。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-163">For transient data representing navigation state, model the data as a part of the URL.</span></span> <span data-ttu-id="8d3f9-164">URL 中的状态模型示例包括:</span><span class="sxs-lookup"><span data-stu-id="8d3f9-164">Examples of state modeled in the URL include:</span></span>

* <span data-ttu-id="8d3f9-165">已查看实体的 ID。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-165">The ID of a viewed entity.</span></span>
* <span data-ttu-id="8d3f9-166">分页网格中的当前页码。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-166">The current page number in a paged grid.</span></span>

<span data-ttu-id="8d3f9-167">将保留浏览器地址栏的内容:</span><span class="sxs-lookup"><span data-stu-id="8d3f9-167">The contents of the browser's address bar are retained:</span></span>

* <span data-ttu-id="8d3f9-168">如果用户手动重新加载页面, 则为。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-168">If the user manually reloads the page.</span></span>
* <span data-ttu-id="8d3f9-169">如果 web 服务器变为不可用&mdash;, 则会强制用户重新加载页面, 以便连接到其他服务器。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-169">If the web server becomes unavailable&mdash;the user is forced to reload the page in order to connect to a different server.</span></span>

<span data-ttu-id="8d3f9-170">有关用`@page`指令定义 URL 模式的信息, 请参阅<xref:blazor/routing>。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-170">For information on defining URL patterns with the `@page` directive, see <xref:blazor/routing>.</span></span>

### <a name="client-side-in-the-browser"></a><span data-ttu-id="8d3f9-171">浏览器中的客户端</span><span class="sxs-lookup"><span data-stu-id="8d3f9-171">Client-side in the browser</span></span>

<span data-ttu-id="8d3f9-172">对于用户正在主动创建的暂时性数据, 公共后备存储是浏览器的`localStorage`和`sessionStorage`集合。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-172">For transient data that the user is actively creating, a common backing store is the browser's `localStorage` and `sessionStorage` collections.</span></span> <span data-ttu-id="8d3f9-173">如果放弃该线路, 则无需使用该应用程序管理或清除已存储状态, 这与服务器端存储相比, 这是一项优势。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-173">The app isn't required to manage or clear the stored state if the circuit is abandoned, which is an advantage over server-side storage.</span></span>

> [!NOTE]
> <span data-ttu-id="8d3f9-174">本部分中的 "客户端" 指的是浏览器中的客户端方案, 而不是[Blazor 客户端承载模型](xref:blazor/hosting-models#client-side)。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-174">"Client-side" in this section refers to client-side scenarios in the browser, not the [Blazor client-side hosting model](xref:blazor/hosting-models#client-side).</span></span> <span data-ttu-id="8d3f9-175">`localStorage`和`sessionStorage`可用于 Blazor 客户端应用, 但仅通过编写自定义代码或使用第三方包。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-175">`localStorage` and `sessionStorage` can be used in Blazor client-side apps but only by writing custom code or using a 3rd party package.</span></span>

<span data-ttu-id="8d3f9-176">`localStorage`和`sessionStorage`不同之处如下:</span><span class="sxs-lookup"><span data-stu-id="8d3f9-176">`localStorage` and `sessionStorage` differ as follows:</span></span>

* <span data-ttu-id="8d3f9-177">`localStorage`的作用域限定为用户的浏览器。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-177">`localStorage` is scoped to the user's browser.</span></span> <span data-ttu-id="8d3f9-178">如果用户重新加载页面或关闭并重新打开浏览器, 则状态将保持不变。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-178">If the user reloads the page or closes and re-opens the browser, the state persists.</span></span> <span data-ttu-id="8d3f9-179">如果用户打开多个浏览器选项卡, 则状态在选项卡上共享。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-179">If the user opens multiple browser tabs, the state is shared across the tabs.</span></span> <span data-ttu-id="8d3f9-180">数据一直保留`localStorage`在明确清除之前。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-180">Data persists in `localStorage` until explicitly cleared.</span></span>
* <span data-ttu-id="8d3f9-181">`sessionStorage`的作用域限定为用户的浏览器选项卡。如果用户重新加载该选项卡, 则状态将保持不变。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-181">`sessionStorage` is scoped to the user's browser tab. If the user reloads the tab, the state persists.</span></span> <span data-ttu-id="8d3f9-182">如果用户关闭该选项卡或浏览器, 则状态将丢失。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-182">If the user closes the tab or the browser, the state is lost.</span></span> <span data-ttu-id="8d3f9-183">如果用户打开多个浏览器选项卡, 则每个选项卡都有自己独立的数据版本。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-183">If the user opens multiple browser tabs, each tab has its own independent version of the data.</span></span>

<span data-ttu-id="8d3f9-184">通常, `sessionStorage`使用更安全。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-184">Generally, `sessionStorage` is safer to use.</span></span> <span data-ttu-id="8d3f9-185">`sessionStorage`避免了用户打开多个选项卡并遇到以下问题的风险:</span><span class="sxs-lookup"><span data-stu-id="8d3f9-185">`sessionStorage` avoids the risk that a user opens multiple tabs and encounters the following:</span></span>

* <span data-ttu-id="8d3f9-186">跨选项卡的状态存储中的 bug。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-186">Bugs in state storage across tabs.</span></span>
* <span data-ttu-id="8d3f9-187">当选项卡覆盖其他选项卡状态时, 混乱的行为。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-187">Confusing behavior when a tab overwrites the state of other tabs.</span></span>

<span data-ttu-id="8d3f9-188">`localStorage`如果应用必须在关闭时保持状态, 然后重新打开浏览器, 则是更好的选择。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-188">`localStorage` is the better choice if the app must persist state across closing and re-opening the browser.</span></span>

<span data-ttu-id="8d3f9-189">使用浏览器存储时的注意事项:</span><span class="sxs-lookup"><span data-stu-id="8d3f9-189">Caveats for using browser storage:</span></span>

* <span data-ttu-id="8d3f9-190">与使用服务器端数据库类似, 数据的加载和保存都是异步的。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-190">Similar to the use of a server-side database, loading and saving data are asynchronous.</span></span>
* <span data-ttu-id="8d3f9-191">与服务器端数据库不同, 在预呈现期间, 存储不能使用, 因为在预呈现阶段, 请求的页面在浏览器中不存在。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-191">Unlike a server-side database, storage isn't available during prerendering because the requested page doesn't exist in the browser during the prerendering stage.</span></span>
* <span data-ttu-id="8d3f9-192">存储少量数据对于 Blazor 服务器端应用而言是合理的。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-192">Storage of a few kilobytes of data is reasonable to persist for Blazor server-side apps.</span></span> <span data-ttu-id="8d3f9-193">除了几 kb 外, 还必须考虑性能的影响, 因为数据是通过网络加载并保存的。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-193">Beyond a few kilobytes, you must consider the performance implications because the data is loaded and saved across the network.</span></span>
* <span data-ttu-id="8d3f9-194">用户可以查看或篡改数据。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-194">Users may view or tamper with the data.</span></span> <span data-ttu-id="8d3f9-195">ASP.NET Core 的[数据保护](xref:security/data-protection/introduction)可以降低风险。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-195">ASP.NET Core [Data Protection](xref:security/data-protection/introduction) can mitigate the risk.</span></span>

## <a name="third-party-browser-storage-solutions"></a><span data-ttu-id="8d3f9-196">第三方浏览器存储解决方案</span><span class="sxs-lookup"><span data-stu-id="8d3f9-196">Third-party browser storage solutions</span></span>

<span data-ttu-id="8d3f9-197">第三方 NuGet 包提供使用`localStorage`和`sessionStorage`的 api。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-197">Third-party NuGet packages provide APIs for working with `localStorage` and `sessionStorage`.</span></span>

<span data-ttu-id="8d3f9-198">值得一提的是, 选择一个可透明地使用 ASP.NET Core 的[数据保护](xref:security/data-protection/introduction)的包。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-198">It's worth considering choosing a package that transparently uses ASP.NET Core's [Data Protection](xref:security/data-protection/introduction).</span></span> <span data-ttu-id="8d3f9-199">ASP.NET Core 的数据保护会对存储的数据进行加密, 并降低篡改存储数据的潜在风险。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-199">ASP.NET Core Data Protection encrypts stored data and reduces the potential risk of tampering with stored data.</span></span> <span data-ttu-id="8d3f9-200">如果 JSON 序列化的数据以纯文本形式存储, 则用户可以使用浏览器开发人员工具查看数据, 还可以修改存储的数据。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-200">If JSON-serialized data is stored in plaintext, users can see the data using browser developer tools and also modify the stored data.</span></span> <span data-ttu-id="8d3f9-201">保护数据并不总是问题, 因为数据在性质上可能很简单。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-201">Securing data isn't always a problem because the data might be trivial in nature.</span></span> <span data-ttu-id="8d3f9-202">例如, 读取或修改 UI 元素的存储颜色不会对用户或组织造成严重的安全风险。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-202">For example, reading or modifying the stored color of a UI element isn't a significant security risk to the user or the organization.</span></span> <span data-ttu-id="8d3f9-203">避免允许用户检查或篡改*敏感数据*。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-203">Avoid allowing users to inspect or tamper with *sensitive data*.</span></span>

## <a name="protected-browser-storage-experimental-package"></a><span data-ttu-id="8d3f9-204">受保护的浏览器存储试验包</span><span class="sxs-lookup"><span data-stu-id="8d3f9-204">Protected Browser Storage experimental package</span></span>

<span data-ttu-id="8d3f9-205">为提供[数据保护](xref:security/data-protection/introduction) `localStorage` `sessionStorage`的 NuGet 包的一个示例是[AspNetCore. ProtectedBrowserStorage](https://www.nuget.org/packages/Microsoft.AspNetCore.ProtectedBrowserStorage)。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-205">An example of a NuGet package that provides [Data Protection](xref:security/data-protection/introduction) for `localStorage` and `sessionStorage` is [Microsoft.AspNetCore.ProtectedBrowserStorage](https://www.nuget.org/packages/Microsoft.AspNetCore.ProtectedBrowserStorage).</span></span>

> [!WARNING]
> <span data-ttu-id="8d3f9-206">`Microsoft.AspNetCore.ProtectedBrowserStorage`是一个不受支持的实验性包, 此时不适合用于生产。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-206">`Microsoft.AspNetCore.ProtectedBrowserStorage` is an unsupported experimental package unsuitable for production use at this time.</span></span>

### <a name="installation"></a><span data-ttu-id="8d3f9-207">安装</span><span class="sxs-lookup"><span data-stu-id="8d3f9-207">Installation</span></span>

<span data-ttu-id="8d3f9-208">安装`Microsoft.AspNetCore.ProtectedBrowserStorage`包:</span><span class="sxs-lookup"><span data-stu-id="8d3f9-208">To install the `Microsoft.AspNetCore.ProtectedBrowserStorage` package:</span></span>

1. <span data-ttu-id="8d3f9-209">在 Blazor 服务器端应用程序项目中, 将包引用添加到[AspNetCore. ProtectedBrowserStorage](https://www.nuget.org/packages/Microsoft.AspNetCore.ProtectedBrowserStorage)。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-209">In the Blazor server-side app project, add a package reference to [Microsoft.AspNetCore.ProtectedBrowserStorage](https://www.nuget.org/packages/Microsoft.AspNetCore.ProtectedBrowserStorage).</span></span>
1. <span data-ttu-id="8d3f9-210">在顶级 HTML (例如, 在默认项目模板中的*Pages/_Host*文件中) 添加以下`<script>`标记:</span><span class="sxs-lookup"><span data-stu-id="8d3f9-210">In the top-level HTML (for example, in the *Pages/_Host.cshtml* file in the default project template), add the following `<script>` tag:</span></span>

   ```html
   <script src="_content/Microsoft.AspNetCore.ProtectedBrowserStorage/protectedBrowserStorage.js"></script>
   ```

1. <span data-ttu-id="8d3f9-211">在方法中, 调用`AddProtectedBrowserStorage`以将`localStorage`和`sessionStorage`服务添加到服务集合: `Startup.ConfigureServices`</span><span class="sxs-lookup"><span data-stu-id="8d3f9-211">In the `Startup.ConfigureServices` method, call `AddProtectedBrowserStorage` to add `localStorage` and `sessionStorage` services to the service collection:</span></span>

   ```csharp
   services.AddProtectedBrowserStorage();
   ```

### <a name="save-and-load-data-within-a-component"></a><span data-ttu-id="8d3f9-212">保存和加载组件中的数据</span><span class="sxs-lookup"><span data-stu-id="8d3f9-212">Save and load data within a component</span></span>

<span data-ttu-id="8d3f9-213">在需要将数据加载到浏览器存储或将数据保存到[@inject](xref:blazor/dependency-injection#request-a-service-in-a-component)浏览器存储的任何组件中, 使用注入下列任一的实例:</span><span class="sxs-lookup"><span data-stu-id="8d3f9-213">In any component that requires loading or saving data to browser storage, use [@inject](xref:blazor/dependency-injection#request-a-service-in-a-component) to inject an instance of either of the following:</span></span>

* `ProtectedLocalStorage`
* `ProtectedSessionStorage`

<span data-ttu-id="8d3f9-214">选择取决于要使用的备份存储区。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-214">The choice depends on which backing store you wish to use.</span></span> <span data-ttu-id="8d3f9-215">在下面的示例中`sessionStorage` , 使用:</span><span class="sxs-lookup"><span data-stu-id="8d3f9-215">In the following example, `sessionStorage` is used:</span></span>

```cshtml
@using Microsoft.AspNetCore.ProtectedBrowserStorage
@inject ProtectedSessionStorage ProtectedSessionStore
```

<span data-ttu-id="8d3f9-216">语句可以放在 _Imports 文件中, 而不是组件中。 `@using`</span><span class="sxs-lookup"><span data-stu-id="8d3f9-216">The `@using` statement can be placed into an *_Imports.razor* file instead of in the component.</span></span> <span data-ttu-id="8d3f9-217">使用 *_Imports*文件可使命名空间可用于应用或整个应用的更大段。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-217">Use of the *_Imports.razor* file makes the namespace available to larger segments of the app or the whole app.</span></span>

<span data-ttu-id="8d3f9-218">若要`currentCount` `ProtectedSessionStore.SetAsync`在项目模板的`Counter`组件中保留该值, 请将方法`IncrementCount`修改为使用:</span><span class="sxs-lookup"><span data-stu-id="8d3f9-218">To persist the `currentCount` value in the `Counter` component of the project template, modify the `IncrementCount` method to use `ProtectedSessionStore.SetAsync`:</span></span>

```csharp
private async Task IncrementCount()
{
    currentCount++;
    await ProtectedSessionStore.SetAsync("count", currentCount);
}
```

<span data-ttu-id="8d3f9-219">在更大、更真实的应用中, 每个字段的存储都是不太可能的方案。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-219">In larger, more realistic apps, storage of individual fields is an unlikely scenario.</span></span> <span data-ttu-id="8d3f9-220">应用更有可能存储包含复杂状态的整个模型对象。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-220">Apps are more likely to store entire model objects that include complex state.</span></span> <span data-ttu-id="8d3f9-221">`ProtectedSessionStore`自动序列化并反序列化 JSON 数据。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-221">`ProtectedSessionStore` automatically serializes and deserializes JSON data.</span></span>

<span data-ttu-id="8d3f9-222">在上面的代码示例中, `currentCount`数据存储为`sessionStorage['count']`用户浏览器中的。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-222">In the preceding code example, the `currentCount` data is stored as `sessionStorage['count']` in the user's browser.</span></span> <span data-ttu-id="8d3f9-223">数据不会以纯文本形式存储, 而是使用 ASP.NET Core 的[数据保护](xref:security/data-protection/introduction)进行保护。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-223">The data isn't stored in plaintext but rather is protected using ASP.NET Core's [Data Protection](xref:security/data-protection/introduction).</span></span> <span data-ttu-id="8d3f9-224">如果`sessionStorage['count']`在浏览器的开发人员控制台中计算, 则可以查看加密的数据。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-224">The encrypted data can be seen if `sessionStorage['count']` is evaluated in the browser's developer console.</span></span>

<span data-ttu-id="8d3f9-225">若要在`currentCount`用户以后返回`Counter`到组件时恢复数据 (包括在全新线路上时), 请使用`ProtectedSessionStore.GetAsync`:</span><span class="sxs-lookup"><span data-stu-id="8d3f9-225">To recover the `currentCount` data if the user returns to the `Counter` component later (including if they're on an entirely new circuit), use `ProtectedSessionStore.GetAsync`:</span></span>

```csharp
protected override async Task OnInitializedAsync()
{
    currentCount = await ProtectedSessionStore.GetAsync<int>("count");
}
```

<span data-ttu-id="8d3f9-226">如果组件的参数包括导航状态, 请调用`ProtectedSessionStore.GetAsync` `OnParametersSetAsync`, 并将结果分配到, `OnInitializedAsync`而不是。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-226">If the component's parameters include navigation state, call `ProtectedSessionStore.GetAsync` and assign the result in `OnParametersSetAsync`, not `OnInitializedAsync`.</span></span> <span data-ttu-id="8d3f9-227">`OnInitializedAsync`仅在首次实例化组件时调用一次。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-227">`OnInitializedAsync` is only called one time when the component is first instantiated.</span></span> <span data-ttu-id="8d3f9-228">`OnInitializedAsync`如果用户在同一页面上保留到不同的 URL, 则不会再次调用。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-228">`OnInitializedAsync` isn't called again later if the user navigates to a different URL while remaining on the same page.</span></span>

> [!WARNING]
> <span data-ttu-id="8d3f9-229">本部分中的示例仅适用于服务器未启用预呈现功能的情况。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-229">The examples in this section only work if the server doesn't have prerendering enabled.</span></span> <span data-ttu-id="8d3f9-230">启用预呈现后, 会生成错误, 如下所示:</span><span class="sxs-lookup"><span data-stu-id="8d3f9-230">With prerendering enabled, an error is generated similar to:</span></span>
>
> > <span data-ttu-id="8d3f9-231">此时无法发出 JavaScript 互操作调用。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-231">JavaScript interop calls cannot be issued at this time.</span></span> <span data-ttu-id="8d3f9-232">这是因为该组件正在预呈现。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-232">This is because the component is being prerendered.</span></span>
>
> <span data-ttu-id="8d3f9-233">禁用预呈现或添加其他代码以使用预呈现。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-233">Either disable prerendering or add additional code to work with prerendering.</span></span> <span data-ttu-id="8d3f9-234">若要了解有关编写适用于预呈现的代码的详细信息, 请参阅[处理预呈现](#handle-prerendering)部分。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-234">To learn more about writing code that works with prerendering, see the [Handle prerendering](#handle-prerendering) section.</span></span>

### <a name="handle-the-loading-state"></a><span data-ttu-id="8d3f9-235">处理加载状态</span><span class="sxs-lookup"><span data-stu-id="8d3f9-235">Handle the loading state</span></span>

<span data-ttu-id="8d3f9-236">由于浏览器存储是异步的 (通过网络连接进行访问), 因此, 在加载数据之前始终有一段时间, 并且可供组件使用。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-236">Since browser storage is asynchronous (accessed over a network connection), there's always a period of time before the data is loaded and available for use by a component.</span></span> <span data-ttu-id="8d3f9-237">为获得最佳结果, 请在加载正在进行时呈现加载状态消息, 而不是显示空数据或默认数据。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-237">For the best results, render a loading-state message while loading is in progress instead of displaying blank or default data.</span></span>

<span data-ttu-id="8d3f9-238">一种方法是跟踪数据是否为`null` (仍在加载)。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-238">One approach is to track whether the data is `null` (still loading) or not.</span></span> <span data-ttu-id="8d3f9-239">在默认`Counter`组件中, 计数保存`int`在中。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-239">In the default `Counter` component, the count is held in an `int`.</span></span> <span data-ttu-id="8d3f9-240">将问号 (`?`) 添加到类型 (`int`), 可以为null:`currentCount`</span><span class="sxs-lookup"><span data-stu-id="8d3f9-240">Make `currentCount` nullable by adding a question mark (`?`) to the type (`int`):</span></span>

```csharp
private int? currentCount;
```

<span data-ttu-id="8d3f9-241">如果只是加载数据, 请选择仅显示这些元素, 而不是无条件地显示 "计数和**增量**" 按钮:</span><span class="sxs-lookup"><span data-stu-id="8d3f9-241">Instead of unconditionally displaying the count and **Increment** button, choose to display these elements only if the data is loaded:</span></span>

```cshtml
@if (currentCount.HasValue)
{
    <p>Current count: <strong>@currentCount</strong></p>

    <button @onclick="IncrementCount">Increment</button>
}
else
{
    <p>Loading...</p>
}
```

### <a name="handle-prerendering"></a><span data-ttu-id="8d3f9-242">处理预呈现</span><span class="sxs-lookup"><span data-stu-id="8d3f9-242">Handle prerendering</span></span>

<span data-ttu-id="8d3f9-243">预呈现期间:</span><span class="sxs-lookup"><span data-stu-id="8d3f9-243">During prerendering:</span></span>

* <span data-ttu-id="8d3f9-244">与用户浏览器之间的交互连接不存在。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-244">An interactive connection to the user's browser doesn't exist.</span></span>
* <span data-ttu-id="8d3f9-245">浏览器还没有可在其中运行 JavaScript 代码的页面。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-245">The browser doesn't yet have a page in which it can run JavaScript code.</span></span>

<span data-ttu-id="8d3f9-246">`localStorage`或`sessionStorage`在预呈现期间不可用。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-246">`localStorage` or `sessionStorage` aren't available during prerendering.</span></span> <span data-ttu-id="8d3f9-247">如果组件尝试与存储交互, 则会生成类似于以下内容的错误:</span><span class="sxs-lookup"><span data-stu-id="8d3f9-247">If the component attempts to interact with storage, an error is generated similar to:</span></span>

> <span data-ttu-id="8d3f9-248">此时无法发出 JavaScript 互操作调用。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-248">JavaScript interop calls cannot be issued at this time.</span></span> <span data-ttu-id="8d3f9-249">这是因为该组件正在预呈现。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-249">This is because the component is being prerendered.</span></span>

<span data-ttu-id="8d3f9-250">解决错误的一种方法是禁用预呈现。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-250">One way to resolve the error is to disable prerendering.</span></span> <span data-ttu-id="8d3f9-251">如果应用大量使用基于浏览器的存储, 则这通常是最佳选择。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-251">This is usually the best choice if the app makes heavy use of browser-based storage.</span></span> <span data-ttu-id="8d3f9-252">预呈现增加了复杂性, 因此不会给应用带来好处, 因为应用无法`localStorage`将`sessionStorage`任何有用的内容预呈现到或可用。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-252">Prerendering adds complexity and doesn't benefit the app because the app can't prerender any useful content until `localStorage` or `sessionStorage` are available.</span></span>

<span data-ttu-id="8d3f9-253">禁用预呈现:</span><span class="sxs-lookup"><span data-stu-id="8d3f9-253">To disable prerendering:</span></span>

1. <span data-ttu-id="8d3f9-254">打开*Pages/_Host*文件并删除对`Html.RenderComponentAsync`的调用。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-254">Open the *Pages/_Host.cshtml* file and remove the call to `Html.RenderComponentAsync`.</span></span>
1. <span data-ttu-id="8d3f9-255">打开文件, 并将`endpoints.MapBlazorHub<App>("app")`对的调用`endpoints.MapBlazorHub()`替换为。 `Startup.cs`</span><span class="sxs-lookup"><span data-stu-id="8d3f9-255">Open the `Startup.cs` file and replace the call to `endpoints.MapBlazorHub()` with `endpoints.MapBlazorHub<App>("app")`.</span></span> <span data-ttu-id="8d3f9-256">`App`根组件的类型。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-256">`App` is the type of the root component.</span></span> <span data-ttu-id="8d3f9-257">`"app"`指定根组件位置的 CSS 选择器。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-257">`"app"` is a CSS selector specifying the location for the root component.</span></span>

<span data-ttu-id="8d3f9-258">对于不使用`localStorage`或`sessionStorage`的其他页, 预呈现可能很有用。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-258">Prerendering might be useful for other pages that don't use `localStorage` or `sessionStorage`.</span></span> <span data-ttu-id="8d3f9-259">要使预呈现功能保持启用状态, 请推迟加载操作, 直到浏览器连接到线路。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-259">To keep prerendering enabled, defer the loading operation until the browser is connected to the circuit.</span></span> <span data-ttu-id="8d3f9-260">下面是存储计数器值的示例:</span><span class="sxs-lookup"><span data-stu-id="8d3f9-260">The following is an example for storing a counter value:</span></span>

```cshtml
@using Microsoft.AspNetCore.ProtectedBrowserStorage
@inject ProtectedLocalStorage ProtectedLocalStore
@inject IComponentContext ComponentContext

... rendering code goes here ...

@code {
    private int? currentCount;
    private bool isWaitingForConnection;

    protected override async Task OnInitializedAsync()
    {
        if (ComponentContext.IsConnected)
        {
            // It looks like the app isn't prerendering, so the data can be
            // immediately loaded from browser storage.
            await LoadStateAsync();
        }
        else
        {
            // Prerendering is in progress, so the app defers the load operation
            // until later.
            isWaitingForConnection = true;
        }
    }

    protected override async Task OnAfterRenderAsync()
    {
        // By this stage, the client has connected back to the server, and
        // browser services are available. If the app didn't load the data earlier,
        // the app should do so now and then trigger a new render.
        if (isWaitingForConnection)
        {
            isWaitingForConnection = false;
            await LoadStateAsync();
            StateHasChanged();
        }
    }

    private async Task LoadStateAsync()
    {
        currentCount = await ProtectedLocalStore.GetAsync<int>("prerenderedCount");
    }

    private async Task IncrementCount()
    {
        currentCount++;
        await ProtectedSessionStore.SetAsync("count", currentCount);
    }
}
```

### <a name="factor-out-the-state-preservation-to-a-common-location"></a><span data-ttu-id="8d3f9-261">将状态保存分解到常见位置</span><span class="sxs-lookup"><span data-stu-id="8d3f9-261">Factor out the state preservation to a common location</span></span>

<span data-ttu-id="8d3f9-262">如果许多组件依赖于基于浏览器的存储, 则多次重新实现状态提供程序代码会创建代码重复。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-262">If many components rely on browser-based storage, re-implementing state provider code many times creates code duplication.</span></span> <span data-ttu-id="8d3f9-263">避免代码重复的一个选择是创建一个用于封装状态提供程序逻辑的*状态提供程序父组件*。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-263">One option for avoiding code duplication is to create a *state provider parent component* that encapsulates the state provider logic.</span></span> <span data-ttu-id="8d3f9-264">子组件可以使用永久性数据, 而不考虑状态持久性机制。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-264">Child components can work with persisted data without regard to the state persistence mechanism.</span></span>

<span data-ttu-id="8d3f9-265">在以下`CounterStateProvider`组件示例中, 将保留计数器数据:</span><span class="sxs-lookup"><span data-stu-id="8d3f9-265">In the following example of a `CounterStateProvider` component, counter data is persisted:</span></span>

```cshtml
@using Microsoft.AspNetCore.ProtectedBrowserStorage
@inject ProtectedSessionStorage ProtectedSessionStore

@if (hasLoaded)
{
    <CascadingValue Value="@this">
        @ChildContent
    </CascadingValue>
}
else
{
    <p>Loading...</p>
}

@code {
    private bool hasLoaded;

    [Parameter]
    public RenderFragment ChildContent { get; set; }

    public int CurrentCount { get; set; }

    protected override async Task OnInitializedAsync()
    {
        CurrentCount = await ProtectedSessionStore.GetAsync<int>("count");
        hasLoaded = true;
    }

    public async Task SaveChangesAsync()
    {
        await ProtectedSessionStore.SetAsync("count", CurrentCount);
    }
}
```

<span data-ttu-id="8d3f9-266">`CounterStateProvider`组件通过在加载完成之前不呈现其子内容来处理加载阶段。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-266">The `CounterStateProvider` component handles the loading phase by not rendering its child content until loading is complete.</span></span>

<span data-ttu-id="8d3f9-267">若要使用`CounterStateProvider`该组件, 请围绕需要访问计数器状态的任何其他组件环绕组件实例。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-267">To use the `CounterStateProvider` component, wrap an instance of the component around any other component that requires access to the counter state.</span></span> <span data-ttu-id="8d3f9-268">若要使某个应用中的所有组件都可以访问该状态`CounterStateProvider` , 请在`Router` `App`组件 (*app.config*) 中围绕的组件:</span><span class="sxs-lookup"><span data-stu-id="8d3f9-268">To make the state accessible to all components in an app, wrap the `CounterStateProvider` component around the `Router` in the `App` component (*App.razor*):</span></span>

```cshtml
<CounterStateProvider>
    <Router AppAssembly="typeof(Startup).Assembly">
        ...
    </Router>
</CounterStateProvider>
```

<span data-ttu-id="8d3f9-269">包装的组件接收并可以修改持久化计数器状态。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-269">Wrapped components receive and can modify the persisted counter state.</span></span> <span data-ttu-id="8d3f9-270">以下`Counter`组件实现模式:</span><span class="sxs-lookup"><span data-stu-id="8d3f9-270">The following `Counter` component implements the pattern:</span></span>

```cshtml
@page "/counter"

<p>Current count: <strong>@CounterStateProvider.CurrentCount</strong></p>

<button @onclick="IncrementCount">Increment</button>

@code {
    [CascadingParameter]
    private CounterStateProvider CounterStateProvider { get; set; }

    private async Task IncrementCount()
    {
        CounterStateProvider.CurrentCount++;
        await CounterStateProvider.SaveChangesAsync();
    }
}
```

<span data-ttu-id="8d3f9-271">前面的组件无需与进行交互`ProtectedBrowserStorage`, 也不会处理 "正在加载" 阶段。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-271">The preceding component isn't required to interact with `ProtectedBrowserStorage`, nor does it deal with a "loading" phase.</span></span>

<span data-ttu-id="8d3f9-272">若要处理前面所述的预`CounterStateProvider`呈现, 可以修改以使使用计数器数据的所有组件自动使用预呈现。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-272">To deal with prerendering as described earlier, `CounterStateProvider` can be amended so that all of the components that consume the counter data automatically work with prerendering.</span></span> <span data-ttu-id="8d3f9-273">有关详细信息, 请参阅[处理预呈现](#handle-prerendering)部分。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-273">See the [Handle prerendering](#handle-prerendering) section for details.</span></span>

<span data-ttu-id="8d3f9-274">通常, 建议使用*状态提供程序父组件*模式:</span><span class="sxs-lookup"><span data-stu-id="8d3f9-274">In general, *state provider parent component* pattern is recommended:</span></span>

* <span data-ttu-id="8d3f9-275">在许多其他组件中使用状态。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-275">To consume state in many other components.</span></span>
* <span data-ttu-id="8d3f9-276">如果只有一个顶级状态对象要保持, 则为。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-276">If there's just one top-level state object to persist.</span></span>

<span data-ttu-id="8d3f9-277">若要保留许多不同的状态对象并在不同的位置使用不同的对象子集, 最好避免在全局处理状态的加载和保存。</span><span class="sxs-lookup"><span data-stu-id="8d3f9-277">To persist many different state objects and consume different subsets of objects in different places, it's better to avoid handling the loading and saving of state globally.</span></span>