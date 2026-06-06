# Riverpod Provider 销毁后安全更新 State 方案

## 1. 核心痛点

页面级状态控制器中经常会有异步逻辑：

```dart
Future<void> load() async {
  state = state.copyWith(loading: true);

  final result = await someAsyncTask();

  state = state.copyWith(
    loading: false,
    data: result,
  );
}
```

这段代码的问题不在 `copyWith()`，而在异步任务完成后继续执行：

```dart
state = ...
ref.read(...)
ref.watch(...)
```

如果页面已经退出、对应的 autoDispose provider 已经销毁，异步任务才完成，那么
`await` 后继续更新 `state` 或继续使用已经失效的 `ref`，可能触发 Riverpod 的
disposed/ref mounted 相关错误。

典型流程：

```text
页面退出
  -> 页面不再监听 provider
  -> autoDispose provider 被销毁
  -> 异步任务稍后完成
  -> await 后继续执行
  -> 继续 state = ...
  -> 或继续 ref.read(...)
  -> 可能触发 Riverpod disposed/ref mounted 相关错误
```

最直接的规避方式是在每个 `await` 后手写判断：

```dart
if (!ref.mounted) {
  return;
}
```

但这种方式不适合作为长期规范：

- 判断散落在每个异步方法中。
- 容易遗漏。
- 业务代码被生命周期细节污染。
- code review 很难确认每个异步分支都处理正确。

本方案只解决一个问题：**provider 销毁后，异步逻辑完成时不再更新 state，也尽量避免继续使用失效的 ref。**

## 2. 设计目标

业务侧目标写法：

```dart
Future<void> load() async {
  await guard(
    () async {
      emit(state.copyWith(loading: true, error: null));

      final result = await someAsyncTask();

      emit(state.copyWith(
        loading: false,
        data: result,
        error: null,
      ));
    },
    onError: (error, stackTrace) {
      emit(state.copyWith(
        loading: false,
        error: error,
      ));
    },
  );
}
```

业务只需要遵守两条规则：

- 更新 state 统一使用 `emit(...)` 或 `update(...)`。
- 异步流程统一使用 `guard(() async { ... })` 包裹。

## 3. 不做什么

本方案不处理中断异步任务本身。

本方案不引入额外生命周期对象，不处理底层任务调度，也不改变业务数据层接口。

本方案只做：

```text
异步任务完成后，如果 provider 已经销毁，则不再更新 state。
```

## 4. 设计思路

生命周期判断应该放在 Notifier / Controller 层，而不是放进 State。

正确职责划分：

```text
State
  -> 只描述数据
  -> 只提供 copyWith
  -> 不依赖 ref / BuildContext / provider 生命周期

Notifier / Controller
  -> 管状态更新
  -> 管 ref.mounted
  -> 管异步完成后的安全处理
```

不要把 mounted 判断放进 `copyWith()`：

```dart
// 不推荐
state.copyWith(isMounted: ref.mounted);
```

原因是 `copyWith()` 只是构造一个新的数据对象，真正有风险的是后面的赋值：

```dart
state = nextState;
```

所以应该封装赋值动作：

```dart
emit(state.copyWith(data: result));
```

而不是污染数据模型。

核心设计：

- `emit(...)` 安全替代 `state = ...`。
- `update(...)` 安全基于当前 state 计算并更新。
- `guard(...)` 包裹异步流程，统一处理完成后 provider 已销毁的情况。
- `isAlive` 提供必要时的显式判断。

## 5. 总体架构

轻量方案只需要两个组件：

```text
AppStateEmitter
  -> 安全更新 state

AppSafeNotifier
  -> 业务 mixin，暴露 emit / update / guard / isAlive
```

业务 Controller 只使用 `AppSafeNotifier`：

```dart
class ExampleController extends _$ExampleController
    with AppSafeNotifier<ExampleState, ExampleState> {
  // ...
}
```

## 6. 核心 API

### 6.1 emit

安全替代直接赋值：

```dart
emit(state.copyWith(data: result));
```

内部逻辑：

```text
如果 provider 仍然 mounted，执行 state = value。
如果 provider 已经销毁，直接 return。
```

### 6.2 update

安全基于当前 state 更新：

```dart
update((current) {
  return current.copyWith(count: current.count + 1);
});
```

内部逻辑：

```text
如果 provider 仍然 mounted，读取当前 state 并更新。
如果 provider 已经销毁，不读取 state，直接 return。
```

### 6.3 guard

包裹异步流程：

```dart
await guard(() async {
  final result = await someAsyncTask();
  emit(state.copyWith(data: result));
});
```

内部逻辑：

```text
开始前检查 mounted。
异步完成后再次检查 mounted。
发生错误时，如果仍 mounted，则交给 onError。
如果已经销毁，则忽略后续处理。
```

### 6.4 isAlive

在少数必须手动判断的场景使用：

```dart
if (!isAlive) {
  return;
}
```

典型场景是 `await` 后确实需要继续使用 `ref`。

## 7. 实现代码骨架

### 7.1 AppStateEmitter

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';

final class AppStateEmitter<StateT, ValueT> {
  const AppStateEmitter({
    required AnyNotifier<StateT, ValueT> notifier,
    required bool Function() isMounted,
  })  : _notifier = notifier,
        _isMounted = isMounted;

  final AnyNotifier<StateT, ValueT> _notifier;
  final bool Function() _isMounted;

  void emit(StateT value) {
    if (!_isMounted()) {
      return;
    }

    _notifier.state = value;
  }

  void update(StateT Function(StateT current) transform) {
    if (!_isMounted()) {
      return;
    }

    emit(transform(_notifier.state));
  }
}
```

### 7.2 AppSafeNotifier

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';

mixin AppSafeNotifier<StateT, ValueT>
    on AnyNotifier<StateT, ValueT> {
  AppStateEmitter<StateT, ValueT>? _emitter;

  AppStateEmitter<StateT, ValueT> get _stateEmitter {
    return _emitter ??= AppStateEmitter<StateT, ValueT>(
      notifier: this,
      isMounted: () => ref.mounted,
    );
  }

  bool get isAlive {
    return ref.mounted;
  }

  void emit(StateT value) {
    _stateEmitter.emit(value);
  }

  void update(StateT Function(StateT current) transform) {
    _stateEmitter.update(transform);
  }

  Future<T?> guard<T>(
    Future<T> Function() action, {
    void Function(Object error, StackTrace stackTrace)? onError,
  }) async {
    if (!ref.mounted) {
      return null;
    }

    try {
      final result = await action();

      if (!ref.mounted) {
        return null;
      }

      return result;
    } catch (error, stackTrace) {
      if (!ref.mounted) {
        return null;
      }

      onError?.call(error, stackTrace);
      return null;
    }
  }
}
```

## 8. 正常异步流程

业务代码：

```dart
await guard(() async {
  emit(state.copyWith(loading: true));

  final result = await someAsyncTask();

  emit(state.copyWith(
    loading: false,
    data: result,
  ));
});
```

执行流程：

```text
Controller 方法开始
  -> guard()
  -> 检查 ref.mounted == true
  -> action 开始执行
  -> emit(loading: true)
  -> AppStateEmitter.emit()
  -> provider 仍 mounted
  -> 执行 state = newState
  -> await someAsyncTask()
  -> 异步任务完成
  -> emit(data)
  -> AppStateEmitter.emit()
  -> provider 仍 mounted
  -> 执行 state = newState
  -> action 完成
  -> guard 再检查 ref.mounted == true
  -> 返回 result
```

## 9. Provider Dispose 后异步完成流程

业务代码仍然是：

```dart
await guard(() async {
  final result = await someAsyncTask();

  emit(state.copyWith(data: result));
});
```

销毁后流程：

```text
Controller 方法开始
  -> guard()
  -> 检查 ref.mounted == true
  -> action 开始执行
  -> await someAsyncTask()
  -> 页面退出
  -> provider dispose
  -> 异步任务完成
  -> action 继续执行
  -> emit(data)
  -> AppStateEmitter.emit()
  -> ref.mounted == false
  -> 直接 return
  -> 不执行 state = newState
  -> action 完成
  -> guard 最后检查 ref.mounted == false
  -> 返回 null
```

关键点：

```text
销毁后继续调用 emit 是安全的。
emit 内部会判断 mounted。
provider 已销毁时不会执行 state = ...
```

## 10. 错误处理流程

业务代码：

```dart
await guard(
  () async {
    final result = await someAsyncTask();
    emit(state.copyWith(data: result));
  },
  onError: (error, stackTrace) {
    emit(state.copyWith(error: error));
  },
);
```

异步任务失败且 provider 仍然 mounted：

```text
someAsyncTask 抛错
  -> guard catch error
  -> ref.mounted == true
  -> 调用 onError(error, stackTrace)
  -> onError 内使用 emit
  -> emit 判断 mounted
  -> 更新 error state
```

异步任务失败但 provider 已销毁：

```text
someAsyncTask 抛错
  -> guard catch error
  -> ref.mounted == false
  -> 直接 return null
  -> 不调用 onError
  -> 不更新 state
```

注意：`onError` 内也必须使用 `emit(...)` 或 `update(...)`，不要直接 `state = ...`。

## 11. 业务使用示例

### 11.1 State

```dart
class ExampleState {
  const ExampleState({
    this.loading = false,
    this.data,
    this.error,
  });

  final bool loading;
  final Object? data;
  final Object? error;

  ExampleState copyWith({
    bool? loading,
    Object? data,
    Object? error,
  }) {
    return ExampleState(
      loading: loading ?? this.loading,
      data: data ?? this.data,
      error: error,
    );
  }
}
```

### 11.2 Controller

```dart
@riverpod
class ExampleController extends _$ExampleController
    with AppSafeNotifier<ExampleState, ExampleState> {
  @override
  ExampleState build() {
    return const ExampleState();
  }

  Future<void> load() async {
    await guard(
      () async {
        emit(state.copyWith(loading: true, error: null));

        final result = await someAsyncTask();

        emit(state.copyWith(
          loading: false,
          data: result,
          error: null,
        ));
      },
      onError: (error, stackTrace) {
        emit(state.copyWith(
          loading: false,
          error: error,
        ));
      },
    );
  }
}
```

### 11.3 返回值写法

`guard` 返回 `Future<T?>`，所以也可以这样写：

```dart
Future<void> load() async {
  final result = await guard(() {
    return someAsyncTask();
  });

  if (result == null) {
    return;
  }

  emit(state.copyWith(data: result));
}
```

注意：

- `guard` 返回 `null` 可能代表 provider 已销毁。
- `guard` 返回 `null` 也可能代表异步失败且错误已被处理。
- 如果业务结果本身允许 `null`，不建议用返回值区分状态，推荐在 action 内完成 `emit`。

## 12. await 后继续使用 ref 的风险

`guard` 可以保护 `emit(...)`，也能在 action 开始前、完成后、catch 时检查
`ref.mounted`。

但如果 action 内部在 `await` 之后直接使用 `ref`，仍然需要谨慎：

```dart
await guard(() async {
  final result = await someAsyncTask();

  final value = ref.read(otherProvider); // 有风险

  emit(state.copyWith(data: result, extra: value));
});
```

推荐做法是：在 `await` 前读取依赖。

```dart
await guard(() async {
  final service = ref.read(serviceProvider);

  final result = await service.someAsyncTask();

  emit(state.copyWith(data: result));
});
```

如果确实必须在 `await` 后使用 `ref`，先判断 `isAlive`：

```dart
await guard(() async {
  final result = await someAsyncTask();

  if (!isAlive) {
    return;
  }

  final value = ref.read(otherProvider);

  emit(state.copyWith(data: result, extra: value));
});
```

规则：

```text
await 后更新 state 用 emit。
await 后尽量不要直接使用 ref。
确实需要使用 ref 时，先判断 isAlive。
```

## 13. 易错点和注意事项

### 13.1 继续直接写 state =

不要这样：

```dart
state = state.copyWith(data: result);
```

统一改为：

```dart
emit(state.copyWith(data: result));
```

### 13.2 update 中不要做异步

不要这样：

```dart
update((current) async {
  final result = await someAsyncTask();
  return current.copyWith(data: result);
});
```

`update` 只处理同步 state 转换。异步逻辑放进 `guard`：

```dart
await guard(() async {
  final result = await someAsyncTask();
  update((current) => current.copyWith(data: result));
});
```

### 13.3 onError 内也要用 emit

不要这样：

```dart
onError: (error, stackTrace) {
  state = state.copyWith(error: error);
}
```

应该这样：

```dart
onError: (error, stackTrace) {
  emit(state.copyWith(error: error));
}
```

### 13.4 State 不要感知生命周期

State 不应持有：

```text
Ref
BuildContext
ProviderSubscription
```

State 应保持为纯数据对象，方便测试、复制和序列化。

### 13.5 页面级 provider 应使用 autoDispose

如果 provider 是全局常驻的，它不会随页面退出销毁，`ref.mounted` 仍然可能是 true。

本方案解决的是 provider 已经销毁后的安全更新问题。页面级状态应默认使用
autoDispose。

### 13.6 guard 不会让异步任务提前停止

`guard` 不会中断已经开始的异步任务。它只保证异步完成后，如果 provider 已经销毁，
就不再更新 state 或处理错误状态。

## 14. 测试建议

### 14.1 emit

- provider mounted 时，`emit` 能更新 state。
- provider dispose 后，`emit` 不抛错。
- provider dispose 后，`emit` 不执行 `state = ...`。

### 14.2 update

- provider mounted 时，`update` 能基于当前 state 更新。
- provider dispose 后，`update` 不读取 state。
- provider dispose 后，`update` 不抛错。

### 14.3 guard

- action 正常完成时返回结果。
- action 抛错且 provider mounted 时调用 `onError`。
- action 抛错但 provider disposed 时忽略错误。
- action 中 `await` 后 provider dispose，再调用 `emit` 不抛错。
- action 中 `await` 后 provider dispose，`guard` 最终返回 `null`。

### 14.4 业务约束

- Controller 异步方法使用 `guard`。
- Controller 状态更新使用 `emit` / `update`。
- `onError` 内也使用 `emit` / `update`。
- State / `copyWith` 不依赖 Riverpod。

## 15. 建议文件落点

```text
lib/core/riverpod/app_state_emitter.dart
lib/core/riverpod/app_safe_notifier.dart
lib/core/riverpod/app_riverpod.dart
```

统一导出：

```dart
export 'app_safe_notifier.dart';
export 'app_state_emitter.dart';
```

## 16. 最终原则

- 本方案只做安全 state 更新。
- 异步流程用 `guard`。
- 状态更新用 `emit` / `update`。
- `update` 只做同步 state 转换。
- State / `copyWith` 保持纯数据。
- `await` 后尽量不要直接使用 `ref`。
- 必须在 `await` 后使用 `ref` 时，先判断 `isAlive`。
- provider 销毁后，`emit` / `update` 必须不抛错、不更新 state。
