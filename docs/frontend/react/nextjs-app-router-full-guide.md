---
theme: channing-cyan
date: 2026-04-17
---

# Next.js App Router：从 SSR/CSR 到缓存与 Server Action

前言：本文把 Next.js App Router 的高频知识进行总结，给出简洁说明和示例，可直接当作速查清单与复盘资料。

## 1. SSR 和 CSR：先建立整体认知

### CSR（Client Side Rendering）

- 浏览器先拿到基础 HTML，再下载 JS 后在客户端渲染页面。
- 首屏通常依赖 JS 执行，页面交互能力强，适合高交互后台系统。

### SSR（Server Side Rendering）

- 服务端先把页面渲染成 HTML 返回给浏览器，客户端再进行水合（Hydration）。
- 首屏内容到达更早，对 SEO 和社交分享更友好。

### SSR 的核心好处

- SEO 友好，搜索引擎更容易拿到完整内容。
- 可在服务端预处理数据，减少页面加载后的额外请求。
- 能规避很多浏览器端跨域限制（由服务端统一请求外部 API）。
- 便于聚合多后端服务（例如微服务 + GraphQL 网关），前端调用更简单。
- 更好的社交平台预览（OG/meta tags 能在首个 HTML 中返回）。
- 安全性更高：敏感逻辑和密钥可留在服务端执行。

## 2. App Router 文件层级：layout、template、page

App Router 的渲染层级通常是：

`layout > template > page`

### layout

- 支持根 layout 和嵌套 layout。
- 路由切换时，layout 默认会复用，内部状态可保留。

### template

- 每次导航都会重新创建实例。
- 路由切换后其局部状态不会保留。

简单理解：

- 希望状态保留，用 `layout`。
- 希望切换即重置，用 `template`。

## 3. 路由、动态路由、路由组

### 文件路由

- 目录即路由，`page.tsx` 对应该层页面。

### 动态路由

- `[slug]`：单段动态参数。
- `[...slug]`：Catch-all，匹配多段。
- `[[...slug]]`：可选 Catch-all，可匹配空路径。

在 Next.js 15+ 的服务端组件中，`params` 通常按 `Promise` 处理后再使用。

```tsx
// app/blog/[slug]/page.tsx
export default async function BlogDetail({
  params,
}: {
  params: Promise<{ slug: string }>;
}) {
  const { slug } = await params;
  return <div>slug: {slug}</div>;
}
```

### 路由组

- 使用 `(groupName)` 目录分组。
- 只做组织结构，不影响 URL。

最小目录示例：

```txt
app/
  (marketing)/
    about/page.tsx   -> /about
  (shop)/
    cart/page.tsx    -> /cart
```

## 4. 平行路由（Parallel Routes）

平行路由可以理解为“多插槽并行渲染”，和 Vue 插槽思路接近。

### 语法

- 以 `@` 开头的目录，如 `@modal`、`@dashboard`。
- 默认插槽相当于 `@children`。

### 软导航 vs 硬导航

- 软导航：通过 `<Link>` 跳转，会尽量复用已有 UI 状态。
- 硬导航：浏览器刷新或直接输入 URL，会重新加载整页。

### default.tsx 的作用

给某个平行路由槽位提供兜底内容，避免导航时因为缺少对应子路由而出现不一致或空白。

最小目录示例：

```txt
app/
  dashboard/
    layout.tsx
    @analytics/
      page.tsx
      default.tsx
    @team/
      page.tsx
      default.tsx
```

## 5. 拦截路由（Intercepting Routes）

语法：`(..)`。

典型场景：

- 在列表页点击图片后，以弹窗展示详情（仍保留列表上下文）。
- 把详情页链接分享给别人时，对方可直接打开完整详情页。

这类体验通常会配合“平行路由 + 拦截路由”实现。

最小目录示例（列表页弹窗 + 独立详情页）：

```txt
app/
  feed/
    page.tsx
    @modal/
      (..)photo/[id]/page.tsx   // 从 feed 拦截进入弹窗
  photo/[id]/page.tsx           // 直接访问时展示完整详情页
```

## 6. 在客户端获取路径信息

这一块最容易混淆，先记住一句话：

- `useParams` 读“路径动态段”（如 `/posts/[id]`）。
- `useSearchParams` 读“查询参数”（如 `?tab=comment`）。
- `usePathname` 读“当前路径字符串”（不含查询串）。

假设当前 URL 是：`/posts/123?tab=comment&from=home`

- `useParams()` -> `{ id: "123" }`
- `useSearchParams().get("tab")` -> `"comment"`
- `usePathname()` -> `"/posts/123"`

```tsx
"use client";

import { useParams, usePathname, useSearchParams } from "next/navigation";

export default function Demo() {
  const { id } = useParams<{ id: string }>();
  const pathname = usePathname();
  const searchParams = useSearchParams();
  const keyword = searchParams.get("keyword");
  const tab = searchParams.get("tab");

  return (
    <div>
      <p>id: {id}</p>
      <p>pathname: {pathname}</p>
      <p>keyword: {keyword}</p>
      <p>tab: {tab}</p>
    </div>
  );
}
```

## 7. 页面跳转方式

### `<Link>`

- 增强版 `a` 标签，支持预取（prefetch）。
- 推荐用于常规导航。

### `useRouter()`

- 编程式导航：`router.push()`、`router.replace()`、`router.back()`、`router.refresh()`。
- 适合事件回调中的跳转控制。

```tsx
"use client";

import Link from "next/link";
import { useRouter } from "next/navigation";

export default function NavDemo() {
  const router = useRouter();
  return (
    <>
      <Link href="/posts">去列表页</Link>
      <button onClick={() => router.push("/posts/1")}>查看详情</button>
      <button onClick={() => router.refresh()}>刷新当前路由数据</button>
    </>
  );
}
```

## 8. Metadata：静态与动态

App Router 支持通过 `metadata` 或 `generateMetadata` 配置 SEO 信息。

- 静态 metadata：页面级固定标题、描述。
- 动态 metadata：可根据路由参数或接口数据动态生成。

```tsx
// 静态 metadata
export const metadata = {
  title: "文章列表",
  description: "博客文章列表页",
};
```

```tsx
// 动态 metadata
export async function generateMetadata({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;
  return {
    title: `文章-${id}`,
  };
}
```

## 9. 404 处理：全局与局部

### 全局 404

- 根目录 `not-found.tsx`，兜底所有未匹配页面。

### 局部 404

- 某个路由段下也可放置 `not-found.tsx`。
- 业务中手动触发：

```ts
import { notFound } from "next/navigation";

notFound();
```

## 10. 路由处理程序（Route Handlers）

文件约定：`app/api/**/route.ts`。

可实现 RESTful 风格接口，也支持动态路由参数。

示例：

```ts
// app/api/posts/[id]/route.ts
export async function GET() {}
export async function PATCH() {}
export async function DELETE() {}
```

## 11. GET 缓存何时失效（常见误区）

以下场景通常不会走静态缓存（或会触发动态渲染）：

- 在 Route Handler 中读取 `Request` 里的动态信息（如 query、cookie、header）并参与响应计算。
- 使用非 GET 方法（POST/PUT/PATCH/DELETE）。
- 使用动态函数（如 `cookies()`、`headers()`）。
- 显式通过 `dynamic`、`revalidate`、`cache` 配置动态策略。

建议：不要过度依赖“默认缓存行为”，在关键接口上显式写清缓存策略。

## 12. Middleware（请求拦截层），现称 Proxy

常见写法：

```ts
export const config = {
  matcher: "/about/:path*",
};
```

应用场景：

- 登录态校验、权限控制。
- 登录/退出后的重定向策略。
- 国际化前缀处理等。

它的职责更像应用入口处的“网关/代理层”，但工程上仍按 `middleware.ts` 约定使用。

```ts
// middleware.ts
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

export function middleware(request: NextRequest) {
  const token = request.cookies.get("token")?.value;
  if (!token) return NextResponse.redirect(new URL("/login", request.url));
  return NextResponse.next();
}
```

## 13. 客户端组件 vs 服务端组件

### 什么时候必须用客户端组件（`"use client"`）

- 需要事件处理（点击、输入等）。
- 需要 React Hooks（`useState`、`useEffect` 等）。
- 需要浏览器 API（`window`、`localStorage`）。
- 使用依赖 state/effect/browser API 的自定义 Hook。
- 使用类组件。

### 客户端组件执行过程

- 服务端预渲染出初始 HTML（可选）。
- 客户端进行水合并接管交互。
- 后续状态变化在客户端重新渲染。

### 服务端组件（RSC）

- 仅运行在服务端。
- 可在构建时静态生成，也可在请求时动态渲染。
- 更适合数据读取、拼装和安全逻辑处理。

### 组合注意事项

- 服务端组件可以直接使用客户端组件。
- 客户端组件不能直接 import 服务端组件；常见做法是通过 `children` 组合。
- 第三方库若内部使用浏览器 API，需包一层客户端组件再在服务端树中使用。
- Context Provider 放在根时，通常也需要客户端包装层。

## 14. 服务端组件的数据共享

在 RSC 中可直接 `fetch` 读取数据并复用缓存。

- Next.js 14：`fetch` 默认更偏向可缓存策略（视场景而定）。
- Next.js 15+：默认行为改为更偏动态（常见理解是默认不缓存），迁移时要显式声明缓存策略。

迁移建议：把缓存意图写清楚，不依赖“默认行为”。

## 15. 服务端组件渲染策略：静态与动态

### 静态渲染

- 适合内容相对稳定页面。
- 支持 `revalidate` 进行按时间增量更新（ISR）。

### 动态渲染

触发条件常见有：

- 使用动态函数：`cookies()`、`headers()`、`searchParams`。
- 使用未缓存的 `fetch`。

## 16. fetch

```ts
await fetch(url, {
  next: { revalidate: 3000, tags: ["post-list"] },
});
```

### 常见刷新策略

- 基于时间：`revalidate`。
- 按路径：`revalidatePath("/posts")`。
- 按标签：`revalidateTag("post-list")`。

`revalidateTag` 适合批量失效同类查询，通常比路径粒度更灵活。

## 17. 四类缓存机制

### 1）请求级缓存（Request Memoization）

- React 层能力，同一次请求内去重相同 fetch。

### 2）数据缓存（Data Cache）

- Next.js 层能力，跨请求复用数据。
- 受 `cache`、`revalidate`、`tags` 影响。

### 3）全路由缓存（Full Route Cache）

- 面向静态路由产物缓存。

### 4）客户端路由缓存（Router Cache）

- 基于 App Router 在客户端缓存段数据，提升导航速度。
- 页面刷新会清空这类缓存。

补充：

- `<Link>` 会触发预取，常见默认缓存窗口：静态页更长、动态页更短。
- `router.refresh()` 会请求新数据并更新当前路由树。

## 18. Server Action / Server Function

使用方式：`"use server"`。

### 两种常见声明级别

- 函数级别：在具体函数体前声明。
- 模块级别：文件顶部声明，文件内导出函数都在服务端执行。

### 典型应用

- 表单提交写库。
- 数据变更后触发 `revalidatePath` / `revalidateTag`。
- 将客户端“上提请求”改为服务端执行，减少 API 样板代码。

```ts
// app/actions.ts
"use server";

import { revalidatePath } from "next/cache";

export async function createPost(formData: FormData) {
  const title = String(formData.get("title") || "");
  // await db.post.create({ data: { title } });
  revalidatePath("/posts");
}
```

```tsx
// app/posts/new/page.tsx
import { createPost } from "@/app/actions";

export default function NewPostPage() {
  return (
    <form action={createPost}>
      <input name="title" placeholder="请输入标题" />
      <button type="submit">提交</button>
    </form>
  );
}
```

## 19. 类型校验：推荐 Zod

在 Server Action 或 Route Handler 中，建议先做参数校验再执行业务逻辑：

```ts
import { z } from "zod";

const CreatePostSchema = z.object({
  title: z.string().min(1, "标题不能为空"),
  content: z.string().min(1, "内容不能为空"),
});
```

优势：

- 运行时校验 + TS 类型推导统一。
- 错误信息可控，便于前后端协作。

## 20. Turbopack 与 React Compiler

### Turbopack 的价值

- 更快的本地构建与热更新反馈。
- 惰性打包，按需处理模块。
- 增量计算与缓存，改动范围越小收益越明显。

### React Compiler（关注趋势）

- 目标是让 React 编译器自动优化部分渲染性能问题。
- 在真实项目中仍需配合良好组件边界和状态设计，不能把性能完全交给“黑盒优化”。

## 实战建议：如何把这些知识真正用起来

1. 先定渲染策略：页面是静态优先还是动态优先。  
2. 再定路由结构：普通路由、平行路由、拦截路由如何组合。  
3. 明确缓存策略：哪些数据走 `revalidate`，哪些走 tag 失效。  
4. 写操作优先 Server Action：并在动作后精确触发缓存失效。  
5. 最后补齐类型和异常处理：Zod + 404 + 边界兜底。  