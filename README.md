# node-core-study
node源码学习笔记

### event
- 基础
  - 重要性：许多核心模块都是基于Events来实现的。例如Stream是基于Events实现，而net和http等是基于Stream来实现的
  - 注意：
        与on相比，once只触发一次回调。移除监听器用removeListener，移除所有的监听器用removeAllListeners。
        也可以监听一个事件，从而来追踪新的监听器。通过监听error事件，从而管理异常。
  - 例子：
    ```
      const util = require('util');
      const EventEmitter = require('events').EventEmitter;
      const myEmitter = new EventEmitter();
      const connection = (id) => {
        console.log('client id: ' + id);
      };
      myEmitter.once('connection', connection);
      // 移除监听器removeListener
      // myEmitter.removeListener('connection', connection);
      myEmitter.emit('connection', 6); // 6
      myEmitter.emit('connection', 7); // 无效
      
      myEmitter.once('newListener', (event, listener) => {
        if (event === 'event') {
          myEmitter.on('event', () => {
            console.log('B');
          });
        }
      });
      myEmitter.on('event', () => {
        console.log('A');
      });
      myEmitter.emit('event'); // B A

      myEmitter.on('error', (err) => {
        console.log('有错误');
      });
      myEmitter.emit('error', new Error('whoops!'));
      myEmitter.emit('HelloWorld');
    ```
- 源码
  - 最大监听器数量
    - 默认值：10
    - 源码：
      - 文件路径: /lib/events.js
      - 代码：
        ```
          let defaultMaxListeners = 10;
          EventEmitter.prototype._maxListeners = undefined;

          // Obviously not all Emitters should be limited to 10. This function allows
          // that to be increased. Set to zero for unlimited.
          EventEmitter.prototype.setMaxListeners = function setMaxListeners(n) {
            if (typeof n !== 'number' || n < 0 || NumberIsNaN(n)) {
              throw new ERR_OUT_OF_RANGE('n', 'a non-negative number', n);
            }
            this._maxListeners = n;
            return this;
          };

          function _getMaxListeners(that) {
            if (that._maxListeners === undefined)
              return EventEmitter.defaultMaxListeners;
            return that._maxListeners;
          }
          EventEmitter.prototype.getMaxListeners = function getMaxListeners() {
            return _getMaxListeners(this);
          };

          ObjectDefineProperty(EventEmitter, 'defaultMaxListeners', {
            enumerable: true,
            get: function() {
              return defaultMaxListeners;
            },
            set: function(arg) {
              if (typeof arg !== 'number' || arg < 0 || NumberIsNaN(arg)) {
                 throw new ERR_OUT_OF_RANGE('defaultMaxListeners', 'a non-negative number', arg);
              }
              defaultMaxListeners = arg;
            }
          });

        ```
        上面的代码很清晰，也很简单。当用户调用getMaxListeners获取最大监听数时，然后getMaxListeners内部调用_getMaxListeners方法，获取最大监听数。如果用户没有设置最大监听数，那么_maxListeners就为undefined，这时就返回默认的defaultMaxListeners，也就是10.如果用户调用setMaxListeners设置最大监听数，那么内部就将设置的值赋值给_maxListeners，此时调用getMaxListeners就返回用户设置的值。
        在获取最大监听数时，使用了Object.defineProperty的数据劫持。
    - 实例：
      ```
        const myEmitter = new EventEmitter();
        const connection = (id) => {
          console.log('client id: ' + id);
        };
        console.log(myEmitter.getMaxListeners()); // 输出10
        myEmitter.setMaxListeners(20);
        console.log(myEmitter.getMaxListeners()); // 输出20
      ```
  - 事件监听
    - addListener和on
        ```
        function _addListener(target, type, listener, prepend) { // _addListener供addListener、on和prependListener调用
          let m;
          let events;
          let existing;
          
          // 判断是不是一个函数，不是一个回调函数就报错
          checkListener(listener);
          
          // 获取当前作用域的_events。如果没有init，那么_events就不存在。于是就创建一个空对象并初始化_eventsCount
          events = target._events;
          if (events === undefined) {
            events = target._events = ObjectCreate(null);
            target._eventsCount = 0;
          } else {
            // To avoid recursion in the case that type === "newListener"! Before
            // adding it to the listeners, first emit "newListener".
            if (events.newListener !== undefined) { // 触发了newListener事件后
              target.emit('newListener', type,
                          listener.listener ? listener.listener : listener);
          
              // Re-assign `events` because a newListener handler could have caused the
              // this._events to be assigned to a new object
              events = target._events;
            }
            // 获取传入的事件
            existing = events[type];
          }

          if (existing === undefined) { // 事件不存在
            // Optimize the case of one listener. Don't need the extra array object.
            events[type] = listener; // 不存在，就直接使用listener
            ++target._eventsCount;
          } else {
            if (typeof existing === 'function') {
              // Adding the second element, need to change to array.
              existing = events[type] =
                prepend ? [listener, existing] : [existing, listener];
              // If we've already got an array, just append.
            } else if (prepend) {
              existing.unshift(listener); // 将listener添加到数组的最前面
            } else {
              existing.push(listener); // 将listener添加到数组的末尾
            }
          
            // Check for listener leak
            m = _getMaxListeners(target);
            // 判断监听数是否超过最大的监听数目。如果超过，就报错并且标记已警告
            if (m > 0 && existing.length > m && !existing.warned) {
              existing.warned = true;
              // No error code for this since it is a Warning
              // eslint-disable-next-line no-restricted-syntax
              const w = new Error('Possible EventEmitter memory leak detected. ' +
                                  `${existing.length} ${String(type)} listeners ` +
                                  `added to ${inspect(target, { depth: -1 })}. Use ` +
                                  'emitter.setMaxListeners() to increase limit');
              w.name = 'MaxListenersExceededWarning';
              w.emitter = target;
              w.type = type;
              w.count = existing.length;
              process.emitWarning(w);
           }
          }

          return target;
        }

        function checkListener(listener) {
          if (typeof listener !== 'function') {
            throw new ERR_INVALID_ARG_TYPE('listener', 'Function', listener);
          }
        }
        
        
        EventEmitter.prototype.addListener = function addListener(type, listener) {
          return _addListener(this, type, listener, false);
        };
        
        EventEmitter.prototype.on = EventEmitter.prototype.addListener;
        
        EventEmitter.prototype.prependListener = function prependListener(type, listener) {
          return _addListener(this, type, listener, true);
        };
        ```
      - 例子：
        ```
          const EventEmitter = require('events').EventEmitter;
          const emitter = new EventEmitter();

          const OpenTheDoor = (key) => {
            console.log('Got the key: ' + key);
          };
          const Move = (name) => {
            console.log(name + ' Moveed yet!');
          };

          emitter.on('Office', Move);
          emitter.emit('Office', 'Gold'); // 输出Gold Moveed yet!

          emitter.addListener('Open', OpenTheDoor);
          emitter.emit('Open', 'Im Frank,please the door now'); // 输出 Got the key: Im Frank,please the door now

          emitter.once('newListener', (event, listener) => {
           if (event === 'HelloWorld') {
            emitter.on('HelloWorld', () => {
              console.log('HelloWorld');
            });
           }
          });
          emitter.on('HelloWorld', () => {
            console.log('HelloWorld');
          });
          emitter.emit('HelloWorld');
        ```
    - once
        ```
          function _onceWrap(target, type, listener) {
            const state = { fired: false, wrapFn: undefined, target, type, listener };
            // 将state绑定到onceWrapper中
            const wrapped = onceWrapper.bind(state);
            wrapped.listener = listener;
            state.wrapFn = wrapped;
            return wrapped;
          }

          function onceWrapper() {
            if (!this.fired) {
              this.target.removeListener(this.type, this.wrapFn);
              this.fired = true;
              if (arguments.length === 0)
                return this.listener.call(this.target);
              return this.listener.apply(this.target, arguments);
            }
          }

          function on(emitter, event) {
            const unconsumedEvents = [];
            const unconsumedPromises = [];
            let error = null;
            let finished = false;

            const iterator = ObjectSetPrototypeOf({
              next() {
                // First, we consume all unread events
                const value = unconsumedEvents.shift();
                if (value) {
                  return PromiseResolve(createIterResult(value, false));
                }
              
                // Then we error, if an error happened
                // This happens one time if at all, because after 'error'
                // we stop listening
                if (error) {
                  const p = PromiseReject(error);
                  // Only the first element errors
                  error = null;
                  return p;
                }
              
                // If the iterator is finished, resolve to done
                if (finished) {
                  return PromiseResolve(createIterResult(undefined, true));
                }
                
                // Wait until an event happens
                return new Promise(function(resolve, reject) {
                  unconsumedPromises.push({ resolve, reject });
                });
              },

              return() {
                emitter.removeListener(event, eventHandler);
                emitter.removeListener('error', errorHandler);
                finished = true;

                for (const promise of unconsumedPromises) {
                  promise.resolve(createIterResult(undefined, true));
                }

                return PromiseResolve(createIterResult(undefined, true));
              },

              throw(err) {
                if (!err || !(err instanceof Error)) {
                  throw new ERR_INVALID_ARG_TYPE('EventEmitter.AsyncIterator',
                                                 'Error', err);
                }
                error = err;
                emitter.removeListener(event, eventHandler);
                emitter.removeListener('error', errorHandler);
              },
            
              [SymbolAsyncIterator]() {
                return this;
              }
            }, AsyncIteratorPrototype);

            emitter.on(event, eventHandler);
            emitter.on('error', errorHandler);

            return iterator;

            function eventHandler(...args) {
              const promise = unconsumedPromises.shift();
              if (promise) {
                promise.resolve(createIterResult(args, false));
              } else {
                unconsumedEvents.push(args);
              }
            }

            // 针对error事件，移除响应的事件并返回对应的错误原因
            function errorHandler(err) {
              finished = true;
            
              const toError = unconsumedPromises.shift();
            
              if (toError) {
                toError.reject(err);
              } else {
                // The next time we call next()
                error = err;
              }
            
              iterator.return();
            }
            }

          EventEmitter.prototype.once = function once(type, listener) {
            // 首先检查listener是不是一个回调函数。不是的话，就报错
            checkListener(listener);
          
            this.on(type, _onceWrap(this, type, listener));
            return this;
          };
          
          function createIterResult(value, done) {
            return { value, done };
          }
        ```
  - 删除一个回调方法: removeListener
  ```
  // Emits a 'removeListener' event if and only if the listener was removed.
  EventEmitter.prototype.removeListener =
    function removeListener(type, listener) {
      let originalListener;

      checkListener(listener);

      // 当前作用域没有任何事件时，直接退出
      const events = this._events;
      if (events === undefined)
        return this;

      const list = events[type];
      // 没有type类型的事件，直接退出
      if (list === undefined)
        return this;

      // 存在listener事件，删除type类型的事件并更新事件的总数
      if (list === listener || list.listener === listener) {
        if (--this._eventsCount === 0)
          this._events = ObjectCreate(null);
        else {
          delete events[type];
          // 存在删除事件，直接删除该事件
          if (events.removeListener)
            this.emit('removeListener', type, list.listener || listener);
        }
      } else if (typeof list !== 'function') {
        // 对应事件不是一个回调函数
        let position = -1;

        // 遍历获取当前事件所在的位置并保存当前事件
        for (let i = list.length - 1; i >= 0; i--) {
          if (list[i] === listener || list[i].listener === listener) {
            originalListener = list[i].listener;
            position = i;
            break;
          }
        }

        if (position < 0)
          return this;

        if (position === 0)
          list.shift();
        else {
          if (spliceOne === undefined)
            spliceOne = require('internal/util').spliceOne;
          spliceOne(list, position);
        }

        if (list.length === 1)
          events[type] = list[0];

        // 删除当前事件
        if (events.removeListener !== undefined)
          this.emit('removeListener', type, originalListener || listener);
      }

      return this;
    };
  ```
  - 删除多个回方法: removeAllListeners
  ```
    EventEmitter.prototype.removeAllListeners =
    function removeAllListeners(type) {
      const events = this._events;
      if (events === undefined)
        return this;

      // Not listening for removeListener, no need to emit
      if (events.removeListener === undefined) {
        // 如果没有传key去删除事件
        if (arguments.length === 0) {
          this._events = ObjectCreate(null);
          this._eventsCount = 0;
        } else if (events[type] !== undefined) {
          if (--this._eventsCount === 0)
            this._events = ObjectCreate(null);
          else
            // 删除当前事件
            delete events[type];
        }
        return this;
      }

      // Emit removeListener for all listeners on all events
      if (arguments.length === 0) {
        for (const key of ObjectKeys(events)) {
          if (key === 'removeListener') continue; // 如果是removeListener，不做处理，不删除
          this.removeAllListeners(key);
        }
        this.removeAllListeners('removeListener');
        this._events = ObjectCreate(null);
        this._eventsCount = 0;
        return this;
      }

      // 如果指定某个事件，就删除该事件下所有的回调函数
      const listeners = events[type];

      if (typeof listeners === 'function') {
        // 递归调用
        this.removeListener(type, listeners);
      } else if (listeners !== undefined) {
        // LIFO order
        // 如果有多个相同的事件，遍历依次删除
        for (let i = listeners.length - 1; i >= 0; i--) {
          this.removeListener(type, listeners[i]);
        }
      }

      return this;
    };
  ```
  - 事件触发: emit
  ```
    const kErrorMonitor = Symbol('events.errorMonitor');
    EventEmitter.prototype.emit = function emit(type, ...args) {
      let doError = (type === 'error');

      const events = this._events;
      if (events !== undefined) {
        if (doError && events[kErrorMonitor] !== undefined)
          this.emit(kErrorMonitor, ...args);
        doError = (doError && events.error === undefined);
      } else if (!doError)
        // 什么事件都没有，即使error事件也没有。直接退出。
        return false;

      // If there is no 'error' event listener then throw.
      // 当前事件是一个Error事件
      if (doError) {
        let er;
        if (args.length > 0)
          er = args[0];
        if (er instanceof Error) {
          try {
            const capture = {};
            // eslint-disable-next-line no-restricted-syntax
            Error.captureStackTrace(capture, EventEmitter.prototype.emit);
            ObjectDefineProperty(er, kEnhanceStackBeforeInspector, {
              value: enhanceStackTrace.bind(this, er, capture),
              configurable: true
            });
          } catch {}
      
          // Note: The comments on the `throw` lines are intentional, they show
          // up in Node's output if this results in an unhandled exception.
          throw er; // Unhandled 'error' event
        }      
        let stringifiedEr;
        const { inspect } = require('internal/util/inspect');
        try {
          stringifiedEr = inspect(er);
        } catch {
          stringifiedEr = er;
        }
        
        // At least give some kind of context to the user
        const err = new ERR_UNHANDLED_ERROR(stringifiedEr);
        err.context = er;
        throw err; // Unhandled 'error' event
      }

      // 某个指定的事件
      const handler = events[type];
      
      if (handler === undefined)
        return false;
      
      if (typeof handler === 'function') {
        const result = ReflectApply(handler, this, args);
      
        // We check if result is undefined first because that
        // is the most common case so we do not pay any perf
        // penalty
        if (result !== undefined && result !== null) {
          addCatch(this, result, type, args);
        }
      } else {
        const len = handler.length;
        // 赋值定值事件下的所有回调函数
        const listeners = arrayClone(handler, len);
        for (let i = 0; i < len; ++i) {
          const result = ReflectApply(listeners[i], this, args);
        
          // We check if result is undefined first because that
          // is the most common case so we do not pay any perf
          // penalty.
          // This code is duplicated because extracting it away
          // would make it non-inlineable.
          if (result !== undefined && result !== null) {
            addCatch(this, result, type, args);
          }
        }
      }

      return true;
   };

  function addCatch(that, promise, type, args) {
    if (!that[kCapture]) {
      return;
    }
  
    // Handle Promises/A+ spec, then could be a getter
    // that throws on second use.
    try {
      const then = promise.then;
  
      if (typeof then === 'function') {
        then.call(promise, undefined, function(err) {
          // The callback is called with nextTick to avoid a follow-up
          // rejection from this promise.
          process.nextTick(emitUnhandledRejectionOrErr, that, err, type, args);
        });
      }
    } catch (err) {
      that.emit('error', err);
    }
  }
  
  function arrayClone(arr, n) {
    const copy = new Array(n);
    for (let i = 0; i < n; ++i)
      copy[i] = arr[i];
    return copy;
  }
  ```

- [GitHub](https://github.com/sunfeng90/node-core-study)
- [参考](https://github.com/xiaomuzhu/ElemeFE-node-interview/blob/master/%E5%BC%82%E6%AD%A5/Event.md)