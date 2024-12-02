# Gatsby 路由重定向

在一些项目中，常常需要实现路径重定向功能，例如，当用户访问 http://localhost:8080/ 时，自动重定向到首页路由 http://localhost:8080/home。这种功能可以为用户提供更好的导航体验。接下来，让我们看看如何在 Gatsby 静态站点项目中实现这一点。

## @reach/router

@reach/router 是一个轻量级的、专注于用户体验和可访问性的现代 JavaScript 路由库。在 Gatsby 中，@reach/router 已被内置用来处理客户端路由。@reach/router 的 Redirect 组件可以用来在客户端应用中实现路由重定向。

在 Gatsby 中，src/pages 目录下的文件会自动转换成静态 HTML 页面，这样的架构可确保初次加载时的快速响应。而使用 @reach/router 提供的客户端路由机制——通过 history 模式，URL 的变化不会导致页面重载。相反，Router 组件会捕获这些变化，动态地重新计算并渲染匹配的路由组件，从而实现平滑的导航体验。

```tsx
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

尽管我们希望在用户访问 http://localhost:8080/ 时能够自动重定向到首页路由 http://localhost:8080/home，现实情况是页面却跳转到了 404 错误页面。

![Img](./FILES/Gatsby%20路由重定向.md/img-20241202173817.gif)

从断点调试结果可以看出，实际上 Redirect 的重定向操作是成功执行的，并且在过程中确实渲染了 Home 组件。然而，不知为何，最后页面又跳转到了 404 错误页面。

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

以下是 Redirect 组件的实现逻辑。在 componentDidMount 阶段，该组件会调用 navigate 方法来执行路由跳转。

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

navigate 方法用于路由跳转时，会利用 replaceState 或 pushState 这两个 API 进行操作。这些 API 更改 pathname 的过程中不会导致页面重新加载。随后，LocationProvider 组件监听到路径的变化，会根据新的 pathname 重新匹配和渲染对应的路由组件，从而实现页面的无缝过渡。

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

从断点调试来看，之所以页面最终跳转到 404 错误页面，是因为 Gatsby 在构建过程中会为 src/pages 目录中的每个文件生成对应的 HTML 文件和一个相应的页面数据文件 page-data.json。然而，/home 路由并没有实际的 HTML 文件。当 Gatsby 监听到路由变化后，会尝试请求该路由对应的页面数据，即 http://localhost:8888/page-data/home/page-data.json。由于这个请求返回了 404 响应，Gatsby 随即渲染开发环境下的默认 404 页面（这不是用户自定义 404 页面）。

```js
// 1、onClientEntry：Gatsby 提供的钩子，用于在客户端 JavaScript 入口文件执行之前运行代码
apiRunnerAsync(`onClientEntry`).then(() => {
  Promise.all([
    loader.loadPage(`/dev-404-page/`),
    loader.loadPage(`/404.html`),
    loader.loadPage(window.location.pathname + window.location.search),
  ]).then(() => {
    // 加载成功挂载组件
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


// 2、Root
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


// 2、loadPage: 加载页面数据
class BaseLoader {
  loadPage(rawPath) {
    const pagePath = findPath(rawPath)
    if (this.pageDb.has(pagePath)) {
      const page = this.pageDb.get(pagePath)
      if (process.env.BUILD_STAGE !== `develop` || !page.payload.stale) {
        if (page.error) {
          return Promise.resolve({
            error: page.error,
            status: page.status,
          })
        }

        return Promise.resolve(page.payload)
      }
    }

    if (this.inFlightDb.has(pagePath)) {
      return this.inFlightDb.get(pagePath)
    }

    const loadDataPromises = [
      this.loadAppData(),
      this.loadPageDataJson(pagePath),
    ]

    if (global.hasPartialHydration) {
      loadDataPromises.push(this.loadPartialHydrationJson(pagePath))
    }

    const inFlightPromise = Promise.all(loadDataPromises).then(allData => {
      const [appDataResponse, pageDataResponse, rscDataResponse] = allData

      if (
        pageDataResponse.status === PageResourceStatus.Error ||
        rscDataResponse?.status === PageResourceStatus.Error
      ) {
        return {
          status: PageResourceStatus.Error,
        }
      }

      let pageData = pageDataResponse.payload

      const {
        componentChunkName,
        staticQueryHashes: pageStaticQueryHashes = [],
        slicesMap = {},
      } = pageData

      const finalResult = {}

      const dedupedSliceNames = Array.from(new Set(Object.values(slicesMap)))

      const loadSlice = slice => {
        if (this.slicesDb.has(slice.name)) {
          return this.slicesDb.get(slice.name)
        } else if (this.sliceInflightDb.has(slice.name)) {
          return this.sliceInflightDb.get(slice.name)
        }

        const inFlight = this.loadComponent(slice.componentChunkName).then(
          component => {
            return {
              component: preferDefault(component),
              sliceContext: slice.result.sliceContext,
              data: slice.result.data,
            }
          }
        )

        this.sliceInflightDb.set(slice.name, inFlight)
        inFlight.then(results => {
          this.slicesDb.set(slice.name, results)
          this.sliceInflightDb.delete(slice.name)
        })

        return inFlight
      }

      return Promise.all(
        dedupedSliceNames.map(sliceName => this.loadSliceDataJson(sliceName))
      ).then(slicesData => {
        const slices = []
        const dedupedStaticQueryHashes = [...pageStaticQueryHashes]

        for (const { jsonPayload, sliceName } of Object.values(slicesData)) {
          slices.push({ name: sliceName, ...jsonPayload })
          for (const staticQueryHash of jsonPayload.staticQueryHashes) {
            if (!dedupedStaticQueryHashes.includes(staticQueryHash)) {
              dedupedStaticQueryHashes.push(staticQueryHash)
            }
          }
        }

        const loadChunkPromises = [
          Promise.all(slices.map(loadSlice)),
          this.loadComponent(componentChunkName, `head`),
        ]

        if (!global.hasPartialHydration) {
          loadChunkPromises.push(this.loadComponent(componentChunkName))
        }

        // In develop we have separate chunks for template and Head components
        // to enable HMR (fast refresh requires single exports).
        // In production we have shared chunk with both exports. Double loadComponent here
        // will be deduped by webpack runtime resulting in single request and single module
        // being loaded for both `component` and `head`.
        // get list of components to get
        const componentChunkPromises = Promise.all(loadChunkPromises).then(
          components => {
            const [sliceComponents, headComponent, pageComponent] = components

            finalResult.createdAt = new Date()

            for (const sliceComponent of sliceComponents) {
              if (!sliceComponent || sliceComponent instanceof Error) {
                finalResult.status = PageResourceStatus.Error
                finalResult.error = sliceComponent
              }
            }

            if (
              !global.hasPartialHydration &&
              (!pageComponent || pageComponent instanceof Error)
            ) {
              finalResult.status = PageResourceStatus.Error
              finalResult.error = pageComponent
            }

            let pageResources

            if (finalResult.status !== PageResourceStatus.Error) {
              finalResult.status = PageResourceStatus.Success
              if (
                pageDataResponse.notFound === true ||
                rscDataResponse?.notFound === true
              ) {
                finalResult.notFound = true
              }
              pageData = Object.assign(pageData, {
                webpackCompilationHash: appDataResponse
                  ? appDataResponse.webpackCompilationHash
                  : ``,
              })

              if (typeof rscDataResponse?.payload === `string`) {
                pageResources = toPageResources(pageData, null, headComponent)

                pageResources.partialHydration = rscDataResponse.payload

                const readableStream = new ReadableStream({
                  start(controller) {
                    const te = new TextEncoder()
                    controller.enqueue(te.encode(rscDataResponse.payload))
                  },
                  pull(controller) {
                    // close on next read when queue is empty
                    controller.close()
                  },
                  cancel() {},
                })

                return waitForResponse(
                  createFromReadableStream(readableStream)
                ).then(result => {
                  pageResources.partialHydration = result

                  return pageResources
                })
              } else {
                pageResources = toPageResources(
                  pageData,
                  pageComponent,
                  headComponent
                )
              }
            }

            // undefined if final result is an error
            return pageResources
          }
        )

        // get list of static queries to get
        const staticQueryBatchPromise = Promise.all(
          dedupedStaticQueryHashes.map(staticQueryHash => {
            // Check for cache in case this static query result has already been loaded
            if (this.staticQueryDb[staticQueryHash]) {
              const jsonPayload = this.staticQueryDb[staticQueryHash]
              return { staticQueryHash, jsonPayload }
            }

            return this.memoizedGet(
              `${__PATH_PREFIX__}/page-data/sq/d/${staticQueryHash}.json`
            )
              .then(req => {
                const jsonPayload = JSON.parse(req.responseText)
                return { staticQueryHash, jsonPayload }
              })
              .catch(() => {
                throw new Error(
                  `We couldn't load "${__PATH_PREFIX__}/page-data/sq/d/${staticQueryHash}.json"`
                )
              })
          })
        ).then(staticQueryResults => {
          const staticQueryResultsMap = {}

          staticQueryResults.forEach(({ staticQueryHash, jsonPayload }) => {
            staticQueryResultsMap[staticQueryHash] = jsonPayload
            this.staticQueryDb[staticQueryHash] = jsonPayload
          })

          return staticQueryResultsMap
        })

        return (
          Promise.all([componentChunkPromises, staticQueryBatchPromise])
            .then(([pageResources, staticQueryResults]) => {
              let payload
              if (pageResources) {
                payload = { ...pageResources, staticQueryResults }
                finalResult.payload = payload
                emitter.emit(`onPostLoadPageResources`, {
                  page: payload,
                  pageResources: payload,
                })
              }

              this.pageDb.set(pagePath, finalResult)

              if (finalResult.error) {
                return {
                  error: finalResult.error,
                  status: finalResult.status,
                }
              }

              return payload
            })
            // when static-query fail to load we throw a better error
            .catch(err => {
              return {
                error: err,
                status: PageResourceStatus.Error,
              }
            })
        )
      })
    })

    inFlightPromise
      .then(() => {
        this.inFlightDb.delete(pagePath)
      })
      .catch(error => {
        this.inFlightDb.delete(pagePath)
        throw error
      })

    this.inFlightDb.set(pagePath, inFlightPromise)

    return inFlightPromise
  }
}
```


## createRedirect

onClientEntry --> loader.loadPage(window.location.pathname + window.location.search) --> this.loadPageDataJson(pagePath) --> this.fetchPageDataJson({ pagePath })

```
src/
└── pages/
    ├── home/
    │   └── index.tsx
    └── index.tsx
```
