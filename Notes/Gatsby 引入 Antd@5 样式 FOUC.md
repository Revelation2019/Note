# Gatsby 引入 Antd@5 样式 FOUC

当使用 `Gatsby` 搭建的静态站点项目结合 `Antd@5` 版本的组件库，并采用客户端渲染（`CSR`）时，可能会出现样式闪烁（`FOUC，Flash of Unstyled Content`）的现象。这主要是由于 `Antd@5` 采用了 `CSS-in-JS` 的样式方案，在运行时才动态生成和应用样式。因此，可能会出现组件已经被渲染但样式尚未应用的情况，从而导致短暂的样式闪烁问题。

以下是一个**生产环境**样式闪烁问题的示例：

![Img](./FILES/Gatsby%20样式%20FOUC.md/img-20241205171829.gif)

为什么要强调是生产环境呢？是因为本地开发环境不会出现样式闪烁的问题。以下是本地环境的运行情况：

![Img](./FILES/Gatsby%20引入%20Antd@5%20样式%20FOUC.md/img-20241206150542.gif)

我们可以对比不同环境下构建生成的资源产物，以了解它们之间的区别。

## 生产环境构建

以下是生产环境构建时生成的页面 `HTML`。可以看到，在构建过程中，组件元素已经被挂载到 `body` 元素上。页面中使用 `SASS Modules` 范式编写的样式被单独抽取，并通过 `<style>` 标签注入到文档流中。

![Img](./FILES/Gatsby%20引入%20Antd@5%20样式%20FOUC.md/img-20241206111827.png)

然而，`Antd@5` 组件的样式与 `SASS Modules` 不同，并未被预先抽取并通过 `<style>` 标签注入，而是依赖于运行时执行的脚本来进行样式注入。**因此，当页面元素完成解析和渲染时，如果脚本尚未完成样式注入，就可能导致样式闪烁的现象。**

![Img](./FILES/Gatsby%20引入%20Antd@5%20样式%20FOUC.md/img-20241205174319.png)

## 本地环境构建

以下是在本地环境中访问的 `HTML`。从中可以看出，页面元素并不是在构建时生成的，而是通过脚本执行后进行挂载的。因此，在脚本执行以挂载元素的同时进行样式注入，从而避免了样式闪烁的问题。

![Img](./FILES/Gatsby%20引入%20Antd@5%20样式%20FOUC.md/img-20241206145417.png)

## 样式注入过程

以 `Empty` 组件为例，我们可以探讨它在运行时如何注入样式的过程。

```tsx
const Empty = (props) => {
  const { getPrefixCls, direction, empty } = React.useContext(ConfigContext);
  const prefixCls = getPrefixCls('empty', customizePrefixCls);
  // 默认 prefixCls: ant-empty
  const [wrapCSSVar, hashId, cssVarCls] = useStyle(prefixCls);

  return wrapCSSVar(
    <div
      className={classNames(
        hashId,
        cssVarCls,
        prefixCls,
        empty?.className,
        {
          [`${prefixCls}-normal`]: image === simpleEmptyImg,
          [`${prefixCls}-rtl`]: direction === 'rtl',
        },
        className,
        rootClassName,
      )}
      style={{ ...empty?.style, ...style }}
      {...restProps}
    >
      ...
    </div>,
  );
};
```

在 `Empty` 组件内部，会调用 `useStyle` 钩子函数，该函数返回一个名为 `wrapCSSVar` 的变量。`wrapCSSVar` 是一个函数，其主要作用是注入 `CSS` 样式变量。至于其具体的实现过程，我们将在后续进行详细讲解。

```tsx
const genSharedEmptyStyle: GenerateStyle<EmptyToken> = (token): CSSObject => {
  const { componentCls, margin, marginXS, marginXL, fontSize, lineHeight } = token;

  return {
    [componentCls]: {
      marginInline: marginXS,
      fontSize,
      lineHeight,
      textAlign: 'center',

      // 原来 &-image 没有父子结构，现在为了外层承担我们的 hashId，改成父子结构
      [`${componentCls}-image`]: {
        height: token.emptyImgHeight,
        marginBottom: marginXS,
        opacity: token.opacityImage,

        img: {
          height: '100%',
        },

        svg: {
          maxWidth: '100%',
          height: '100%',
          margin: 'auto',
        },
      },

      ...
    },
  };
};

// ============================== Export ==============================
export default genStyleHooks('Empty', (token) => {
  const { componentCls, controlHeightLG, calc } = token;

  const emptyToken: EmptyToken = mergeToken<EmptyToken>(token, {
    emptyImgCls: `${componentCls}-img`,
    emptyImgHeight: calc(controlHeightLG).mul(2.5).equal(),
    emptyImgHeightMD: controlHeightLG,
    emptyImgHeightSM: calc(controlHeightLG).mul(0.875).equal(),
  });

  return [genSharedEmptyStyle(emptyToken)];
});
```

`useStyle` 函数实际上是通过执行 `genStyleHooks("Empty", styleFn)` 得到的返回值。在此过程中，我们主要关注的是第二个参数 `styleFn`。当 `styleFn` 被执行时，它会生成一个样式对象，具体形式如下：

![Img](./FILES/Gatsby%20引入%20Antd@5%20样式%20FOUC.md/img-20241206140916.png)

`genStyleHooks` 返回的函数实际上就是之前调用的 `useStyle`。在这里，有两个关键的钩子函数需要关注：`useStyle`（与前面提到的不同）和 `useCSSVar`。`useStyle` 钩子的主要作用是将通过 `styleFn` 函数生成的样式对象转换为样式字符串，然后通过 `<style>` 标签将其注入到文档流中。另一方面，当 `useCSSVar` 钩子函数执行时，它返回 `wrapCSSVar`，如前面提到的，`wrapCSSVar` 用于将样式变量通过 `<style>` 标签注入到文档流中。在此处，我们将主要展开讲解 `useStyle` 的具体工作机制和流程。

```tsx
function genStyleHooks<C extends TokenMapKey<CompTokenMap>>(
  component: C | [C, string],
  styleFn: GenStyleFn<CompTokenMap, AliasToken, C>,
  getDefaultToken?: GetDefaultToken<CompTokenMap, AliasToken, C>,
  options?: any,
) {
  const componentName = Array.isArray(component) ? component[0] : component;

  ...

  // Hooks
  const useStyle = genComponentStyleHook(component, styleFn, getDefaultToken, mergedOptions);

  const useCSSVar = genCSSVarRegister(componentName, getDefaultToken, mergedOptions);

  return (prefixCls: string, rootCls: string = prefixCls) => {
    const [, hashId] = useStyle(prefixCls, rootCls);
    const [wrapCSSVar, cssVarCls] = useCSSVar(rootCls);

    return [wrapCSSVar, hashId, cssVarCls] as const;
  };
}
```

`useStyle` 钩子函数实际上是通过执行 `genComponentStyleHook` 函数生成的。

```tsx
function genComponentStyleHook<C extends TokenMapKey<CompTokenMap>>(
  componentName: C | [C, string],
  styleFn: GenStyleFn<CompTokenMap, AliasToken, C>,
  getDefaultToken?: GetDefaultToken<CompTokenMap, AliasToken, C>,
  options: any,
) {
  ...

  // 返回的函数就是 useStyle
  return (prefixCls: string, rootCls: string = prefixCls): UseComponentStyleResult => {
    const { theme, realToken, hashId, token, cssVar } = useToken();

    ...

    const wrapSSR = useStyleRegister(
      { ...sharedConfig, path: [concatComponent, prefixCls, iconPrefixCls] },
      () => {
        ...

        // styleInterpolation：样式对象
        const styleInterpolation = styleFn(mergedToken, {
          hashId,
          prefixCls,
          rootPrefixCls,
          iconPrefixCls,
        });

        const commonStyle =
          typeof getCommonStyle === 'function'
            ? getCommonStyle(mergedToken, prefixCls, rootCls, options.resetFont)
            : null;
        return [options.resetStyle === false ? null : commonStyle, styleInterpolation];
      },
    );

    return [wrapSSR, hashId];
  };
}
```

`useStyle` 钩子函数在执行时，会调用 `useStyleRegister` 钩子函数，最终生成一个函数（即上面的 `wrapSSR`）。不过，在调用时，如 `const [, hashId] = useStyle(prefixCls, rootCls)`，这个生成的函数并未被使用。真正负责处理样式的是传递给 `useGlobalCache` 钩子函数的第五个参数，这是一个函数类型。该函数在获取 `styleStr` 之后，会调用 `updateCSS` 函数。而 `updateCSS` 函数则调用 `injectCSS`，以动态生成 `<style>` 标签，并将 `styleStr` 插入到文档中。

```tsx
export default function useStyleRegister(
  info: any,
  styleFn: () => CSSInterpolation,
) {

  ...

  const [cachedStyleStr, cachedTokenKey, cachedStyleId] =
    useGlobalCache<StyleCacheValue>(
      STYLE_PREFIX,
      fullPath,
      // Create cache if needed
      () => {
        ...
      },

      // Remove cache if no need
      ([, , styleId], fromHMR) => {
        if ((fromHMR || autoClear) && isClientSide) {
          removeCSS(styleId, { mark: ATTR_MARK });
        }
      },

      // Effect: Inject style here
      ([styleStr, _, styleId, effectStyle]) => {
        if (isMergedClientSide && styleStr !== CSS_FILE_STYLE) {
          ...

          // ==================== Inject Style ====================
          // styleStr：样式字符串，updateCSS 会调用 injectCSS 动态生成 style 标签注入样式字符串
          const style = updateCSS(styleStr, styleId, mergedCSSConfig);

          (style as any)[CSS_IN_JS_INSTANCE] = cache.instanceId;

          // Used for `useCacheToken` to remove on batch when token removed
          style.setAttribute(ATTR_TOKEN, tokenKey);
        }
      },
    );

  return (node: React.ReactElement) => {
    let styleNode: React.ReactElement;

    if (!ssrInline || isMergedClientSide || !defaultCache) {
      styleNode = <Empty />;
    } else {
      styleNode = (
        <style
          {...{
            [ATTR_TOKEN]: cachedTokenKey,
            [ATTR_MARK]: cachedStyleId,
          }}
          dangerouslySetInnerHTML={{ __html: cachedStyleStr }}
        />
      );
    }

    return (
      <>
        {styleNode}
        {node}
      </>
    );
  };
}
```

`updateCSS` 的实现主要涉及以下部分：

```tsx
export function updateCSS(css, key) {
  ...
  var newNode = injectCSS(css, option);
  newNode.setAttribute(getMark(option), key);
  return newNode;
}
```

`injectCSS` 的实现主要涉及以下部分：

```tsx
export function injectCSS(css) {
  ...
  var styleNode = document.createElement('style');
  styleNode.setAttribute(APPEND_ORDER, mergedOrder);
  if (isPrependQueue && priority) {
    styleNode.setAttribute(APPEND_PRIORITY, "".concat(priority));
  }
  if (csp !== null && csp !== void 0 && csp.nonce) {
    styleNode.nonce = csp === null || csp === void 0 ? void 0 : csp.nonce;
  }
  styleNode.innerHTML = css;
  var container = getContainer(option);
  var firstChild = container.firstChild;
  if (prepend) {
    ...

    // Use `insertBefore` as `prepend`
    container.insertBefore(styleNode, firstChild);
  } else {
    container.appendChild(styleNode);
  }
  return styleNode;
}
```

## 构建过程的差异

让我们先来看看 production 和 development 构建流程之间有什么不同。

```js
// 默认的 webpack 构建配置
const config = {
  ...
  entry: getEntry(),
  ...
}

function getEntry() {
  switch (stage) {
    case `develop`:
      ...
    case `develop-html`:
      return {
        "render-page": process.env.GATSBY_EXPERIMENTAL_DEV_SSR
          ? directoryPath(`.cache/ssr-develop-static-entry`)
          : directoryPath(`.cache/develop-static-entry`),
      }
    case `build-html`: {
      return {
        "render-page": directoryPath(`.cache/static-entry`),
      }
    }
    case `build-javascript`:
      ...
    default:
      throw new Error(`The state requested ${stage} doesn't exist.`)
  }
}
```

在这个代码片段中，我们特别关注构建过程中的两个阶段：`develop-html` 和 `build-html`。`develop-html` 阶段用于开发环境的 `HTML` 构建，而 `build-html` 阶段则用于生产环境的 HTML 构建。在不考虑 `SSR`（服务器端渲染）构建流程的情况下，使用 `CSR`（客户端渲染）时，开发环境生成 `HTML` 的入口文件是 `.cache/develop-static-entry`，而生产环境的入口文件则是 `.cache/static-entry`。这两个文件来源于 `Gatsby` 库，并在项目构建过程中复制到 `.cache` 目录中。

首先，让我们来看一下 `.cache/static-entry` 的实现：

```js
export default async function staticPage({
  pagePath,
  pageData,
  staticQueryContext,
  styles,
  scripts,
  reversedStyles,
  reversedScripts,
  inlinePageData = false,
  context = {},
  webpackCompilationHash,
  sliceData,
}) {
  try {
    let bodyHtml = ``
    const { componentChunkName, slicesMap } = pageData
    // 加载对应组件的 componentChunk
    const pageComponent = await asyncRequires.components[componentChunkName]()

    class RouteHandler extends React.Component {
      render() {
        const props = {
          ...this.props,
          ...pageData.result,
          params: {
            ...grabMatchParams(this.props.location.pathname),
            ...(pageData.result?.pageContext?.__params || {}),
          },
        }
        // 执行页面组件函数得到 React.ReactElement
        const pageElement = createElement(pageComponent.default, props)

        // wrapPageElement 包装页面组件
        const wrappedPage = apiRunner(
          `wrapPageElement`,
          { element: pageElement, props },
          pageElement,
          ({ result }) => {
            return { element: result, props }
          }
        ).pop()

        return wrappedPage
      }
    }

    // 在这里，每当检测到路由变化时，系统会通过 RouteHandler 加载相应的 componentChunk，并执行 wrapPageElement 来包装该页面组件。
    const routerElement = (
      <ServerLocation url={`${__BASE_PATH__}${pagePath}`}>
        <Router id="gatsby-focus-wrapper" baseuri={__BASE_PATH__}>
          <RouteHandler path="/*" />
        </Router>
        <div {...RouteAnnouncerProps} />
      </ServerLocation>
    )

    // wrapRootElement 用于包装整个应用程序，并且仅在应用的初次加载时执行
    let body = apiRunner(
      `wrapRootElement`,
      { element: routerElement, pathname: pagePath },
      routerElement,
      ({ result }) => {
        return { element: result, pathname: pagePath }
      }
    ).pop()

    const bodyComponent = (
      <StaticQueryContext.Provider value={staticQueryContext}>
        {body}
      </StaticQueryContext.Provider>
    )

    // If no one stepped up, we'll handle it.
    if (!bodyHtml) {
      try {
        const writableStream = new WritableAsPromise()
        // 调用 renderToPipeableStream 将 React 组件树渲染为 HTML 后注入 Node.js 流
        const { pipe } = renderToPipeableStream(bodyComponent, {
          onAllReady() {
            pipe(writableStream)
          },
          onError(error) {
            writableStream.destroy(error)
          },
        })

        bodyHtml = await writableStream
      } catch (e) {
        // ignore @reach/router redirect errors
        if (!isRedirect(e)) throw e
      }
    }

    let htmlElement = (
      <Html
        {...bodyProps}
        headComponents={headComponents}
        htmlAttributes={htmlAttributes}
        bodyAttributes={bodyAttributes}
        preBodyComponents={preBodyComponents}
        postBodyComponents={postBodyComponents}
        body={bodyHtml}
        path={pagePath}
      />
    )
    // renderToStaticMarkup 会将非交互的 React 组件树渲染成 HTML 字符串。
    const html = `<!DOCTYPE html>${renderToStaticMarkup(htmlElement)}`

    return {
      html,
      unsafeBuiltinsUsage: global.unsafeBuiltinUsage,
      sliceData: sliceProps,
    }
  } catch (e) {
    e.unsafeBuiltinsUsage = global.unsafeBuiltinUsage
    throw e
  }
}
```

