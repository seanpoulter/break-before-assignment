# Start the debugger when a value[<sup>*</sup>](#issues) changes

Paste the following in your Dev Tools Console:
```js
var breakBeforeAssignment = breakBeforeAssignment || (() => {
  const enabledByTarget = new WeakMap();
  const targetByProxy = new WeakMap();
  const predicateByTarget = new WeakMap();

  const conditionEnabled = target => enabledByTarget.get(target) === true;

  const getTarget = targetOrProxy => {
    const target = targetByProxy.get(targetOrProxy) || targetOrProxy
    if (target) {
      return target;
    }

    throw new Error(`Could not find the target or proxy for ${arg}`);
  }

  const setEnabled = (arg, value) => {
    const target = getTarget(arg);
    enabledByTarget.set(target, value);
  };

  const checkPredicate = (obj, prop, value) =>
    predicateByTarget.has(obj) ? predicateByTarget.get(obj)(obj, prop, value) : true;

  const setPredicate = (targetOrProxy, predicate) => {
    const target = getTarget(targetOrProxy);
    predicateByTarget.set(target, predicate);
  };

  const deletePredicate = targetOrProxy => {
    const target = getTarget(targetByProxy);
    predicateByTarget.delete(target);
  };

  const withConditionalBreakpointBeforeAssignment = (obj, prop, value) => {
    if (conditionEnabled(obj) && checkPredicate(obj, prop, value)) {
      debugger;
    }
    obj[prop] = value;
    return true;
  };

  const createProxy = (target, enabled, predicate) => {
    enabledByTarget.set(target, enabled);
    predicateByTarget.set(target, predicate);
    const proxy = new Proxy(target, { set: withConditionalBreakpointBeforeAssignment });
    targetByProxy.set(proxy, target);
    return proxy;
  };

  return {
    disable: targetOrProxy => setEnabled(targetOrProxy, false),
    enable: targetOrProxy => setEnabled(targetOrProxy, true),
    on: target => createProxy(target, true),
    predicates: {
      set: setPredicate,
      delete: deletePredicate,
    },
    whenEnabled: target => createProxy(target, false),
    withPredicate: (target, predicate) => createProxy(target, true, predicate),
  };
})();
```


## Break when the value changes

```js
let target = [];
target = breakBeforeAssignment.on(target);
target.push(1);
// ^ Will invoke the debugger
```


## Enable and disable the breakpoint using the Proxy

```js
let target = {};
target = breakBeforeAssignment.whenEnabled(target);
target.prop = 'value';

breakBeforeAssignment.enable(target);
target.prop = 'new value';
// ^ Will invoke the debugger

breakBeforeAssignment.disable(target);
target.prop = 'value';
// ^ Doesn't interfere with your workflow
```


## Enable and disable the breakpoint using the original Object

```js
let target = {};
const orig = target;
target = breakBeforeAssignment.whenEnabled(target);
target.prop = 'value';

breakBeforeAssignment.enable(orig);
target.prop = 'new value';
// ^ Will invoke the debugger

breakBeforeAssignment.disable(orig);
target.prop = 'value';
// ^ Doesn't interfere with your workflow
```


## Advanced filtering

For situations where you'd like even more control over when the debugger is invoked you can provide your own function. When the breakpoint is enabled the predicate is called with the same arguments as the `set` trap we're hooked into: `(target: Object, name: String | Symbol, value: any) => any`.

```js
let target = {};
const nestedKeyIsBlank = (target, prop, value) => value && value.nested && value.nested.key === '';
target = breakBeforeAssignment.withPredicate(target, nestedKeyIsBlank);

// Change the predicate as you work
breakBeforeAssignment.predicates.set(target, (target, prop, value) => value > 10);

// Or remove it to use enabled/disabled only
breakBeforeAssignment.predicates.delete(target);
```


## Issues

* This only works for objects. Primitive types will give you trouble.
* Good luck if you have to reassign a `const`
