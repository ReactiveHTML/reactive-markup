# RML - Reactive Markup Language
RML is a "conceptual" functional-reactive extension of HTML and the DOM with first-class support for promises and observables

Both HTML and JavaScript have evolved significantly over time.
HTML introduced new tags and new magic on one end, JavaScript is creating new primites like Promises and Observables.

Despite this progress, HTML markup is still limited to strings and string representations of numbers and some basic function expressions.

```html
<div id="a-string" class="more strings" onclick="someFunction()" data-foo="bar">
  some <strong>HTML</strong> string
</div>
```

JavaScript, on the other hand, deals with a wide range of primitives and data types, let alone it can access the above through a number of different DOM APIs, like event emitters (`.on('event', fn)`), object properties (`.innerHTML`), etc.

This leads to a lot of boilerplate.

## Extending HTML
The next logical step is to make HTML support more primitives and data types natively, or more transparently to scripts.

This document is the specification of such thing, the Reactive Markup Language, or RML for short.

At this stage RML is a concept, most easily implemented by template engines, UI libraries or frameworks, either based on template literals or other abstractions like JSX/ESX.

A reference implementation is provided by [Rimmel.js](https://github.com/reactivehtml/rimmel)

# Static markup vs JSON notation
RML templates can take any of the following forms:
- Simple HTML Strings
- JavaScript tagged templates with string/nlmber/function/Promise/Observable expressions). E.g.: `<div>${someJavaScriptVariable}</div>`
- DOM Objects / JavaScript Maps
- Arrays

The JSON form is called JSON, but in reality it refers to regular JavaScript objects.
These are used to represent and map to HTML attributes, including some special-purpose ones such as `class`, `data-*`, `disabled`, event listeners and can present themselves in a variety of forms, depending on the type of attribute they represent.
  - Generic attribute objects, e.g.: `{class: ['class1', 'class2'], dataset: {'key1': 'value1'}, onclick: () => null}`
  - Single dataset objects, e.g.: `{key: 'value'}`
  - Multiple dataset objects, e.g.: `{key1: 'value1', key2: 'value2'}`
  - Class Objects, e.g.: `{class1: true, class2: false}`
  - ClassList arrays, e.g.: `['class1', 'class2']`
  - CSS Style Objects, e.g.: `{position: 'absolute', width: '100%'}`
  - CSS Style Values, e.g.: `'absolute'`, or `new Promise(resolve => setTimeout(() => resolve('100%'), 1000))`

Values that can be assigned to CSS styles, class names, attribute values, dataset items, can be either static (strings) or deferred values (promises, observables).
In the first case, they are assigned and sinked to the DOM immediately.
```<div data-title="title1">```

Promises are sinked as they resolve, Observables every time they emit.
```
  <div data-title="${somePromiseOrObservable}">
  <div class="class1 class2 ${someClassObject}">
```

Scripts can generate either and sink it into existing DOM by direct assignment, through a unified and simplified API.

E.G.:
```js
  const classes = { class1: true, class2: false }
  const template = rml`<div id="" class="${classes}">`
```

### Tagged Templates vs JSX/ESX
RML can be implemented in both tagged-template and JSX/ESX-based template engines. For convenience, in this document only tagged-templates will be used for examples, but their meaning should be equivalent in a JSX/ESX-like implementation.
```js
const content = 'some data'

const taggedTemplate = rml`
  <div>${content}</div>
`
const ESXTemplate = (
  <div>{content}</div>
)
```

### Promises
Promises are first-class citizens in RML, so they can be assigned to child nodes and attributes.
Whatever the promise resolves to, will be injected as child of the `div` below:
```js
const p = new Promise(/*...*/)

const template = rml`
  <div>${p}</div>
`
```

### Observables
Just like Promises, Observables are also first-class citizens in RML, so they can be assigned to child nodes and attributes alike.
In interactive web applications Observables can play a major role and bring many of the benefits of functional programming.

#### Child nodes
```js
const stream = Observable(/*...*/)

const template = rml`
  <div>${stream}</div>
`
```

#### Attributes
```js
const data = Observable.of({author: 'Stephen King', title: 'Misery'})
const classes = Observable.of({class1: true, class2: false})
const moreStuff = Promise.resolve({
  class: {
    class3: true,
    class4: false,
  }
  data-year: 1987,
})

const template = rml`
  <div ...${data} class="...${classes}" ...${moreStuff}>some content</div>
`
```

This should, once all observables have emitted and promises resolved, generate the following tag:
```
  <div data-author="Stephen King" data-title="Misery" class="class1 class3" data-year="1987">some content</div>
```

## Sources and Sinks
RML is a reactive markup, supporting the functional-reactive paradigm. This means some key concepts like Sources and Sinks have their special place in the syntax.

### Sources
Most HTML attributes whose name start with "on" represent event sources and are implemented as event emitters.

In RML, event sources can be connected to plain JavaScript functions or to a special, writable type of Observables typically referred to as `Subject` in RxJS.
This means, every time an event happens, a bound function will be called or a bound Subject will have its .next() method called with the corresponding HTMLEvent instance.

```js
  const handlerFunction = (e: InputEvent) => {
    console.log('A plain JS function that does something')
  }
  
  const template = rml`
    <button onclick="${handlerFunction}">click me</button>
  `
```
This is the simplest case. Button clicks will simply call the `handlerFunction`.

```js
  const handlerStream = new Subject()
    .pipe(
      // some further processing...
    )

  const template = rml`
    <button onclick="${handlerStream}">click me</button>
  `
```
In the code above, every time the button is clicked, it will call `handlerStream.next()` passing a `ClickEvent`.

By convention, every HTML attribute whose name starts with "on", will be treated as an event source.

### The rml:onmount source
There is one special event source, 'rml:onmount', not part ofthe HTML specification, which will fire immediately after a given element has been attached to the DOM.
Note how every RML attribute that's not an HTML standard is prefixed with `rml:`, similarly to the way XML Namespaces are used.

```js
  const init = (e: MountEvent) => {
    console.log('An element has been mounted')
  }

  const template = rml`
    <div rml:onmount="${handlerFunction}"></div>
  `
```

### Sinks
Sinks are the opposite of sources. Sources emit events, sinks render them to the DOM.
There are many types of sinks, depending on the use case. Typically they perform some final transformation before calling any relevant DOM API to display data.
Sinks can be implicit, when it's obvious from the syntax and the context what should happen, and explicit, where developers can request a particular sink to be used to render data.
There are three categories of sinks: single-value, multi-value, and runtime sinks, which can respectively sink one or more values every time some data is emitted.
Dynamic sinks will determine at runtime what they need to do. An `Observable<unknown>` would typically emit into a runtime sync.

#### Implicit Sinks
Following is a list of implicit sinks used in RML

##### InnerHTML Sink
This any-value sink takes a string and sets `innerHTML` on the specified node. It's used by default when a Promise or an Observable are placed as a child element of a tag:
```js
  const str = fetch(/*some.api/data*/).then(x=>x.text())

  const template = rml`
    <div>${str}</div>
  `
```

If non-string values get emitted to this sink, the following will apply.
- If it's an array with 2 elements `[string | number, Observable<string> | Promise<any>]`, then the former item will be rendered synchronously, the latter will be subscribed to and asynchronously synched on emission.
- Otherwise, array items will be concatenated. Static values immediately, deferred ones on subscription.




##### Style Object Sink
This multi-value sink takes a JavaScript Object and sets styles on the target object when an object or a deferred object (Promise<Object> or Observable<Object>) are set in a tag's `style` attribute
```js
  const styles = {
    position: 'relative',
    marginTop: '1rem',
  }

  const template = rml`
    <svg style="${styles}">
      <some-shapes />
    </svg>
  `
```
##### Style Sink
This single-value sink takes a static or deferred string and sets is as a style on the target object when set in a tag's `style` attribute's value:
```js
  const position = Promise.resolve('absolute')

  const template = rml`
    <div style="position: ${position}; top: 0; color: red;">
      Red text floating
    </div>
  `
```

##### Value Sink
This single-value sink takes a static or deferred string and sets is as the `value` on an `<input>` tag:
```js
  const laterValue = Promise.resolve('hello world')

  const template = rml`
    <input type="text" value="${laterValue}">
  `
```
  
##### Attribute Sink
This single-value sink takes a static or deferred string and sets is as the `value` for the specified attribute in a tag:
```js
  const laterValue = Promise.resolve('hello world')

  const template = rml`
    <some-tag some-attribute="${laterValue}" />
  `
```

##### Attribute Object Sink
This multi-value sink takes a static or deferred object and sets the corresponding key-value pairs as attribute-value pairs in the target tag:
```js
  const laterValue = Promise.resolve({
    style: 'position: relative',
    'data-some-key': 'some-dataset-value',
    onmouseover: mouseoverHandler,
  })

  const template = rml`
    <some-tag ...${laterValue} />
  `
```  
  
#### Explicit Sinks
Sometimes the default sinks will not be a convenient way to sink data to the DOM. In that case, it is possible to request other sinks to be used by wrappingn data in a `Sink` object.
  
  Sink objects are instances of the Sink class and have a `.sink` attribute that helps identify them.
  
  ```js
    const data = "Dirty text from <script>doSometingNasty</script> untrusted sources"
  
    const template = rml`
      <div>${InnerText(data)}</div>
    `
  ```
  In the code above, the string can be sinked to the DOM through `.innerText` for security.
  
  The following sinks need to be called explicitly:
  
- innerTextSink: same as the innerHTMLSink, except this one sets innerText on the target node.
- appendHTMLSink: similar to innerHTMLSink, this will call `.append()` on the target node.

### Initial vs deferred values
Some advanced Observable primitives like the `BehaviorSubject` have an initial static value, and then they can emit subsequent values asynchronously.
When a BehaviorSubject-like item is set in a RML template, its `.value` property will be used for the initial rendering, then its `.subscribe()` method will be called to get and sink subsequent values.
  
```js
  const stream = new BehaviorSubject('initial data')
  setInterval(() => stream.next(getSomeRandomData()), 1000)

  const template = rml`
    <div>${stream}</div>
  `
```
In this case, the `div` will be initially rendered with the text `initial data` without any <acronym title="Flash Of Unstyled Content">FOUC</acronmym>, then every second will have a new value set via `innerHTML`

## Extensible components / Mixins
Extensible components are regular HTML Elements that can be enriched by synchronous or asynchronous mixins.
Mixins are just partial RML/DOM objects used to enrich their host tags with new attributes, event handlers, classes, etc.
They can be useful to make certain HTML tags gain new functionality without repeating code, or gain it at a later point in time.
Mixins are created easily, by implicitly invoking an `Attribute Object Sink`.

```js
// Make an element "content editable" when clicked
const editableNow = {
  class: 'class1',
  onclick: e => e.target.contentEditable = true,
}

const template = rml`
  <div ...${editableNow}</div>
`
```
The same can be applied asynchronously, e.g. when some features are requested by the user:

```js
// Make an element "content editable" when clicked
const editableWhenEnabled = new Promise(/*some trigger*/)
  .then(() => ({
    class: 'class1',
    onclick: e => e.target.contentEditable = true,
  }))

const template = rml`
  <div ...${editableWhenEnabled}</div>
`
```
## RML Compliant UI libraries
  The reference implementation of RML is [Rimmel.js](https://github.com/reactivehtml/rimmel)
  
## References
There are discussions going on around making HTML and/or the DOM natively support Observables at [WHATWG DOM/544](https://github.com/whatwg/dom/issues/544) and the more recent [Observable DOM](https://github.com/WICG/observable).

