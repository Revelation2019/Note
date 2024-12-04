# Gatsby 路由重定向

在一些项目中，常常需要实现路径重定向功能，例如，当用户访问 `http://localhost:8080/` 时，自动重定向到首页路由 `http://localhost:8080/home`。这种功能可以为用户提供更好的导航体验。接下来，让我们看看如何在 `Gatsby` 静态站点项目中实现这一点。

## @reach/router

`@reach/router` 是一个轻量级的、专注于用户体验和可访问性的现代 `JavaScript` 路由库。在 `Gatsby` 中，`@reach/router` 已被内置用来处理客户端路由。`@reach/router` 的 `Redirect` 组件可以用来在客户端应用中实现路由重定向。

在 `Gatsby` 中，`src/pages` 目录下的文件会自动转换成静态 `HTML` 页面，这样的架构可确保初次加载时的快速响应。而使用 `@reach/router` 提供的客户端路由机制——通过 `history` 模式，`URL` 的变化不会导致页面重载。相反，`Router` 组件会捕获这些变化，动态地重新计算并渲染匹配的路由组件，从而实现平滑的导航体验。

```tsx
// src/pages/index.tsx
import { Router, Redirect, Location, RouteComponentProps } from '@reach/router';
import { FC, Fragment } from 'react';

const Home: FC<RouteComponentProps> = () => <div>Home Page</div>;
const About: FC<RouteComponentProps> = () => <div>About Page</div>;
const Blog: FC<RouteComponentProps> = () => <div>Blog Page</div>;

const IndexPage = () => {
  return (
    <Location>
      {({ location }) => {
        return (
          <Fragment>
            <Redirect from="/" to="/home" noThrow />
            <Router location={location}>
              <Home path="/home" />
              <Blog path="/blog" />
              <About path="/about" />
            </Router>
          </Fragment>
        );
      }}
    </Location>
  );
};

export default IndexPage;
```

尽管我们希望在用户访问 `http://localhost:8080/` 时能够自动重定向到首页路由 `http://localhost:8080/home`，现实情况是页面却跳转到了 `404` 错误页面。

![Img](./FILES/Gatsby%20路由重定向.md/img-20241202173817.gif)

从断点调试结果可以看出，实际上 `Redirect` 的重定向操作是成功执行的，并且在过程中确实渲染了 `Home` 组件。然而，不知为何，最后页面又跳转到了 `404` 错误页面。

以下展示了在监听到路由变化后，重新匹配并渲染相应的路由组件的过程。

```js
const Location = ({ children }) => {
  return <LocationProvider>{children}</LocationProvider>;
};

class LocationProvider extends React.Component {
  state = {
    context: this.getContext(),
    refs: { unlisten: null }
  };

  getContext() {
    let {
      props: {
        history: { navigate, location }
      }
    } = this;
    return { navigate, location };
  }

  componentDidMount() {
    // 监听路由变化
    history.listen(() => {
      Promise.resolve().then(() => {
        requestAnimationFrame(() => {
          if (!this.unmounted) {
            this.setState(() => ({ context: this.getContext() }));
          }
        });
      });
    });
  }

  render() {
    let {
      state: { context },
      props: { children }
    } = this;
    return (
      // 向子组件提供上下文 location
      <LocationContext.Provider value={context}>
        {typeof children === "function" ? children(context) : children || null}
      </LocationContext.Provider>
    );
  }
}

const Router = (props) => <RouterImpl {...props} />;

class RouterImpl extends React.PureComponent {
  render() {
    // 获取子组件数组
    let routes = React.Children.toArray(children).reduce((array, child) => {
      const routes = createRoute(basepath)(child);
      return array.concat(routes);
    }, []);
    let { pathname } = location;
    // 匹配路由
    let match = pick(routes, pathname);

    if (match) {
      let clone = React.cloneElement(
        match.route.element,
        props,
        element.props.children ? (
          <Router location={location} primary={primary}>
            {element.props.children}
          </Router>
        ) : (
          undefined
        )
      );

      return clone;
    }

    return null;
  }
}
```

以下是 `Redirect` 组件的实现逻辑。在 `componentDidMount` 阶段，该组件会调用 `navigate` 方法来执行路由跳转。

```js
const Redirect = (props) => <RedirectImpl {...locationContext} baseuri={baseuri} {...props} />;

class RedirectImpl extends React.Component {
  componentDidMount() {
    let {
      props: {
        navigate,
        to,
        from,
        replace = true,
        state,
        noThrow,
        baseuri,
        ...props
      }
    } = this;
    Promise.resolve().then(() => {
      let resolvedTo = resolve(to, baseuri);
      navigate(insertParams(resolvedTo, props), { replace, state });
    });
  }

  render() {
    let {
      props: { navigate, to, from, replace, state, noThrow, baseuri, ...props }
    } = this;
    let resolvedTo = resolve(to, baseuri);
    if (!noThrow) redirectTo(insertParams(resolvedTo, props));
    return null;
  }
}
```

`navigate` 方法用于路由跳转时，会利用 `replaceState` 或 `pushState` 这两个 `API` 进行操作。这些 `API` 更改 `pathname` 的过程中不会导致页面重新加载。随后，`LocationProvider` 组件监听到路径的变化，会根据新的 `pathname` 重新匹配和渲染对应的路由组件，从而实现页面的无缝过渡。

```js
navigate(to, { state, replace = false } = {}) {
  if (typeof to === "number") {
    source.history.go(to);
  } else {
    state = { ...state, key: Date.now() + "" };
    try {
      if (transitioning || replace) {
        source.history.replaceState(state, null, to);
      } else {
        source.history.pushState(state, null, to);
      }
    } catch (e) {
      source.location[replace ? "replace" : "assign"](to);
    }
  }

  location = getLocation(source);
  transitioning = true;
  let transition = new Promise(res => (resolveTransition = res));
  listeners.forEach(listener => listener({ location, action: "PUSH" }));
  return transition;
}
```

从上述路由监听逻辑来看，与结果一致，路由重定向确实已经成功执行。

通过断点调试可以发现，页面最终跳转至 `404` 错误页面的原因在于 `Gatsby` 的构建过程。`Gatsby` 在构建过程中，会为 `src/pages` 目录中的每个文件生成相应的 `HTML`、`componentChunk` 文件以及页面数据文件 `page-data.json`。然而，由于 `src/pages` 目录下并不存在 `home.tsx` 文件，所以在构建过程中不会生成与之对应的页面资源。当 `Gatsby` 监听到路由变化时，它会尝试请求该路由对应的页面数据，即 `http://localhost:8888/page-data/home/page-data.json`。由于该请求返回了 404 响应，`Gatsby` 在开发环境下渲染了默认的 `404` 页面（不是用户自定义的 `404` 页面）。

详细过程如下：

（1）首先，在 `Gatsby` 提供的浏览器 `API` 钩子函数 `onClientEntry` 中，该钩子允许在客户端 `JavaScript` 入口文件执行之前运行指定代码。在这一阶段，`Gatsby` 会加载与当前路由页面相对应的资源（包括 `chunk` 文件和 `page-data.json`）。一旦这些资源成功加载，`Root` 组件便会被挂载。

```js
apiRunnerAsync(`onClientEntry`).then(() => {
  Promise.all([
    loader.loadPage(`/dev-404-page/`),
    loader.loadPage(`/404.html`),
    loader.loadPage(window.location.pathname + window.location.search),
  ]).then(() => {
    // 加载成功挂载 Root
    function App() { return <Root /> }
    function runRender() { renderer(<App />, rootElement) }

    const doc = document
    if (
      doc.readyState === `complete` ||
      (doc.readyState !== `loading` && !doc.documentElement.doScroll)
    ) {
      setTimeout(function () {
        runRender()
      }, 0)
    } else {
      const handler = function () {
        doc.removeEventListener(`DOMContentLoaded`, handler, false)
        window.removeEventListener(`load`, handler, false)

        runRender()
      }

      doc.addEventListener(`DOMContentLoaded`, handler, false)
      window.addEventListener(`load`, handler, false)
    }
  })
})
```

（2）在渲染 `Root` 组件时，该组件会根据 `location` 确认当前路由所对应的页面资源是否存在。如果页面资源存在，组件将渲染与该路由匹配的组件；如果页面资源不存在，则渲染开发环境下的默认 `404` 组件。

```js
class LocationHandler extends React.Component {
  render() {
    const { location } = this.props;

    // 如果存在页面资源
    if (!loader.isPageNotFound(location.pathname + location.search)) {
      return (
        <EnsureResources location={location}>
          {locationAndPageResources => (
            <Router basepath={__BASE_PATH__} location={location}>
              <RouteHandler
                path={encodeURI(
                  (
                    locationAndPageResources.pageResources.page.matchPath ||
                    locationAndPageResources.pageResources.page.path
                  ).split(`?`)[0]
                )}
                {...this.props}
                {...locationAndPageResources}
              />
            </Router>
          )}
        </EnsureResources>
      );
    }

    // 如果不存在页面资源，则渲染 404 页面
    const dev404PageResources = loader.loadPageSync(`/dev-404-page`);
    const real404PageResources = loader.loadPageSync(`/404.html`);
    const custom404 = real404PageResources && (
      <PageQueryStore {...this.props} pageResources={real404PageResources} />
    );

    return (
      <EnsureResources location={location}>
        {() => (
          <Router basepath={__BASE_PATH__} location={location}>
            <RouteHandler
              path={location.pathname}
              location={location}
              pageResources={dev404PageResources}
              custom404={custom404}
            />
          </Router>
        )}
      </EnsureResources>
    );
  }
}

// Location 内部有实现 LocationProvider 组件，监听路由变化，提供上下文
const Root = () => (
  <Location>
    {locationContext => <LocationHandler {...locationContext} />}
  </Location>
);

export default function AppRoot() {
  return (
    <FastRefreshOverlay>
      <SliceDataStore>
        <StaticQueryStore>
          <Root />
        </StaticQueryStore>
      </SliceDataStore>
    </FastRefreshOverlay>
  );
}
```

有些同学可能会疑惑，为什么在 `Promise.all` 中，即便 `loader.loadPage("/home")` 加载页面资源失败，`then` 函数仍然会被执行。这是因为内部机制已对 `/page-data/home/page-data.json` 返回 `404` 的情况进行了处理。会将 `/home` 的页面资源映射到 `/404` 的页面资源，并将其记录在 `this.pageDataDb` 的 `Map` 结构中。接下来，在渲染 `Root` 组件时，通过判断 `loader.isPageNotFound("/home")` 返回 `true`，从而决定渲染 `404` 页面。

![Img](./FILES/Gatsby%20路由重定向.md/img-20241203162335.png)

![Img](./FILES/Gatsby%20路由重定向.md/img-20241203163753.png)

![Img](./FILES/Gatsby%20路由重定向.md/img-20241203165541.png)

## createRedirect

在断点调试过程中发现，如果项目中配置了重定向（例如 `{fromPath: '/', toPath: '/home'}`），那么 `loader.isPageNotFound(fromPath)` 将转化为 `loader.isPageNotFound(toPath)` 的判断。这意味着系统会依据重定向的目标路径来判定页面的存在性。

```js
import redirects from "./redirects.json"

// Convert to a map for faster lookup in maybeRedirect()

const redirectMap = new Map()
const redirectIgnoreCaseMap = new Map()

redirects.forEach(redirect => {
  if (redirect.ignoreCase) {
    redirectIgnoreCaseMap.set(redirect.fromPath, redirect)
  } else {
    redirectMap.set(redirect.fromPath, redirect)
  }
})

export function maybeGetBrowserRedirect(pathname) {
  let redirect = redirectMap.get(pathname)
  if (!redirect) {
    redirect = redirectIgnoreCaseMap.get(pathname.toLowerCase())
  }
  return redirect
}

export const findPath = rawPathname => {
  const trimmedPathname = trimPathname(absolutify(rawPathname))
  if (pathCache.has(trimmedPathname)) {
    return pathCache.get(trimmedPathname)
  }

  // 存在重定向配置
  const redirect = maybeGetBrowserRedirect(rawPathname)
  if (redirect) {
    return findPath(redirect.toPath)
  }

  // 路由匹配，比如 "/" 会匹配 "/[...]"
  let foundPath = findMatchPath(trimmedPathname)

  if (!foundPath) {
    // 没有匹配上，就返回本身，比如 /home 并没有对应 ./home.tsx，也没有匹配的动态路由
    foundPath = cleanPath(rawPathname)
  }

  pathCache.set(trimmedPathname, foundPath)

  return foundPath
}
```

以下是重定向配置的过程：如果在 `gatsby-node.js` 中定义了 `createPages`，可以在创建页面的同时创建重定向。通过调用 `actions.createRedirect`，将生成一个类型为 `CREATE_REDIRECT` 的 `Action`。派发该 `Action` 后，会通过 `redirectsReducer` 将重定向配置存入内置的 `Redux` 状态对象中。最终，在 `CREATE_REDIRECT` 事件触发时，这些配置会被写入到 `.cache-dir/redirects.json` 文件中。

```js
/**
 * // Generally you create redirects while creating pages.
 * exports.createPages = ({ graphql, actions }) => {
 *   const { createRedirect } = actions
 *
 *   createRedirect({ fromPath: '/old-url/', toPath: '/new-url/', isPermanent: true })
 *   createRedirect({ fromPath: '/url/', toPath: '/zn-CH/url/', conditions: { language: 'zn' }})
 *   createRedirect({ fromPath: '/url/', toPath: '/en/url/', conditions: { language: ['ca', 'us'] }})
 *   createRedirect({ fromPath: '/url/', toPath: '/ca/url/', conditions: { country: 'ca' }})
 *   createRedirect({ fromPath: '/url/', toPath: '/en/url/', conditions: { country: ['ca', 'us'] }})
 *   createRedirect({ fromPath: '/not_so-pretty_url/', toPath: '/pretty/url/', statusCode: 200 })
 *
 *   // Create pages here
 * }
 */
actions.createRedirect = ({
  fromPath,
  isPermanent = false,
  redirectInBrowser = false,
  toPath,
  ignoreCase = true,
  ...rest
}) => {
  let pathPrefix = ``;
  if (store.getState().program.prefixPaths) {
    pathPrefix = store.getState().config.pathPrefix;
  }
  return {
    type: `CREATE_REDIRECT`,
    payload: {
      fromPath: maybeAddPathPrefix(fromPath, pathPrefix),
      isPermanent,
      ignoreCase,
      redirectInBrowser,
      toPath: maybeAddPathPrefix(toPath, pathPrefix),
      ...rest
    }
  };
};
```

以下是定义的 `Reducer`：

```js
const redirectsReducer = (state = [], action) => {
  switch (action.type) {
    case `CREATE_REDIRECT`:
      {
        const redirect = action.payload;

        // Add redirect only if it wasn't yet added to prevent duplicates
        if (!exists(redirect)) {
          add(redirect);
          state.push(redirect);
        }
        return state;
      }
    default:
      return state;
  }
};
```

以下是将重定向配置写入 `.cache-dir/redirects.json` 的过程：

![Img](./FILES/Gatsby%20路由重定向.md/img-20241203171957.png)

因此，即使在动态创建页面时设置了重定向，也无法解决问题，因为 `/home` 页面资源依然会被判断为不存在，最终仍然会显示 `404` 页面。

```js
// gatsby-node.js
export const createPages = async ({ graphql, actions }) => {
  const { createRedirect } = actions;

  createRedirect({
    fromPath: '/',
    toPath: '/home',
    isPermanent: true,
    redirectInBrowser: true,
    statusCode: 302,
  });
};
```

## Dynamic Routes

在 `Gatsby` 中，内部制定了一套路由约定，允许通过文件名配置动态路由。例如，`/src/pages/[...].tsx` 会生成 `[...].html` 和 `/page-data/[...]/page-data.json`，以及相应的组件块（`componentChunk`）。当访问 `/` 时，如果 `src/pages` 中没有 `index.tsx`，就不会生成 `index.html`，因此会匹配到 `[...]` 路由，并加载其页面资源。类似地，当访问 `/home` 时，如果 `src/pages` 中没有 `home.tsx`，也不会生成 `home.html`，从而会匹配到 `[...]` 路由，并加载其页面资源。从 `/` 重定向到 `/home` 的逻辑也是遵循这一规则。

![Img](./FILES/Gatsby%20路由重定向.md/img-20241203200441.png)

![Img](./FILES/Gatsby%20路由重定向.md/img-20241203200530.png)

因此，我们可以将 `[...].tsx` 视作一个路由分发层，负责处理路由的分发和重定向。

```tsx
// src/
// └── pages/
//     └── [...].tsx

// src/pages/[...].tsx
import { Router, Redirect, Location, RouteComponentProps } from '@reach/router';
import { FC, Fragment } from 'react';

const Home: FC<RouteComponentProps> = () => <div>Home Page</div>;
const About: FC<RouteComponentProps> = () => <div>About Page</div>;
const Blog: FC<RouteComponentProps> = () => <div>Blog Page</div>;

const IndexPage = () => {
  return (
    <Location>
      {
        ({ location }) => (
          <Router>
            <Redirect from="/" to="/home" noThrow />
            <Home path="/home"/>
            <About path="/about"/>
            <Blog path="/blog"/>
          </Router>
        )
      }
    </Location>
  );
}
```

为了避免将所有路由组件打包到一个单独的组件块（`component chunk`）中，可以使用 `React.Suspense` 和 `React.lazy` 来动态加载组件，实现代码拆分。

```jsx
const LazyComponent = React.lazy(() => import('./LazyComponent'));

function App() {
  return (
    <div>
      <React.Suspense fallback={<div>Loading...</div>}>
        <LazyComponent />
      </React.Suspense>
    </div>
  );
}
// src/pages/[...].tsx
import { Router, Redirect, Location, RouteComponentProps } from '@reach/router';
import { FC, Fragment, Suspense, lazy } from 'react';

// NOTE - 预加载资源，即较高优先级的加载
const LazyHomeComponent = lazy(
  () =>
    import(
      /* webpackChunkName: "component--src-pages-home" */
      /* webpackPreload: true */
      /* webpackMode: "lazy" */
      '../components/home'
    )
);
const Home: FC<RouteComponentProps> = () => (
  <Suspense fallback={<></>}>
    <LazyHomeComponent />
  </Suspense>
);

// NOTE - 浏览器空闲时提前加载
const LazyAboutComponent = lazy(
  () =>
    import(
      /* webpackChunkName: "component--src-pages-about" */
      /* webpackPrefetch: true */
      /* webpackMode: "lazy" */
      '../components/about'
    )
);
const About: FC<RouteComponentProps> = () => (
  <Suspense fallback={<></>}>
    <LazyAboutComponent />
  </Suspense>
);

const Blog: FC<RouteComponentProps> = () => <div>Blog Page</div>;

const IndexPage = () => {
  return (
    <Location>
      {
        ({ location }) => (
          <Router>
            <Redirect from="/" to="/home" noThrow />
            <Home path="/home"/>
            <About path="/about"/>
            <Blog path="/blog"/>
          </Router>
        )
      }
    </Location>
  );
}
```

![Img](./FILES/Gatsby%20路由重定向.md/img-20241204094837.png)

通过这种方式，只有在实际需要某个组件时，才会加载其相应的代码，从而实现更高效的路由管理和资源利用。

## Redirect plugin

### gatsby-plugin-meta-redirect

生成元重定向 `HTML` 文件，以便在任何静态文件主机上进行重定向。

```js
// In your gatsby-config.js
plugins: [
  `gatsby-plugin-meta-redirect` // make sure to put last in the array
];
```

当使用 `createRedirect` 操作创建重定向时，比如在以下示例中：

```js
export const createPages = async ({ graphql, actions }) => {
  const { createRedirect } = actions;

  createRedirect({
    fromPath: '/',
    toPath: '/home',
    isPermanent: true,
    redirectInBrowser: true,
    statusCode: 302,
  });
};
```

构建完成后将生成以下 `HTML` 文件：

```html
// /index.html
<meta http-equiv="refresh" content="0; URL='/home/'" />
```

当用户访问 `/` 时，加载的是生成的 `/index.html`，该页面通过 `meta` 刷新（`meta refresh`）重定向到目标页面 `/home/`，这会导致页面刷新。

**原理**

`gatsby-plugin-meta-redirect` 插件会在 `Gatsby` 的 `onPostBuild` 钩子函数中，从内置的 `Redux` 状态中获取已创建的重定向信息（存储在 `redirects` 状态中）。如果配置的重定向信息中不存在来源页面 (`fromPath`)，插件会自动创建该页面，并插入 `<meta http-equiv="refresh" content="0; URL='${toPath}'" />` 标签，以便实现重定向功能。

![Img](./FILES/Gatsby%20路由重定向.md/img-20241204193746.png)

`onPostBuild` 是 `Gatsby` 在构建过程中的一个生命周期钩子，它仅在构建过程结束后执行，也就是说，它只会在生产构建环境中运行。在开发环境下（即使用 `gatsby develop` 启动时），`onPostBuild` 并不会被调用。



### gatsby-plugin-client-side-redirect

与 `gatsby-plugin-meta-redirect` 类似，该插件也用于生成客户端重定向的 `HTML` 文件，以便在任何静态文件主机上实现重定向。不同之处在于，它通过在 `HTML` 中注入脚本 `<script>window.location.href="${url}"</script>` 的方式来执行重定向。

需要注意的是，无论使用哪个插件来实现重定向，都依赖于通过 `createRedirect` 方法生成的状态信息。
