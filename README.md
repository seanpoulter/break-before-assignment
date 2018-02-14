# Adding a Breakpoint On Assignment

## Usage

Paste this into your Dev Tools Console:
```js
var breakBeforeAssignment = breakBeforeAssignment || (() => {
  const enabledByTarget = new WeakMap();
  const targetByProxy = new WeakMap();

  const conditionEnabled = target => enabledByTarget.get(target) === true;

  const setEnabled = (arg, value) => {
    const target = enabledByTarget.has(arg) ? arg : targetByProxy.get(arg);
    if (!target) {
      throw new Error(`Could not find the target or proxy for ${arg}`);
    }

    enabledByTarget.set(target, value);
  };

  const withConditionalBreakpointBeforeAssignment = (target, property, value) => {
    if (conditionEnabled(target)) {
      debugger;
    }
    target[property] = value;
    return true;
  };

  const createProxy = (enabled, target) => {
    enabledByTarget.set(target, enabled);
    const proxy = new Proxy(target, { set: withConditionalBreakpointBeforeAssignment });
    targetByProxy.set(proxy, target);
    return proxy;
  };

  return {
    enable: arg => setEnabled(arg, true),
    disable: arg => setEnabled(arg, false),
    on: target => createProxy(true, target),
    whenEnabled: target => createProxy(false, target),
  };
})();
```


**Break when the value changes**:
```js
let target = [];
target = breakBeforeAssignment.on(target);
target.push(1);
// ^ Will invoke the debugger
```


**Enable and disable the breakpoint using the Proxy**:
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


**Enable and disable the breakpoint using the original Object**:
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
