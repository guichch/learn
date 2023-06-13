## 批量更新（连续调用多次更新）
### legacy
1. 在事件回调触发
    1. 同步触发：在触发回调时会调用batchedUpdate方法，因此会将多次更新合并为一次，表现为异步
    2. 异步触发：在触发回调时会调用bachedUpdate方法，但是异步代码执行时，batchedContext已经复原为NoContext，因此每次更新触发一次render，表现为同步
2. 在生命周期中触发
    1. 同步触发：不会在scheduleUpdateOnFiber方法中触发flushSyncCallbacksOnlyInLegacyMode，因为此时的context不是NoContext，因此多次更新会合并为一次，表现为异步
    2. 异步触发：会在scheduleUpdateOnFiber方法中触发flushSyncCallbacksOnlyInLegacyMode，因为此时的context已经复原成NoContext，因此每次更新触发一次render，表现为同步
    3. 不管是哪一个生命周期都是一样的，因此只要在执行声明周期，表明当前context处于renderContext || commitContext，而且执行过程是同步的，声明周期中的异步代码执行时，此时的context一定已经复原成NoContext了，除非原始的context就不是NoContext(有无这种情况呢？)
### concurrent
1. 在事件回调触发
    1. 同步触发：在触发回调时会调用batchedUpdate方法，且scheduleUpdateOnFiber中的判断条件一定不会成立，因此会将多次更新合并为一次，表现为异步
    2. 异步触发：在触发回调时会调用batchedUpdate方法，但是异步代码执行时，batchedContext已经复原为NoContext，但scheduleUpdateOnFiber中的判断条件不会成立，因此会将多次更新合并为一次，表现为异步
    3. 通过flushSync触发：无论是在同步代码中还是在异步代码中，flushSync执行时，context一定不是renderContext || commitContext，因此每次更新会触发一次render，表现为同步
2. 在生命周期中触发
    1. 同步触发：不会在scheduleUpdateOnFiber方法中触发flushSyncCallbacksOnlyInLegacyMode，此时的context不是NoContext，判断不会成立，因此多次更新会合并为一次，表现为异步
    2. 异步触发：不会在scheduleUpdateOnFiber方法中触发flushSyncCallbacksOnlyInLegacyMode，此时的context是NoContext，但concurrent模式下判断不会成立，因此多次更新会合并为一次，表现为异步
    3. 通过flushSync触发
        1. 同步触发：执行flushSync时的context一定包含renderContext || commitContext，因此flushSync不会发生作用，多次更新会被合并为一次，表现为异步
        2. 异步触发：执行flushSync时的context一定不包含renderContext || commitContext，因此flushSync会发生作用，每次更新触发一次render，表现为同步

## Hooks
### 为什么发明hooks
1. class component 逻辑复用困难。
  举个栗子：组件A和组件B都需要浏览器窗口大小的信息复用浏览器窗口大小的内容，当浏览器大小发生变化时，获取变化后的值
  class component 有几个方法处理这个问题
    1. 组件A和组件B都监听浏览器窗口大小变化
    2. 高阶组件，在高阶组件中监听浏览器大小的变化，然后通过props传给组件A和组件B
    3. renderProps，在公共组件中监听浏览器大小的变化，在公共组件中通过参数传给组件A和组件B
    ```javascript
    class WindowSize extends React.Component {
      state = {
        size: this.getSize()
      }
      getSize() {
        return window.innerWidth > 1000 ? 'large' : 'small'
      }
      // 能否只监听一次resize方法呢？不能
      componentDidMount() {
        window.addEventListener('resize', function(e) {
          this.setState({
            size: this.getSize()
          })
        })
      }
      render() {
        return this.props.render(this.state.size)
      }
    }
    ```
  hooks处理这个问题
  ```javascript
  const useWindowSize = () => {
    const getSize = () => window.innerWidth > 1000 ? 'large' : 'small'
    const [size, setSize] = useState(getSize())
    useEffect(() => {
      window.addEventListener('resize', function() {
        setSize(getSize())
      })
    }, [])
    return size
  }
  ```
2. class component 代码维护困难，由于生命周期的原因，控制某一位置的逻辑必须要分别写在不同的生命周期中，不能做到关注分离
### 都有哪些hooks，是如何实现的
1. useCallback
    这个hook的使用场景是在父组件中定义一个function传给子组件的时候，保证传给子组件的function地址引用是一样的
    ```javascript
    function mountCallback<T>(callback: T, deps: Array<mixed> | void | null): T {
      const hook = mountWorkInProgressHook(); // 将hooks挂在到fiber上
      const nextDeps = deps === undefined ? null : deps;
      hook.memoizedState = [callback, nextDeps]; // 存储callback and dep
      return callback; // return callback
    }
    function updateCallback<T>(callback: T, deps: Array<mixed> | void | null): T {
      const hook = updateWorkInProgressHook();
      const nextDeps = deps === undefined ? null : deps;
      const prevState = hook.memoizedState;
      if (prevState !== null) {
        if (nextDeps !== null) {
          const prevDeps: Array<mixed> | null = prevState[1];
          if (areHookInputsEqual(nextDeps, prevDeps)) { // 通过object.is判断各个元素是否相等，所以这个位置是有一次遍历成本的
            return prevState[0];
          }
        }
      }
      hook.memoizedState = [callback, nextDeps];
      return callback;
    }
    ```
2. useContext
    1. readContext // context的相关逻辑
3. useEffect 会在浏览器绘制完页面之后执行，我们的副作用没有特殊情况均使用这个hook
  执行时机
    1. fiber树没有在commit阶段调用更新，那么useEffect会异步执行。因为useEffect的执行会通过schedule调度，肯定是异步的
    2. fiber树有在commit阶段中调用更新，那么useEffect会同步执行。因为在commit阶段调用更新之后，在commit阶段结束后立即flush掉同步更新，在render之前会flush掉passive effect，因此表现出来的结果是useEffect是同步执行的且在更新的render之前执行。
  是否会阻碍浏览器绘制页面？
    1. useEffect异步执行，不会阻碍
    2. useEffect同步执行，会阻碍
4. useLayoutEffect 会在react执行完DOM API，但是浏览器还没有进行绘制页面时执行，因此这个hook会阻碍浏览器绘制页面。这个hook中所有的update也会在浏览器绘制页面之前被执行，为什么呢？因为在这个hook中调用update之后，会在下面执行flushSyncCallbacks，将同步任务flush掉。因此虽然会执行两次render，但浏览器页面只绘制一次
5. useInsertionEffect 会在react执行DOM API之前执行，这个hook是提供给CSS-IN-JS作者的，开发者不应该使用它
    三个effect的实现方式是一样的，只不过传的fiberFlags和hookFlags不同
    fiberFlags: Passive | Update
    hookFlags: HookPassive | HookInsertion | HookLayout，不同的flag表示不同的effect
    ```javascript
    function mountEffect(
      create: () => (() => void) | void,
      deps: Array<mixed> | void | null,
    ): void {
      return mountEffectImpl(
        PassiveEffect | PassiveStaticEffect, // fiberFlags
        HookPassive, // hookFlags
        create,
        deps,
      )
    }
    function updateEffect(
      create: () => (() => void) | void,
      deps: Array<mixed> | void | null,
    ) {
      return updateEffectImp(PassiveEffect, HookPassive, create, deps)
    }
    function mountEffectImpl(fiberFlags, hookFlags, create, deps): void {
      const hook = mountWorkInProgressHook();
      const nextDeps = deps === undefined ? null : deps;
      currentlyRenderingFiber.flags |= fiberFlags; // 把fiberFlags挂载到fiber上，flag会在commit阶段处理
      hook.memoizedState = pushEffect( // 将effect信息挂载到hook上，同时将effect也挂载到fiber.updateQueue上
        HookHasEffect | hookFlags,
        create,
        undefined,
        nextDeps,
      );
    }
    function updateEffectImpl(fiberFlags, hookFlags, create, deps): void {
      const hook = updateWorkInProgressHook();
      const nextDeps = deps === undefined ? null : deps;
      let destroy = undefined;

      if (currentHook !== null) {
        const prevEffect = currentHook.memoizedState;
        destroy = prevEffect.destroy;
        if (nextDeps !== null) {
          const prevDeps = prevEffect.deps;
          if (areHookInputsEqual(nextDeps, prevDeps)) {
            hook.memoizedState = pushEffect(hookFlags, create, destroy, nextDeps);
            return;
          }
        }
      }

      currentlyRenderingFiber.flags |= fiberFlags;

      hook.memoizedState = pushEffect(
        HookHasEffect | hookFlags,
        create,
        destroy,
        nextDeps,
      );
    }
    ```
    上述代码是在effect执行时将对应flag,effect挂在到fiber上，effect真正执行是在commit阶段
6. useImperativeHandle // 用于父组件调用子组件内的方法，和forwardRef搭配使用
7. useMemo
    ```javascript
    function mountMemo<T>(
      nextCreate: () => T,
      deps: Array<mixed> | void | null,
    ): T {
      const hook = mountWorkInProgressHook();
      const nextDeps = deps === undefined ? null : deps;
      const nextValue = nextCreate();
      hook.memoizedState = [nextValue, nextDeps];
      return nextValue;
    }
    function updateMemo<T>(
      nextCreate: () => T,
      deps: Array<mixed> | void | null,
    ): T {
      const hook = updateWorkInProgressHook();
      const nextDeps = deps === undefined ? null : deps;
      const prevState = hook.memoizedState;
      if (prevState !== null) {
        // Assume these are defined. If they're not, areHookInputsEqual will warn.
        if (nextDeps !== null) {
          const prevDeps: Array<mixed> | null = prevState[1];
          if (areHookInputsEqual(nextDeps, prevDeps)) {
            return prevState[0];
          }
        }
      }
      const nextValue = nextCreate();
      hook.memoizedState = [nextValue, nextDeps];
      return nextValue;
    }
    ```
8. useReducer
    ```javascript
    function mountReducer<S, I, A>(
      reducer: (S, A) => S,
      initialArg: I,
      init?: I => S,
    ): [S, Dispatch<A>] {
      const hook = mountWorkInProgressHook();
      let initialState;
      if (init !== undefined) {
        initialState = init(initialArg);
      } else {
        initialState = initialArg;
      }
      hook.memoizedState = hook.baseState = initialState;
      const queue: UpdateQueue<S, A> = {
        pending: null,
        interleaved: null,
        lanes: NoLanes,
        dispatch: null,
        lastRenderedReducer: reducer,
        lastRenderedState: initialState,
      };
      hook.queue = queue;
      const dispatch: Dispatch<A> = (queue.dispatch = (dispatchReducerAction.bind(
        null,
        currentlyRenderingFiber,
        queue,
      ): any));
      return [hook.memoizedState, dispatch];
    }
    function updateReducer<S, I, A>(
      reducer: (S, A) => S,
      initialArg: I,
      init?: I => S,
    ): [S, Dispatch<A>] {
      const hook = updateWorkInProgressHook();
      const queue = hook.queue;

      if (queue === null) {
        throw new Error(
          'Should have a queue. This is likely a bug in React. Please file an issue.',
        );
      }

      queue.lastRenderedReducer = reducer;

      const current: Hook = currentHook;

      // The last rebase update that is NOT part of the base state.
      let baseQueue = current.baseQueue;

      // The last pending update that hasn't been processed yet.
      const pendingQueue = queue.pending;
      if (pendingQueue !== null) {
        // We have new updates that haven't been processed yet.
        // We'll add them to the base queue.
        if (baseQueue !== null) {
          // Merge the pending queue and the base queue.
          const baseFirst = baseQueue.next;
          const pendingFirst = pendingQueue.next;
          baseQueue.next = pendingFirst;
          pendingQueue.next = baseFirst;
        }
        if (__DEV__) {
          if (current.baseQueue !== baseQueue) {
            // Internal invariant that should never happen, but feasibly could in
            // the future if we implement resuming, or some form of that.
            console.error(
              'Internal error: Expected work-in-progress queue to be a clone. ' +
                'This is a bug in React.',
            );
          }
        }
        current.baseQueue = baseQueue = pendingQueue;
        queue.pending = null;
      }

      if (baseQueue !== null) {
        // We have a queue to process.
        const first = baseQueue.next;
        let newState = current.baseState;

        let newBaseState = null;
        let newBaseQueueFirst = null;
        let newBaseQueueLast = null;
        let update = first;
        do {
          const updateLane = update.lane;
          if (!isSubsetOfLanes(renderLanes, updateLane)) {
            // Priority is insufficient. Skip this update. If this is the first
            // skipped update, the previous update/state is the new base
            // update/state.
            const clone: Update<S, A> = {
              lane: updateLane,
              action: update.action,
              hasEagerState: update.hasEagerState,
              eagerState: update.eagerState,
              next: null,
            };
            if (newBaseQueueLast === null) {
              newBaseQueueFirst = newBaseQueueLast = clone;
              newBaseState = newState;
            } else {
              newBaseQueueLast = newBaseQueueLast.next = clone;
            }
            // Update the remaining priority in the queue.
            // TODO: Don't need to accumulate this. Instead, we can remove
            // renderLanes from the original lanes.
            currentlyRenderingFiber.lanes = mergeLanes(
              currentlyRenderingFiber.lanes,
              updateLane,
            );
            markSkippedUpdateLanes(updateLane);
          } else {
            // This update does have sufficient priority.

            if (newBaseQueueLast !== null) {
              const clone: Update<S, A> = {
                // This update is going to be committed so we never want uncommit
                // it. Using NoLane works because 0 is a subset of all bitmasks, so
                // this will never be skipped by the check above.
                lane: NoLane,
                action: update.action,
                hasEagerState: update.hasEagerState,
                eagerState: update.eagerState,
                next: null,
              };
              newBaseQueueLast = newBaseQueueLast.next = clone;
            }

            // Process this update.
            if (update.hasEagerState) { // 性能优化的逻辑，仅有dispatchSetState方法才会有
              // If this update is a state update (not a reducer) and was processed eagerly,
              // we can use the eagerly computed state
              newState = update.eagerState;
            } else {
              const action = update.action;
              newState = reducer(newState, action);
            }
          }
          update = update.next;
        } while (update !== null && update !== first);

        if (newBaseQueueLast === null) {
          newBaseState = newState;
        } else {
          newBaseQueueLast.next = newBaseQueueFirst;
        }

        // Mark that the fiber performed work, but only if the new state is
        // different from the current state.
        if (!is(newState, hook.memoizedState)) {
          markWorkInProgressReceivedUpdate();
        }

        hook.memoizedState = newState;
        hook.baseState = newBaseState;
        hook.baseQueue = newBaseQueueLast;

        queue.lastRenderedState = newState;
      }

      // Interleaved updates are stored on a separate queue. We aren't going to
      // process them during this render, but we do need to track which lanes
      // are remaining.
      const lastInterleaved = queue.interleaved;
      if (lastInterleaved !== null) {
        let interleaved = lastInterleaved;
        do {
          const interleavedLane = interleaved.lane;
          currentlyRenderingFiber.lanes = mergeLanes(
            currentlyRenderingFiber.lanes,
            interleavedLane,
          );
          markSkippedUpdateLanes(interleavedLane);
          interleaved = interleaved.next;
        } while (interleaved !== lastInterleaved);
      } else if (baseQueue === null) {
        // `queue.lanes` is used for entangling transitions. We can set it back to
        // zero once the queue is empty.
        queue.lanes = NoLanes;
      }

      const dispatch: Dispatch<A> = queue.dispatch;
      return [hook.memoizedState, dispatch];
    }
    ```
9. useRef
    ```javascript
    function mountRef<T>(initialValue: T): {current: T} {
      const hook = mountWorkInProgressHook();
      const ref = {current: initialValue};
      hook.memoizedState = ref;
      return ref;
    }
    function updateRef<T>(initialValue: T): {current: T} {
      const hook = updateWorkInProgressHook();
      return hook.memoizedState;
    }
    ```
10. useState
    ```javascript
    function mountState<S>(
      initialState: (() => S) | S,
    ): [S, Dispatch<BasicStateAction<S>>] {
      const hook = mountWorkInProgressHook();
      if (typeof initialState === 'function') {
        initialState = initialState();
      }
      hook.memoizedState = hook.baseState = initialState;
      const queue: UpdateQueue<S, BasicStateAction<S>> = {
        pending: null,
        interleaved: null,
        lanes: NoLanes,
        dispatch: null,
        lastRenderedReducer: basicStateReducer,
        lastRenderedState: initialState,
      };
      hook.queue = queue;
      const dispatch: Dispatch<
        BasicStateAction<S>,
      > = queue.dispatch = dispatchSetState.bind(
        null,
        currentlyRenderingFiber,
        queue,
      );
      return [hook.memoizedState, dispatch];
    }
    function updateState<S>(
      initialState: (() => S) | S,
    ): [S, Dispatch<BasicStateAction<S>>] {
      // updateState是通过updateReducer实现的，区别在于reducer是在react内部定义的而不是用户自己定义的
      return updateReducer(basicStateReducer, initialState);
    }
    ```
11. useDebugValue
12. useDeferredValue
    ```javascript
    function mountDeferredValue<T>(value: T): T {
      const hook = mountWorkInProgressHook();
      hook.memoizedState = value;
      return value;
    }
    function updateDeferredValue<T>(value: T): T {
      const hook = updateWorkInProgressHook();
      const resolvedCurrentHook: Hook = currentHook;
      const prevValue: T = resolvedCurrentHook.memoizedState;
      return updateDeferredValueImpl(hook, prevValue, value); // 这个方法会判断当前是否有高优先级任务，如果有则返回preValue，如果没有则返回value，并标记一个transition更新，重新render
    }
    ```
13. useTransition
    ```javascript
    function mountTransition(): [
      boolean,
      (callback: () => void, options?: StartTransitionOptions) => void,
    ] {
      const [isPending, setPending] = mountState(false);
      // The `start` method never changes.
      const start = startTransition.bind(null, setPending);
      const hook = mountWorkInProgressHook();
      hook.memoizedState = start;
      return [isPending, start];
    }
    function updateTransition(): [
      boolean,
      (callback: () => void, options?: StartTransitionOptions) => void,
    ] {
      const [isPending] = updateState(false);
      const hook = updateWorkInProgressHook();
      const start = hook.memoizedState;
      return [isPending, start];
    }
    ```
14. useMutableSource
15. useSyncExternalStore
16. useId 服务端渲染用的
### hooks的使用有哪些限制条件，为什么有这些条件
1. 有哪些条件？
    1. 只能在函数组件最顶层调用，且不能在循环、判断、子函数中调用
2. 为什么有这些条件？
    1. 在实现中来看，react内部是将一个fiber的所有hooks以链表的形式顺序放在fiber.memoizedState上的，且wip会将current中的hook拿过来，所以需要保持顺序且数量一致
    2. react为什么要这么实现呢？因为react需要一个位置来保存hook的信息，比如state的值，react需要将state的值保存下来，以便每次执行renderWidthHooks时都能拿到正确的state的值

## react diff
### 比较对象：react element(通过执行对应的组件获取的jsx的内容)和current fiber。current树中对应层的fiber节点
### 单点diff（reconcileSingleElement）
1. key相同（单个节点我们一般不是手动给予key值，因此这个条件通常是成立的，因为key都是null）
    1. type相同：则命中，复用当前current fiber
    2. type不同：未命中，打上删除标记，继续遍历currentFiber.sibling
    3. current fiber遍历完成后仍未命中，则以react element生成fiber
2. key不同
    1. 将current fiber打上删除标记，继续遍历currentFiber.sibling
### 多点diff（reconcileChildrenArray）
1. 遍历reactElementArray 和 current，此轮遍历用于处理更新的元素
    1. key相同，判断是否能够复用，能复用则复用，不能复用则生成
    2. key不同，终止遍历
2. reactElementArray遍历完了，将current剩余的fiber全部标记为删除，结束。处理删除了元素
3. current遍历完了，通过剩下的reactElement生成fiber，结束。处理新增了元素
4. reactElementArray和current都没有遍历完。处理有元素位置发生了变化
    1. 将current剩下的所有fiber收集到一个map中，map的key是current的key || index，map的value是current的value
    2. 继续遍历reactElementArray，如果在mapRemainingChildren中找到了对应的reactElement对应的key || index，则查看能否复用，能则复用，不能则生成
    3. 如果没能找到对应的key || index，则通过reactElement生成fiber
