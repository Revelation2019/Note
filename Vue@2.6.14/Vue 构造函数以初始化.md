# Vue 初始化

Vue.js 的构造函数及其初始化过程的实现

```js
function Vue (options) {
  // 检查是否有使用 new 关键字实例化对象，提升代码的健壮性
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword');
  }
  this._init(options);
}

// 将与初始化相关的方法混入到 Vue 构造函数中。这些方法通常用于设置数据、计算属性等
initMixin(Vue);
// 将与状态管理相关的方法混入到 Vue 构造函数中。这些方法包括对 data、computed 和 watch 的处理
stateMixin(Vue);
// 将与事件管理相关的方法混入到 Vue 构造函数中。这些方法用于处理事件的监听和触发
eventsMixin(Vue);
// 将与生命周期管理相关的方法混入到 Vue 构造函数中。这些方法用于处理组件的生命周期钩子，如 created、mounted 等
lifecycleMixin(Vue);
// 将与渲染相关的方法混入到 Vue 构造函数中。这些方法用于处理组件的渲染逻辑
renderMixin(Vue);
```

## initMixin（_init）

这段代码是 Vue.js 中的 initMixin 函数的实现，在 Vue 的原型上定义了一个 \_init 方法，用于初始化 Vue 实例。

```js
function initMixin (Vue) {
  Vue.prototype._init = function (options) {
    var vm = this;
    // a uid
    vm._uid = uid$3++; // 为每个 Vue 实例分配一个唯一的标识符 _uid，用于跟踪和标识实例

    var startTag, endTag;
    /* istanbul ignore if */
    // 这段代码用于在开发环境中进行性能标记，以便在性能分析工具中跟踪 Vue 实例的初始化过程
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      startTag = "vue-perf-start:" + (vm._uid);
      endTag = "vue-perf-end:" + (vm._uid);
      mark(startTag);
    }

    // a flag to avoid this being observed
    // 将实例的 _isVue 属性设置为 true，用于标识该对象是一个 Vue 实例
    vm._isVue = true;
    // merge options
    if (options && options._isComponent) {
      // 检查传入的选项是否表示一个组件。如果是，则调用initInternalComponent 方法进行内部组件的初始化
      // optimize internal component instantiation
      // since dynamic options merging is pretty slow, and none of the
      // internal component options needs special treatment.
      initInternalComponent(vm, options);
    } else {
      // 如果不是组件，则合并构造函数的选项和传入的选项，生成实例的 $options
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      );
    }
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      // 在开发环境中，调用 initProxy 方法为实例设置代理，以便在模板中访问数据属性
      initProxy(vm);
    } else {
      // 在生产环境中，直接将 _renderProxy 设置为实例本身
      vm._renderProxy = vm;
    }
    // expose real self
    // 将实例的 _self 属性设置为自身，方便在内部方法中引用
    vm._self = vm;
    // 初始化生命周期、事件和渲染
    initLifecycle(vm);
    initEvents(vm);
    initRender(vm);
    // 生命周期钩子
    // 调用生命周期钩子 beforeCreate 和 created，在这些钩子中可以执行用户定义的逻辑
    // initInjections、initState 和 initProvide 方法用于处理依赖注入、状态管理和提供的属性
    callHook(vm, 'beforeCreate');
    initInjections(vm); // resolve injections before data/props
    initState(vm);
    initProvide(vm); // resolve provide after data/props
    callHook(vm, 'created');

    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      vm._name = formatComponentName(vm, false);
      mark(endTag);
      measure(("vue " + (vm._name) + " init"), startTag, endTag);
    }

    // 如果在选项中指定了 el，则调用 $mount 方法将实例挂载到指定的 DOM 元素上
    if (vm.$options.el) {
      vm.$mount(vm.$options.el);
    }
  };
}
```

resolveConstructorOptions 函数用于处理组件构造器的选项，尤其是在组件继承结构中。

```js
function resolveConstructorOptions (Ctor) {
  var options = Ctor.options; // 获取构造函数 Ctor 的选项
  if (Ctor.super) {
    var superOptions = resolveConstructorOptions(Ctor.super);
    var cachedSuperOptions = Ctor.superOptions;
    // 调用 resolveConstructorOptions 递归地获取父类的选项，并将其与缓存的父类选项进行比较
    if (superOptions !== cachedSuperOptions) {
      // 如果父类的选项发生了变化，说明需要更新当前构造函数的选项
      // super option changed,
      // need to resolve new options.
      // 更新 Ctor.superOptions
      Ctor.superOptions = superOptions;
      // check if there are any late-modified/attached options (#4976)
      // 检查选项在初始设置后是否被修改，modifiedOptions 表示被修改的选项
      var modifiedOptions = resolveModifiedOptions(Ctor);
      // update base extend options
      if (modifiedOptions) {
        // 将修改的选项与拓展选项合并
        extend(Ctor.extendOptions, modifiedOptions);
      }
      // 子类会继承父类的选项，所以将当前构造函数的扩展选项和父类选项合并，并更新 options
      options = Ctor.options = mergeOptions(superOptions, Ctor.extendOptions);
      if (options.name) {
        // 如果组件有名称，将其注册到 options.components 中
        options.components[options.name] = Ctor;
      }
    }
  }
  return options
}
```

![Img](./FILES/Vue%20构造函数以初始化.md/img-20241031213931.png)

最终得到的 options 是一个包含子类和父类的全面配置对象。这个对象包括：

1. 继承自父类的选项 : 包括所有父类的配置，如 data, methods, computed 等。
2. 子类特有的扩展 : Ctor.extendOptions 中的选项，以及任何在运行时动态更新或修改的选项。
3. 整合整个继承链的更改 : 任何父类的更新，或全局混入等影响都通过这个方法得到反映。

resolveModifiedOptions 函数的作用是在组件构造函数内，识别和返回那些在最新的组件选项 (Ctor.options) 和其封存的（初始）选项 (Ctor.sealedOptions) 之间发生了变动的选项。

```js
function resolveModifiedOptions (Ctor) {
  var modified; // 用于存储检测到的修改后的选项，会在检测到实际变化时初始化
  var latest = Ctor.options; // 指向当前使用的选项 (Ctor.options)
  var sealed = Ctor.sealedOptions; // 指向最初封存的选项 (Ctor.sealedOptions)。在组件构造时，sealedOptions 通常被设置为 options 的快照
  for (var key in latest) {
    if (latest[key] !== sealed[key]) {
      if (!modified) { modified = {}; }
      modified[key] = latest[key];
    }
  }
  return modified
}
```

sealedOptions 和 options 是 Vue 内部用于管理组件选项状态的两个重要属性。在大多数情况下，这两个对象是相同的，都是组件的配置选项的快照。然而，存在某些场景下，它们可能会存在不同，具体来说：

1. **组件选项的动态修改**：如果组件的选项在组件创建后被动态修改，那么 sealedOptions 与 options 就可能会不一样。sealedOptions 通常是在组件构造时创建的一份选项快照，它不会随着动态修改而改变，而 options 会反映当前的配置状态。
2. **插件或混入的影响**：如果有插件或混入在组件创建后对组件选项进行了修改，这些修改将仅应用于当前使用的 options，而不会影响到 sealedOptions。例如，某些插件可能在全局混入中添加选项或动态调整组件行为。
3. **框架内部逻辑**：Vue 内部可能会在某些特殊情况下调整或扩展组件的选项来实现特定的功能。此种情况下，options 表现的随时可变性是至关重要的。

在 Vue 2 中，子类构造器的 super 属性指向父类构造器的情况主要出现在通过 Vue.extend 方法创建组件时。这种结构允许子类继承父类的属性、方法等选项，实现组件的继承和扩展。以下是 super 指向父类构造器的常见情况：

当你使用 Vue.extend 方法来创建一个新的子组件时，子组件构造器会有一个指向父组件构造器的 super 属性。下面是一个例子：

```js
// 创建一个父类组件构造器
var ParentComponent = Vue.extend({
  data: function() {
    return {
      message: "Hello from Parent"
    };
  }
});

// 创建一个子类组件构造器，它继承自 ParentComponent
var ChildComponent = ParentComponent.extend({
  data: function() {
    return {
      childMessage: "Hello from Child"
    };
  }
});

// ChildComponent.super --> ParentComponent
// ParentComponent.super --> Vue
// Vue.super --> undefined
console.log(ChildComponent.super === ParentComponent); // 输出：true
```

这段代码是 Vue.js 中的 initProxy 函数的实现，主要用于为 Vue 实例设置代理，以便在模板中访问数据属性

```js
var getHandler = {
  get: function get (target, key) {
    if (typeof key === 'string' && !(key in target)) {
      if (key in target.$data) { warnReservedPrefix(target, key); }
      else { warnNonPresent(target, key); }
    }
    return target[key]
  }
};

var initProxy = function initProxy (vm) {
  // 当前环境是否支持 JavaScript 的 Proxy 特性
  if (hasProxy) {
    // determine which proxy handler to use
    var options = vm.$options;
    var handlers = options.render && options.render._withStripped
      ? getHandler
      : hasHandler;
    vm._renderProxy = new Proxy(vm, handlers);
  } else {
    // 如果当前环境不支持 Proxy，则直接将 _renderProxy 设置为实例本身 vm。这意味着在没有代理的情况下，Vue 实例的属性访问将直接使用实例本身
    vm._renderProxy = vm;
  }
};
```

### initLifecycle

这个函数 initLifecycle 是 Vue.js 框架的一部分，用于初始化 Vue 实例的生命周期相关属性。以下是它的工作原理：

```js
function initLifecycle (vm) {
  var options = vm.$options;

  // locate first non-abstract parent
  // 向上查找第一个非抽象的父组件，将当前实例 vm 添加到父组件的 $children 数组中
  var parent = options.parent;
  if (parent && !options.abstract) {
    while (parent.$options.abstract && parent.$parent) {
      parent = parent.$parent;
    }
    parent.$children.push(vm);
  }

  // 设置当前实例的父组件
  vm.$parent = parent;
  // 设置根组件
  vm.$root = parent ? parent.$root : vm;

  vm.$children = [];
  vm.$refs = {};

  // 这是 Vue 实例的主观察者对象，用于响应式系统。当数据变化时，观察者会触发相应的更新
  vm._watcher = null;
  // 这个属性用于标记组件是否处于非活动状态。它在 keep-alive 组件中使用，以避免不必要的更新
  vm._inactive = null;
  // 这个属性用于标记组件是否直接被父组件停用。它与 _inactive 一起使用，以确定组件的活动状态
  vm._directInactive = false;
  // 这个属性用于标记组件是否已经挂载到 DOM 上。当组件挂载完成后，这个属性会被设置为 true
  vm._isMounted = false;
  // 这个属性用于标记组件是否已经被销毁。当组件销毁完成后，这个属性会被设置为 true
  vm._isDestroyed = false;
  // 这个属性用于标记组件是否正在被销毁的过程中。当组件开始销毁时，这个属性会被设置为 true，销毁完成后会被设置为 false
  vm._isBeingDestroyed = false;
}
```

#### 抽象组件

Vue.js 提供了一个 abstract 选项，允许在组件配置项中进行设置以定义抽象组件。当 abstract: true 设置时，该组件自身不会渲染为一个 DOM 元素，也不会作为节点出现在父组件链中。

在抽象组件的生命周期中，开发者可以拦截和处理包裹子组件的事件，也可以对其进行 DOM 操作。这种能力使得我们能够封装所需的功能和逻辑，而无需关注子组件的具体实现细节。

**抽象组件的特点**

1. 不产生 DOM：抽象组件不会在 DOM 树中生成任何对应的节点。它们的存在不会影响实际的 DOM 结构，而是用于处理一些逻辑或提供某些功能。
2. 逻辑封装 ：可以用于封装和复用逻辑，与普通组件类似，但不承担任何 UI 渲染任务。
3. 不渲染自身 ：通常不包含模板（template），即使包含 <template></template> 也不会渲染。它们的 render 函数一般是通过 this.$slots.default 来直接渲染子级内容。

利用 Vue.js 的抽象组件特性结合 Lodash 的 debounce 函数，实现一个防抖功能的 Vue 组件

```js
// debounce.js
import { debounce } from 'lodash';

export default {
  name: 'Debounce',
  abstract: true, // 声明该组件为抽象组件
  props: {
    wait: {
      type: Number,
      default: 350,
    },
  },
  render() {
    const slot = this.$slots.default;
    // 重写 click 事件
    slot[0].data.on.click = debounce(slot[0].data.on.click, this.wait);

    return slot[0];
  }
}
```

可以在其它组件中使用这个 Debounce 组件来包裹需要应用防抖逻辑的按钮或其他元素

```vue
<template>
  <debounce :wait="500">
    <button @click="handleClick">Click Me</button>
  </debounce>
</template>

<script>
import Debounce from './debounce.js';

export default {
  components: {
    Debounce
  },
  methods: {
    handleClick() {
      console.log('Button clicked with debounce!');
    }
  }
}
</script>
```

在 Vue 2 中，虽然没有明确的常用内置组件被标记为“抽象组件”，但有一些组件确实在逻辑上充当了抽象角色，主要用于提供特定功能而不会直接渲染为 DOM 元素。这些组件主要用于实现页面或组件的逻辑或功能模块化，如：

1. `<transition>`
   用于在元素或组件上应用进入和离开的过渡效果。
   不渲染实际的 DOM 元素，但是控制其中包含的元素的动画过渡。
2. `<transition-group>`
   针对一组元素或组件的过渡效果。
   尽管它会渲染一个实际的包装 DOM 元素（例如 span 或 div），其主要用于管理组内子元素的过渡，而不是自身的展示。
3. `<keep-alive>`
   用于在切换动态组件时缓存不活跃组件实例，从而避免销毁和重建组件。
   不直接渲染任何 DOM，作用是为组件缓存提供逻辑支持。

### initEvents

这个函数 initEvents 是 Vue.js 框架的一部分，用于初始化 Vue 实例的事件系统

```js
function initEvents (vm) {
  vm._events = Object.create(null); // 初始化一个空对象 _events，用于存储事件监听器
  vm._hasHookEvent = false; // 标记是否有钩子事件监听器
  // init parent attached events
  var listeners = vm.$options._parentListeners; // 获取父组件传递的事件监听器
  if (listeners) {
    // 如果存在父组件传递的事件监听器，则调用 updateComponentListeners 函数，将这些监听器添加到当前组件实例中
    updateComponentListeners(vm, listeners);
  }
}
```

\_parentListeners 是 Vue 实例选项中的一个属性，用于存储父组件传递给子组件的事件监听器。这些监听器在子组件初始化时会被添加到子组件的事件系统中。

在 Vue.js 中，父组件可以通过在模板中使用 v-on 指令将事件监听器传递给子组件。例如：

```js
<child-component v-on:custom-event="handleCustomEvent"></child-component>
```

在这种情况下，handleCustomEvent 函数会作为事件监听器传递给 child-component，并存储在 child-component 实例的 $options.\_parentListeners 属性中。

在子组件初始化时，initEvents 函数会检查 $options.\_parentListeners 是否存在，并调用 updateComponentListeners 函数将这些监听器添加到子组件的事件系统中。

```js
var target;

// 将事件监听器添加到 Vue 实例的 _events 属性中，比如 vm._events = { "custom-event": [ handleCustomEvent ] }
function add (event, fn) {
  target.$on(event, fn);
}

// 从 Vue 实例的 _events 属性中移除对应事件
function remove$1 (event, fn) {
  target.$off(event, fn);
}

// 用于创建一个一次性事件处理器。一次性事件处理器在第一次被触发后，会自动移除自身，以确保该事件处理器只执行一次
function createOnceHandler (event, fn) {
  var _target = target;
  return function onceHandler () {
    var res = fn.apply(null, arguments);
    if (res !== null) {
      _target.$off(event, onceHandler);
    }
  }
}

function updateComponentListeners (
  vm,
  listeners,
  oldListeners
) {
  target = vm;
  updateListeners(listeners, oldListeners || {}, add, remove$1, createOnceHandler, vm);
  target = undefined;
}

// 根据新的和旧的事件监听器列表，添加新的监听器并移除不再需要的监听器
function updateListeners (
  on, // 新的事件监听器对象
  oldOn, // 旧的事件监听器对象
  add,
  remove$$1,
  createOnceHandler,
  vm
) {
  var name, def$$1, cur, old, event;
  for (name in on) {
    def$$1 = cur = on[name]; // 获取当前事件监听器
    old = oldOn[name]; // 获取旧的事件监听器
    event = normalizeEvent(name);
    if (isUndef(cur)) {
      process.env.NODE_ENV !== 'production' && warn(
        "Invalid handler for event \"" + (event.name) + "\": got " + String(cur),
        vm
      );
    } else if (isUndef(old)) {
      if (isUndef(cur.fns)) {
        // 如果新的事件监听器 cur 没有 fns 属性，创建一个函数调用器
        cur = on[name] = createFnInvoker(cur, vm);
      }
      if (isTrue(event.once)) {
        // 如果事件是一次性的，创建一次性处理器
        cur = on[name] = createOnceHandler(event.name, cur, event.capture);
      }
      // 调用 add 函数添加事件监听器到实例 _events 属性上，属于订阅发布者模式
      add(event.name, cur, event.capture, event.passive, event.params);
    } else if (cur !== old) {
      // 如果新的事件监听器 cur 与旧的事件监听器 old 不同，更新旧的事件监听器的 fns 属性
      old.fns = cur;
      on[name] = old;
    }
  }
  for (name in oldOn) {
    if (isUndef(on[name])) {
      event = normalizeEvent(name);
      // 如果新的事件监听器对象 on 中没有对应的事件，调用 remove$$1 函数移除旧的事件监听器。
      remove$$1(event.name, oldOn[name], event.capture);
    }
  }
}
```

### initRender

这个函数 initRender 用于初始化 Vue 实例的渲染相关属性和方法。

```js
function initRender (vm) {
  // 初始化 _vnode 属性为 null，它表示子树的根节点。
  vm._vnode = null; // the root of the child tree
  // 初始化 _staticTrees 属性为 null，用于缓存 v-once 指令的静态树。
  vm._staticTrees = null; // v-once cached trees
  var options = vm.$options;
  // 获取父组件的虚拟节点
  var parentVnode = vm.$vnode = options._parentVnode; // the placeholder node in parent tree
  // 获取父组件的渲染上下文，其实就是父组件的实例对象
  var renderContext = parentVnode && parentVnode.context;
  // 解析插槽内容并赋值给 vm.$slots
  vm.$slots = resolveSlots(options._renderChildren, renderContext);
  // 初始化 vm.$scopedSlots 为一个空对象
  vm.$scopedSlots = emptyObject;
  // bind the createElement fn to this instance
  // so that we get proper render context inside it.
  // args order: tag, data, children, normalizationType, alwaysNormalize
  // internal version is used by render functions compiled from templates
  // 内部版本的 createElement 函数，用于模板编译生成的渲染函数
  vm._c = function (a, b, c, d) { return createElement(vm, a, b, c, d, false); };
  // normalization is always applied for the public version, used in
  // user-written render functions.
  // 公共版本的 createElement 函数，用于用户编写的渲染函数
  vm.$createElement = function (a, b, c, d) { return createElement(vm, a, b, c, d, true); };

  // $attrs & $listeners are exposed for easier HOC creation.
  // they need to be reactive so that HOCs using them are always updated
  // 获取父组件的虚拟节点数据
  var parentData = parentVnode && parentVnode.data;

  /* istanbul ignore else */
  if (process.env.NODE_ENV !== 'production') {
    // 当向组件传入的属性没有在组件的 props 中定义时，这些属性会被视为普通的 HTML 特性，并存储在 parentData.attrs 中
    defineReactive$$1(vm, '$attrs', parentData && parentData.attrs || emptyObject, function () {
      !isUpdatingChildComponent && warn("$attrs is readonly.", vm);
    }, true);
    // options._parentListeners 存储父组件传递给子组件的事件监听器
    defineReactive$$1(vm, '$listeners', options._parentListeners || emptyObject, function () {
      !isUpdatingChildComponent && warn("$listeners is readonly.", vm);
    }, true);
  } else {
    defineReactive$$1(vm, '$attrs', parentData && parentData.attrs || emptyObject, null, true);
    defineReactive$$1(vm, '$listeners', options._parentListeners || emptyObject, null, true);
  }
}
```

这个函数 resolveSlots 是 Vue.js 的运行时辅助函数，用于将原始的子虚拟节点（VNode）解析为插槽对象。它的主要作用是将传递给组件的子节点整理成插槽，以便在组件内部使用。

```js
function resolveSlots (
  children, // 传递给组件的子虚拟节点数组
  context // 当前组件的上下文（通常是父组件的实例）
) {
  if (!children || !children.length) {
    return {}
  }
  var slots = {}; // 初始化插槽对象
  for (var i = 0, l = children.length; i < l; i++) {
    var child = children[i];
    var data = child.data;
    // remove slot attribute if the node is resolved as a Vue slot node
    if (data && data.attrs && data.attrs.slot) {
      delete data.attrs.slot;
    }
    // named slots should only be respected if the vnode was rendered in the
    // same context.
    if ((child.context === context || child.fnContext === context) &&
      data && data.slot != null
    ) {
      // 具名插槽
      var name = data.slot;
      var slot = (slots[name] || (slots[name] = []));
      if (child.tag === 'template') {
        slot.push.apply(slot, child.children || []);
      } else {
        slot.push(child);
      }
    } else {
      // 默认插槽
      (slots.default || (slots.default = [])).push(child);
    }
  }
  // ignore slots that contains only whitespace
  for (var name$1 in slots) {
    // 如果插槽中的所有节点都是空白节点，则删除该插槽
    if (slots[name$1].every(isWhitespace)) {
      delete slots[name$1];
    }
  }
  return slots
}
```

这个函数 defineReactive$$1 用于在对象上定义一个响应式属性。它通过 Object.defineProperty 方法拦截属性的读取和写入操作，从而实现响应式数据绑定。

```js
/**
 * Define a reactive property on an Object.
 */
function defineReactive$$1 (
  obj,
  key,
  val,
  customSetter,
  shallow
) {
  var dep = new Dep(); // 创建依赖管理器

  var property = Object.getOwnPropertyDescriptor(obj, key); // 获取属性描述符
  // 获取对象上现有属性的描述符，如果属性不可配置，则直接返回
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  var getter = property && property.get;
  var setter = property && property.set;
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key];
  }
  // 如果不是浅层响应式，则递归地将属性值转换为响应式对象
  var childOb = !shallow && observe(val);
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      var value = getter ? getter.call(obj) : val;
      if (Dep.target) {
        dep.depend(); // 依赖收集
        if (childOb) {
          childOb.dep.depend();
          if (Array.isArray(value)) {
            dependArray(value);
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      var value = getter ? getter.call(obj) : val;
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter();
      }
      // #7981: for accessor properties without setter
      if (getter && !setter) { return }
      if (setter) {
        setter.call(obj, newVal);
      } else {
        val = newVal;
      }
      childOb = !shallow && observe(newVal);
      dep.notify(); // 触发更新
    }
  });
}
```

### initInjections

这个函数 initInjections 用于初始化 Vue 实例的依赖注入（injections）。依赖注入是 Vue.js 提供的一种机制，用于在组件树中向下传递数据。

```js
function initInjections (vm) {
  //
  var result = resolveInject(vm.$options.inject, vm);
  if (result) {
    toggleObserving(false);
    // 将依赖注入的数据转化成响应式
    Object.keys(result).forEach(function (key) {
      /* istanbul ignore else */
      if (process.env.NODE_ENV !== 'production') {
        defineReactive$$1(vm, key, result[key], function () {
          warn(
            "Avoid mutating an injected value directly since the changes will be " +
            "overwritten whenever the provided component re-renders. " +
            "injection being mutated: \"" + key + "\"",
            vm
          );
        });
      } else {
        defineReactive$$1(vm, key, result[key]);
      }
    });
    toggleObserving(true);
  }
}
```

这个函数 resolveInject 用于解析 Vue 组件的 inject 选项，获取注入的依赖。它会向上查找组件树中的父组件，直到找到提供依赖项的组件实例，并获取依赖项的值，并将其返回为一个对象。

```js
function resolveInject (inject, vm) {
  if (inject) {
    // inject is :any because flow is not smart enough to figure out cached
    var result = Object.create(null);
    var keys = hasSymbol
      ? Reflect.ownKeys(inject)
      : Object.keys(inject);

    for (var i = 0; i < keys.length; i++) {
      var key = keys[i];
      // #6574 in case the inject object is observed...
      if (key === '__ob__') { continue }
      var provideKey = inject[key].from;
      var source = vm;
      while (source) {
        if (source._provided && hasOwn(source._provided, provideKey)) {
          result[key] = source._provided[provideKey];
          break
        }
        source = source.$parent;
      }
      if (!source) {
        // 如果没有找到提供依赖项的组件实例，检查是否有默认值。如果有默认值，将其添加到结果对象 result 中
        if ('default' in inject[key]) {
          var provideDefault = inject[key].default;
          result[key] = typeof provideDefault === 'function'
            ? provideDefault.call(vm)
            : provideDefault;
        } else if (process.env.NODE_ENV !== 'production') {
          // 如果在开发环境中没有找到提供依赖项且没有默认值，发出警告
          warn(("Injection \"" + key + "\" not found"), vm);
        }
      }
    }
    return result
  }
}
```

在 Vue.js 中，当组件实例化时，会执行 \_init 方法。在这个过程中，Vue 会合并选项（mergeOptions），并对 inject 选项进行标准化处理。标准化处理是通过 normalizeInject 函数完成的。

```js
/**
 * Normalize all injections into Object-based format
 */
function normalizeInject (options, vm) {
  var inject = options.inject;
  if (!inject) { return }
  var normalized = options.inject = {};
  if (Array.isArray(inject)) {
    // 如果 inject 是一个数组，将每个字符串转换为对象形式 { from: inject[i] }，比如 inject: ['foo', 'bar'] 转化成 { foo: { from: 'foo' }, bar: { from: 'bar' } }
    for (var i = 0; i < inject.length; i++) {
      normalized[inject[i]] = { from: inject[i] };
    }
  } else if (isPlainObject(inject)) {
    // 如果 inject 是一个对象，遍历对象的每个键值对
    // 如果值是一个对象，扩展它并添加 from 属性
    // 否则，将值转换为对象形式 { from: val }
    // 比如：
    // inject: {
    //   foo: 'bar',
    //   baz: { from: 'qux', default: 'defaultBaz' }
    // }
    // 转化成：
    // {
    //   foo: { from: 'bar' },
    //   baz: { from: 'qux', default: 'defaultBaz' }
    // }
    for (var key in inject) {
      var val = inject[key];
      normalized[key] = isPlainObject(val)
        ? extend({ from: key }, val)
        : { from: val };
    }
  } else if (process.env.NODE_ENV !== 'production') {
    warn(
      "Invalid value for option \"inject\": expected an Array or an Object, " +
      "but got " + (toRawType(inject)) + ".",
      vm
    );
  }
}
```

需要注意的是，在 Vue.js 的依赖注入机制中，inject[key].from 字段表示的是提供者对象（provide）中的某个键值，而不是来自于哪个具体的组件。resolveInject 函数会根据这个键值向上查找组件树中的提供者对象，以获取相应的依赖项。

假设我们有一个父组件和一个子组件，父组件提供一些数据，子组件通过依赖注入获取这些数据：

```js
// 父组件
Vue.component('parent-component', {
  provide: {
    message: 'Hello from Parent'
  },
  template: `
    <div>
      <child-component></child-component>
    </div>
  `
});

// 子组件
Vue.component('child-component', {
  inject: {
    message: {
      from: 'message',
      default: 'Default Message'
    }
  },
  template: `
    <div>
      {{ message }}
    </div>
  `
});

new Vue({
  el: '#app'
});
```

### initState

这个函数 initState 用于初始化 Vue 实例的状态，包括 props、methods、data、computed 和 watch 等选项。

```js
function initState (vm) {
  vm._watchers = []; // 初始化观察者数组
  var opts = vm.$options;
  if (opts.props) { initProps(vm, opts.props); }
  if (opts.methods) { initMethods(vm, opts.methods); }
  if (opts.data) {
    initData(vm);
  } else {
    observe(vm._data = {}, true /* asRootData */);
  }
  if (opts.computed) { initComputed(vm, opts.computed); }
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch);
  }
}
```

#### initProps

函数用于初始化 Vue 实例的 props。它会根据组件的 props 选项和传入的 propsData，将 props 转换为响应式属性，并进行必要的验证和处理。

```js
function initProps (vm, propsOptions) {
  var propsData = vm.$options.propsData || {}; // 获取传入的 propsData
  var props = vm._props = {};
  // cache prop keys so that future props updates can iterate using Array
  // instead of dynamic object key enumeration.
  var keys = vm.$options._propKeys = [];
  var isRoot = !vm.$parent;
  // root instance props should be converted
  if (!isRoot) {
    toggleObserving(false);
  }
  var loop = function ( key ) {
    keys.push(key);
    var value = validateProp(key, propsOptions, propsData, vm);
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      var hyphenatedKey = hyphenate(key);
      if (isReservedAttribute(hyphenatedKey) ||
          config.isReservedAttr(hyphenatedKey)) {
        warn(
          ("\"" + hyphenatedKey + "\" is a reserved attribute and cannot be used as component prop."),
          vm
        );
      }
      defineReactive$$1(props, key, value, function () {
        if (!isRoot && !isUpdatingChildComponent) {
          warn(
            "Avoid mutating a prop directly since the value will be " +
            "overwritten whenever the parent component re-renders. " +
            "Instead, use a data or computed property based on the prop's " +
            "value. Prop being mutated: \"" + key + "\"",
            vm
          );
        }
      });
    } else {
      // 转化成响应式
      defineReactive$$1(props, key, value);
    }
    // static props are already proxied on the component's prototype
    // during Vue.extend(). We only need to proxy props defined at
    // instantiation here.
    if (!(key in vm)) {
      proxy(vm, "_props", key);
    }
  };

  for (var key in propsOptions) loop( key );
  toggleObserving(true);
}
```

validateProp 函数用于验证和处理 Vue 组件的 props。它会根据 props 的定义和传入的数据进行类型转换、默认值处理和验证。主要对以下两种情况做特殊处理：

- prop.type 定义中包含 Boolean 类型：

  - 如果 propsData 中没有传入该 prop，且 prop 没有默认值，则该 prop 的值会被设置为 false。
  - 如果传入的 prop 值为空字符串或与 prop 的连字符形式相同，则该 prop 的值会被设置为 true（前提是 Boolean 类型优先于 String 类型）。

- prop.type 定义中不包含 Boolean 类型，且没有传入该 prop：

  - 该 prop 的值会取默认值。

在其他场景下，validateProp 函数会直接使用传入的 prop 值。

```js
function validateProp (
  key,
  propOptions,
  propsData,
  vm
) {
  var prop = propOptions[key];
  var absent = !hasOwn(propsData, key);
  var value = propsData[key];
  // boolean casting
  // 获取 Boolean 类型在 prop.type 中的索引
  var booleanIndex = getTypeIndex(Boolean, prop.type);
  if (booleanIndex > -1) {
    if (absent && !hasOwn(prop, 'default')) {
      // 如果 propsData 中没有该 prop 且 prop 没有默认值，将 value 设置为 false
      value = false;
    } else if (value === '' || value === hyphenate(key)) {
      // only cast empty string / same name to boolean if
      // boolean has higher priority
      // 如果 value 是空字符串或与 key 的连字符形式相同，将 value 设置为 true（前提是 Boolean 类型优先于 String 类型）
      var stringIndex = getTypeIndex(String, prop.type);
      if (stringIndex < 0 || booleanIndex < stringIndex) {
        value = true;
      }
    }
  }
  // check default value
  if (value === undefined) {
    // 如果 value 是 undefined，调用 getPropDefaultValue 获取默认值
    value = getPropDefaultValue(vm, prop, key);
    // since the default value is a fresh copy,
    // make sure to observe it.
    var prevShouldObserve = shouldObserve;
    toggleObserving(true);
    observe(value); // 将默认值转换为响应式对象
    toggleObserving(prevShouldObserve);
  }
  if (
    process.env.NODE_ENV !== 'production' &&
    // skip validation for weex recycle-list child component props
    !(false)
  ) {
    assertProp(prop, key, value, vm, absent);
  }
  return value
}
```

Vue.js 对 prop.type 的校验并不是强制性的。这意味着即使子组件定义了 prop.type 为某种类型（例如 String），父组件仍然可以传入其他类型的值（例如 Array），子组件依然可以接收并使用这个值。

```js
Vue.component('example-component', {
  props: {
    propA: {
      type: String,
      default: 'defaultA'
    }
  },
  template: `
    <div>
      <p>{{ propA }}</p>
    </div>
  `
});

new Vue({
  el: '#app',
  template: `
    <div>
      <example-component :propA="['item1', 'item2']"></example-component>
    </div>
  `
});
```

getTypeIndex 函数用于检查传入的类型是否在 prop.type 定义的类型数组中，并返回匹配类型的索引。将 prop.type 定义的类型数组中的函数转换为字符串，并使用正则表达式 functionTypeCheckRE 匹配类型名称，然后比较类型名称。

```js
var functionTypeCheckRE = /^\s*function (\w+)/;

function getType (fn) {
  var match = fn && fn.toString().match(functionTypeCheckRE);
  return match ? match[1] : ''
}

function isSameType (a, b) {
  return getType(a) === getType(b)
}

function getTypeIndex (type, expectedTypes) {
  if (!Array.isArray(expectedTypes)) {
    return isSameType(expectedTypes, type) ? 0 : -1
  }
  // prop.type 可以定义成一个数组，以表示该 prop 可以接受多种类型的值
  for (var i = 0, len = expectedTypes.length; i < len; i++) {
    if (isSameType(expectedTypes[i], type)) {
      return i
    }
  }
  return -1
}
```

#### initMethods

函数用于初始化 Vue 实例的 methods。它会将组件定义中的方法绑定到 Vue 实例上

```js
function initMethods (vm, methods) {
  var props = vm.$options.props;
  for (var key in methods) {
    if (process.env.NODE_ENV !== 'production') {
      if (typeof methods[key] !== 'function') {
        warn(
          "Method \"" + key + "\" has type \"" + (typeof methods[key]) + "\" in the component definition. " +
          "Did you reference the function correctly?",
          vm
        );
      }
      if (props && hasOwn(props, key)) {
        warn(
          ("Method \"" + key + "\" has already been defined as a prop."),
          vm
        );
      }
      if ((key in vm) && isReserved(key)) {
        warn(
          "Method \"" + key + "\" conflicts with an existing Vue instance method. " +
          "Avoid defining component methods that start with _ or $."
        );
      }
    }
    vm[key] = typeof methods[key] !== 'function' ? noop : bind(methods[key], vm);
  }
}
```

#### initData

函数用于初始化 Vue 实例的 data。它会根据组件的 data 选项，将数据对象转换为响应式对象

```js
function initData (vm) {
  var data = vm.$options.data;
  data = vm._data = typeof data === 'function'
    ? getData(data, vm) // 如果 data 是一个函数，调用 getData 函数获取数据对象
    : data || {};
  if (!isPlainObject(data)) {
    data = {}; // 如果 data 不是纯对象，将 data 设置为空对象
    process.env.NODE_ENV !== 'production' && warn(
      'data functions should return an object:\n' +
      'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
      vm
    );
  }
  // proxy data on instance
  var keys = Object.keys(data);
  var props = vm.$options.props;
  var methods = vm.$options.methods;
  var i = keys.length;
  while (i--) {
    var key = keys[i];
    if (process.env.NODE_ENV !== 'production') {
      if (methods && hasOwn(methods, key)) {
        warn(
          ("Method \"" + key + "\" has already been defined as a data property."),
          vm
        );
      }
    }
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' && warn(
        "The data property \"" + key + "\" is already declared as a prop. " +
        "Use prop default value instead.",
        vm
      );
    } else if (!isReserved(key)) {
      proxy(vm, "_data", key);
    }
  }
  // observe data
  observe(data, true /* asRootData */); // 将 data 转换为响应式对象
}
```

函数用于为一个值创建一个观察者实例，使其成为响应式对象。如果该值已经有一个观察者实例，则返回现有的观察者实例。

```js
/**
 * Attempt to create an observer instance for a value,
 * returns the new observer if successfully observed,
 * or the existing observer if the value already has one.
 */
function observe (value, asRootData) {
  if (!isObject(value) || value instanceof VNode) {
    // 如果 value 不是对象或是一个虚拟节点（VNode），则直接返回
    return
  }
  var ob;
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    // 检查值是否已有观察者实例
    ob = value.__ob__;
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    // 创建新的观察者实例
    ob = new Observer(value);
  }
  if (asRootData && ob) {
    ob.vmCount++;
  }
  return ob
}
```

Observer 类用于将对象转换为响应式对象。每个被观察的对象都会附加一个 Observer 实例，该实例会将目标对象的属性键转换为 getter/setter，以便收集依赖并分发更新。

```js
/**
 * Observer class that is attached to each observed
 * object. Once attached, the observer converts the target
 * object's property keys into getter/setters that
 * collect dependencies and dispatch updates.
 */
var Observer = function Observer (value) {
  this.value = value;
  this.dep = new Dep();
  this.vmCount = 0;
  def(value, '__ob__', this); // 使用 def 函数在 value 上定义一个不可枚举的 __ob__ 属性，指向当前的 Observer 实例
  if (Array.isArray(value)) {
    if (hasProto) {
      // 如果环境支持 __proto__，使用 protoAugment 函数将 value 数组的原型指向 arrayMethods
      protoAugment(value, arrayMethods);
    } else {
      // 否则，使用 copyAugment 函数将 arrayMethods 的方法直接拷贝到数组上。
      copyAugment(value, arrayMethods, arrayKeys);
    }
    // 调用 observeArray 方法，递归地将数组中的每个元素转换为响应式对象
    this.observeArray(value);
  } else {
    // 如果 value 是对象，调用 walk 方法，将对象的每个属性转换为响应式属性
    this.walk(value);
  }
};

/**
 * Walk through all properties and convert them into
 * getter/setters. This method should only be called when
 * value type is Object.
 */
Observer.prototype.walk = function walk (obj) {
  var keys = Object.keys(obj);
  for (var i = 0; i < keys.length; i++) {
    defineReactive$$1(obj, keys[i]);
  }
};

/**
 * Observe a list of Array items.
 */
Observer.prototype.observeArray = function observeArray (items) {
  for (var i = 0, l = items.length; i < l; i++) {
    observe(items[i]);
  }
};

/**
 * Augment a target Object or Array by intercepting
 * the prototype chain using __proto__
 */
function protoAugment (target, src) {
  /* eslint-disable no-proto */
  target.__proto__ = src;
  /* eslint-enable no-proto */
}

/**
 * Augment a target Object or Array by defining
 * hidden properties.
 */
/* istanbul ignore next */
function copyAugment (target, src, keys) {
  for (var i = 0, l = keys.length; i < l; i++) {
    var key = keys[i];
    def(target, key, src[key]);
  }
}
```

拦截数组的变异方法（如 push、pop 等），使得这些方法在修改数组时能够触发 Vue 的响应式系统

```js
var arrayProto = Array.prototype;
var arrayMethods = Object.create(arrayProto);

var methodsToPatch = [
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
];

/**
 * Intercept mutating methods and emit events
 */
methodsToPatch.forEach(function (method) {
  // cache original method
  var original = arrayProto[method];
  def(arrayMethods, method, function mutator () {
    var args = [], len = arguments.length;
    while ( len-- ) args[ len ] = arguments[ len ];

    var result = original.apply(this, args);
    var ob = this.__ob__;
    var inserted;
    switch (method) {
      case 'push':
      case 'unshift':
        inserted = args;
        break
      case 'splice':
        inserted = args.slice(2);
        break
    }
    // 如果有新插入的元素，调用 ob.observeArray(inserted) 方法将新元素转换为响应式对象
    if (inserted) { ob.observeArray(inserted); }
    // notify change
    // 调用 ob.dep.notify() 方法通知所有依赖该数组的观察者进行更新
    ob.dep.notify();
    return result
  });
});

var arrayKeys = Object.getOwnPropertyNames(arrayMethods);
```

![Img](./FILES/Vue%20构造函数以初始化.md/img-20241108103355.jpg)

Vue 2 对数据的响应式处理是通过数据劫持和发布-订阅模式实现的。Vue 使用 Object.defineProperty() 方法对数据对象的属性进行劫持，使之能够在数据变化时更新视图。具体来说，Vue 在初始化时，会对实例的 data 属性上定义的所有属性使用 Object.defineProperty() 进行转换，加上 getter/setter 转化后，任何对 data 属性的修改都会自动触发依赖收集和派发更新，从而实现数据与视图的双向绑定。

然而，当通过 push、unshift 等方法修改数组时，并不会触发响应更新。这是因为 Object.defineProperty() 使用 [[DefineOwnProperty]] 内部方法添加或修改对象上的属性，只会拦截对象的 [[Get]] 和 [[Set]] 操作。因此，在使用 push、unshift 等方法修改数组时，并不会被拦截。

为了实现数组的响应式，Vue 2 通过重写数组的原型方法来实现。当调用 push、unshift 等方法时，Vue 会拦截这些方法，并在修改数组后触发依赖更新。

在 Vue 3 中，使用 Proxy 对象来实现响应式系统，取代了 Vue 2 中使用的 Object.defineProperty 方法。Proxy 提供了一种更强大和灵活的方式来拦截和处理对对象的操作，包括对数组的操作。两者的本质区别在于，Proxy 是对所有基本操作的拦截器，而 DefineProperty 则是一个具体的操作。

#### initComputed

函数用于初始化 Vue 实例的计算属性（computed properties）。它会为每个计算属性创建一个内部观察者（Watcher），并存储到 vm.\_computedWatchers。

```js
var computedWatcherOptions = { lazy: true };

function initComputed (vm, computed) {
  // $flow-disable-line
  var watchers = vm._computedWatchers = Object.create(null);
  // computed properties are just getters during SSR
  var isSSR = isServerRendering();

  for (var key in computed) {
    var userDef = computed[key];
    // 如果 userDef 是函数，将其作为 getter；否则，将 userDef.get 作为 getter
    var getter = typeof userDef === 'function' ? userDef : userDef.get;
    if (process.env.NODE_ENV !== 'production' && getter == null) {
      warn(
        ("Getter is missing for computed property \"" + key + "\"."),
        vm
      );
    }

    if (!isSSR) {
      // create internal watcher for the computed property.
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions
      );
    }

    // component-defined computed properties are already defined on the
    // component prototype. We only need to define computed properties defined
    // at instantiation here.
    if (!(key in vm)) {
      defineComputed(vm, key, userDef);
    } else if (process.env.NODE_ENV !== 'production') {
      if (key in vm.$data) {
        warn(("The computed property \"" + key + "\" is already defined in data."), vm);
      } else if (vm.$options.props && key in vm.$options.props) {
        warn(("The computed property \"" + key + "\" is already defined as a prop."), vm);
      } else if (vm.$options.methods && key in vm.$options.methods) {
        warn(("The computed property \"" + key + "\" is already defined as a method."), vm);
      }
    }
  }
}
```

#### initWatch

initWatch 函数用于初始化 Vue 实例的 watch 选项。它会遍历 watch 对象，并为每个属性创建一个观察者（Watcher）

```js
function initWatch (vm, watch) {
  for (var key in watch) {
    var handler = watch[key];
    if (Array.isArray(handler)) {
      for (var i = 0; i < handler.length; i++) {
        createWatcher(vm, key, handler[i]);
      }
    } else {
      createWatcher(vm, key, handler);
    }
  }
}
```

watch 选项中的 handler 可以是一个数组。这样可以为同一个属性定义多个观察者（Watcher），每个观察者都有自己的处理函数

```js
Vue.component('example-component', {
  data() {
    return {
      message: 'Hello, Vue!'
    };
  },
  watch: {
    message: [
      function (newVal, oldVal) {
        console.log('Watcher 1: Message changed from', oldVal, 'to', newVal);
      },
      function (newVal, oldVal) {
        console.log('Watcher 2: Message changed from', oldVal, 'to', newVal);
      }
    ]
  },
  template: `
    <div>
      <p>{{ message }}</p>
      <input v-model="message">
    </div>
  `
});
```

### initProvide

函数用于初始化 Vue 实例的 provide 选项。provide 选项用于向下传递数据，使得子组件可以通过 inject 选项来接收这些数据。

```js
function initProvide (vm) {
  var provide = vm.$options.provide;
  if (provide) {
    vm._provided = typeof provide === 'function'
      ? provide.call(vm)
      : provide;
  }
}
```

provide 选项可以是一个对象或一个返回对象的函数。如果 provide 是一个函数，Vue 会在实例化时调用这个函数，并将其返回值作为 provide 的内容。**这使得 provide 可以动态生成数据**。

```js
Vue.component('parent-component', {
  provide() {
    return {
      users: this.users
    };
  },
  data() {
    return {
      users: ["Alice", "Bob", "Charlie"],
    }
  },
  template: `
    <div>
      <child-component></child-component>
    </div>
  `
});

Vue.component('child-component', {
  inject: ['users'],
  template: `
    <div v-for="(user, index) in users" :key="index">
      {{ user }}
    </div>
  `
});

new Vue({
  el: '#app'
});
```

initInjections 在 initProvide 之前执行是没有问题的。这是因为 initInjections 负责从父组件中查找并注入依赖，而 initProvide 负责处理当前组件的提供数据。先查找父组件的注入，再处理子组件的提供，这样的顺序是合理的。

## stateMixin

## eventsMixin

## lifecycleMixin

## renderMixin
