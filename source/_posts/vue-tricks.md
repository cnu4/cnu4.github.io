---
title: Vue 实战技巧
date: 2020-02-03 21:15:00
tags:
	- vue
---

虽然在日常使用 Vue 已经很熟悉了，但是在开发中还是会遇到一些一时解释不了的原因，其实还是自己对原理的理解不够深刻。下面总结了一些在日常开发的遇到的问题的一些思考，还有 Vue 的一些小技巧。

<!-- more -->

## v-if 会复用已有的元素

Vue 对渲染做了非常多的优化，经常在可以复用元素的时候会尽量复用元素，避免从头开始渲染。

比如下面的代码示例

```html
<template v-if="loginType === 'username'">
  <label>Username</label>
  <input placeholder="Enter your username">
</template>
<template v-else>
  <label>Email</label>
  <input placeholder="Enter your email address">
</template>
```

上面的代码中通过 `loginType` 切换两个不同的 template 的时候，因为两个模板使用了相同的元素，Vue 会复用 `input` 元素，只通过替换 `placeholder` 来切换。

这样做除了渲染非常快，还可以保留输入框中的文本。你可以自己实际操作试试。

但是这样也会导致一些不符合需求的情况发生，因为不同的状态切换时他总是同一个元素，如果我们有需要手动从操作元素操作 DOM 的时候，则会导致以为情况的发生。

这个时候，我们只需要为元素添加一个 `key` 属性即可。

```html
<template v-if="loginType === 'username'">
  <label>Username</label>
  <input placeholder="Enter your username" key="username-input">
</template>
<template v-else>
  <label>Email</label>
  <input placeholder="Enter your email address" key="email-input">
</template>
```

这样，每次切换状态时，输入框都讲被重新渲染。

## 父子组件传值

### 1. props/$emit

这是最常见的一种父子组件传值的方式。

通过 `props` 我们可以将值从父组件传递到子组件。

`$emit` 可以使我们出发一个事件，通过参数将值从子组件传递给父组件。

### 2. ref/refs

在元素上定义 ref 属性，再通过 refs 获取。如果 ref 是定义在 DOM 元素上，获取的则是 DOM 元素的引用，如果是用在组件上，获取的则是组件的示例，我们可以通过这个示例直接调用组件的方法和访问组件间的数据。

下面看一个例子。

```html
// 子组件 child.vue
<script>
export default {
  data () {
    return {
      name: 'foo'
    }
  },
  methods: {
    say () {
      console.log('foo component')
    }
  }
}
</script>

// 父组件 app.vue
<template>
  <child ref="child"></child>
</template>
<script>
  export default {
    mounted () {
      const comA = this.$refs.child;
      console.log(comA.name);  // foo
      comA.say();  // foo component
    }
  }
</script>

```

### 3. $children/$parent

在子组件中可以用 `this.$parent` 访问父组件的实例，而子组件的示例则会被推入父组件示例的 `$children` 数组中。

与 props/\$emit 一样，这两种方式都是用于父子组件的通信，但是 props 的通信方式更加普遍，而且官方也建议节制地使用 $children/$parent, 更多的应该作为应急方案使用。


### 4. provide/inject

`$parent` property 无法很好的扩展到更深层级的嵌套组件上。Vue2.2 之后提供了两个新的示例选项 `provide` 和 `inject`。父组件通过 `provide` 来提供变量，子组件通过 `inject` 来注入变量，父组件借此可以将值注入到深层次的子组件中。

下面看就看一个例子。

```html
// foo.vue

<template>
  <div>
	<bar></bar>
  </div>
</template>

<script>
  import Bar from '.bar.vue'
  export default {
    name: "Foo",
    provide: {
      for: "demo"
    },
    components:{
      bar
    }
  }
</script>


// bar.vue

<template>
  <div>
    {{demo}}
    <comC></comC>
  </div>
</template>

<script>
  import baz from './baz.vue'
  export default {
    name: "Bar",
    inject: ['for'],
    data() {
      return {
        demo: this.for
      }
    },
    components: {
      baz
    }
  }
</script>


// baz.vue
<template>
  <div>
    {{demo}}
  </div>
</template>

<script>
  export default {
    name: "Baz",
    inject: ['for'],
    data() {
      return {
        demo: this.for
      }
    }
  }
</script>

```

另外的对于非父子组件之间的复杂状态的管理，建议引入 eventBus 和 Vuex 进行管理。这两种方式将在其他文章中进行介绍。

## 组件 v-model

Vue 2.0 中 v-model 内置的 `v-model` 指令是一个语法糖，可以拆解为 props:value 和 events:input。只要在组件中提供一个名为 value 的 props，以及名为 input 的自定义事件，满足这两个条件，父组件中就能在子组件上使用 `v-model` 指令。

```html
<template>
  <div>
    <button @click="changeValue(-1)">-1</button>
    <span>{{val}}</span>
    <button @click="changeValue(1)">+1</button>
  </div>
</template>

<script>
export default {
  props: {
    value: {
      type: Number
    }
  },
  data() {
    return {
      val: this.value
    };
  },
  methods: {
    changeVal(val) {
      this.val += parseInt(val);
      this.$emit("input", this.val); // 定义input事件
    }
  }
};
</script>
```

只需要通过以下方式就能使用 `v-model`

`<counter v-model="val"/>`

使用 `v-model` 指令可以方便的在子组件中同步父组件的数据。2.2 后的版本中，可以定制 `v-model` 指令的 prop 和 event 名称。参考下面的代码

```js
export default {
  model: {
    prop: 'value',
    event: 'input'
  }
}

```

## Vue 动态编译模板

思考一下以下需求：在多语言需求的项目中，我们需要在页面上显示的某一段文案显示一个按钮，并且这个按钮点击后有相应的点击事件触发。如下面代码所示，ID 为 `myBtn` 的按钮就是在文案中定义的按钮。

`{ text: 'I\'m the <button id="myBtn">Button</button>' }`

我们可以将整段文案当成 html 渲染，并在组件的 `mounted` 事件后获取改按钮，并注册点击事件。代码如下所示。那有没有其他方式实现呢，操作 DOM 元素总显得有点不够优雅。

```html
<template>
  <div v-html="$t(text)"></div>
</template>

<script>
export default {
  data() {
    return {
      text: 'I\'m the <button id="myBtn">Button</button>'
    };
  },
  mounted () {
    const $el = document.getElementById('myBtn')
    $el.addEventListener('click', handleClick.bind(this))
  },
  methods: {
    handleClick () {
      console.log('click')
    }
  }
};
</script>
```

答案是有的，我们可以使用组件的实例属性 `template` 来动态定义组件的模板，组件中使用 `this.$options.template` 获取和定义模板。上面的例子可以使用下面的方式代替。注意，我们不需要在 Vue 单文件里定义 template 模块了。

```html
<script>
export default {
  data() {
    return {
      text: 'I\'m the <button id="myBtn" @click="handleClick">Button</button>
    };
  },
  created () {
    this.$options.template = this.text
  },
  methods: {
    handleClick () {
      console.log('click')
    }
  }
};
</script>
```