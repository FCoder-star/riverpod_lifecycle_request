# Riverpod 销毁后安全更新 State 方案

## 1. 核心痛点

页面级异步请求常见代码如下：

```dart
Future<void> load() async {
  state = state.copyWith(loading: true);

  final data = await repository.fetch();

  state = state.copyWith(
    loading: false,
    data: data,
  );
}
```

如果页面已经销毁、对应 provider 已经 dispose，网络请求才返回，那么 `await`
之后继续做下面这些事就可能出问题：

```dart
state = state.copyWith(data: data);
ref.read(otherProvider);
ref.watch(otherProvider);
```

Riverpod 3 中，provider dispose 后，除了 `ref.mounted` 之外，继续使用 `ref`
或 notifier 方法可能抛错。典型风险是：

```text
页面退出
  -> provider dispose
  -> 网络请求返回
  -> 继续更新 state / 使用 ref
  -> 触发 disposed/ref mounted 相关错误
```

最直接的规避方式是在每个 `await` 后写：

```dart
if (!ref.mounted) {
  return;
}
```

但这种写法不适合作为长期规范：

- 判断散落在每个异步方法中。
- 容易遗漏。
- 业务代码被生命周期细节污染。
- 写法重复，不利于 code review。

本方案只解决一个问题：**provider dispose 后，请求返回时不再更新 state，也不再继续执行依赖 ref 的逻辑。**

本方案不做：

- 不取消网络请求。
- 不引入 `CancelToken`。
- 不改 Repository / AppApi 签名。
- 不使用 Dart Zone。
- 不加 Dio interceptor。

## 2. 设计目标

业务侧目标写法：

```dart
Future<void> load() async {
  await guard(
    () async {
      emit(state.copyWith(loading: true, error: null));

      final data = await ref.read(appRepositoryProvider).health();

      emit(state.copyWith(
        loading: false,
        data: data,
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

业务只需要记住两条规则：

- 更新 state 用 `emit(...)` 或 `update(...)`。
- 异步方法用 `guard(() async { ... })` 包裹。

## 3. 设计思想

生命周期判断应该放在 Notifier / Controller 层，而不是放进 State。

正确职责划分：

```text
State
  -> 只描述数据
  -> 提供 copyWith
  -> 不依赖 Riverpod

Repository
  -> 只负责业务数据
  -> 不感知页面生命周期

Notifier / Controller
  -> 管状态更新
  -> 管 provider 是否仍 mounted
  -> 管异步请求返回后的安全处理
```

不要把判断放进 `copyWith()`：

```dart
// 不推荐
state.copyWith(ref: ref);
```

原因是报错点不在 `copyWith()`，而在：

```dart
state = nextState;
```

所以应该封装赋值动作：

```dart
emit(state.copyWith(data: data));
```

而不是封装数据构造。

## 4. 总体架构

轻量方案只需要两个 mixin/工具：

```text
AppStateEmitter
  -> 安全更新 state

AppSafeAsyncNotifier
  -> 业务 mixin，暴露 emit / update / guard
```

如果不想拆文件，也可以第一版只做一个 mixin：

```text
AppSafeNotifier
  -> emit / update / guard
```

考虑到后续可维护性，推荐拆成 `AppStateEmitter` + `AppSafeNotifier`。

## 5. 核心 API

### 5.1 AppSafeNotifier

业务 Controller 使用的 mixin：

```dart
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

### 5.2 AppStateEmitter

```dart
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

## 6. 完整执行流程

业务代码：

```dart
await guard(() async {
  emit(state.copyWith(loading: true));

  final data = await repository.fetch();

  emit(state.copyWith(
    loading: false,
    data: data,
  ));
});
```

正常流程：

```text
Controller.load()
  -> guard()
  -> 检查 ref.mounted == true
  -> action 开始执行
  -> emit(loading: true)
  -> AppStateEmitter.emit()
  -> ref.mounted == true，执行 state = newState
  -> await repository.fetch()
  -> 请求返回
  -> emit(data)
  -> AppStateEmitter.emit()
  -> ref.mounted == true，执行 state = newState
  -> action 完成
  -> guard 再检查 ref.mounted
  -> 返回 result
```

页面销毁后请求才返回的流程：

```text
Controller.load()
  -> guard()
  -> action 开始执行
  -> await repository.fetch()
  -> 页面退出
  -> provider dispose
  -> 请求返回
  -> emit(data)
  -> AppStateEmitter.emit()
  -> ref.mounted == false
  -> 直接 return，不执行 state = newState
  -> action 完成
  -> guard 检查 ref.mounted == false
  -> 返回 null
```

请求失败且 provider 仍然 mounted：

```text
await repository.fetch()
  -> throw error
  -> guard catch
  -> ref.mounted == true
  -> 调用 onError(error, stackTrace)
```

请求失败但 provider 已 dispose：

```text
await repository.fetch()
  -> 页面已退出
  -> throw error
  -> guard catch
  -> ref.mounted == false
  -> 忽略错误
  -> 返回 null
```

## 7. 业务使用示例

### 7.1 State

```dart
class HealthState {
  const HealthState({
    this.loading = false,
    this.data,
    this.error,
  });

  final bool loading;
  final Object? data;
  final Object? error;

  HealthState copyWith({
    bool? loading,
    Object? data,
    Object? error,
  }) {
    return HealthState(
      loading: loading ?? this.loading,
      data: data ?? this.data,
      error: error,
    );
  }
}
```

### 7.2 Controller

```dart
@riverpod
class HealthController extends _$HealthController
    with AppSafeNotifier<HealthState, HealthState> {
  @override
  HealthState build() {
    return const HealthState();
  }

  Future<void> load() async {
    await guard(
      () async {
        emit(state.copyWith(loading: true, error: null));

        final data = await ref.read(appRepositoryProvider).health();

        emit(state.copyWith(
          loading: false,
          data: data,
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

### 7.3 页面

```dart
class HealthPage extends ConsumerWidget {
  const HealthPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(healthControllerProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Health')),
      body: Center(
        child: state.loading
            ? const CircularProgressIndicator()
            : Text(state.data?.toString() ?? 'No data'),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () {
          ref.read(healthControllerProvider.notifier).load();
        },
        child: const Icon(Icons.refresh),
      ),
    );
  }
}
```

## 8. 返回值写法

`guard` 返回 `Future<T?>`，所以也可以这样写：

```dart
Future<void> load() async {
  final data = await guard(() {
    return ref.read(appRepositoryProvider).health();
  });

  if (data == null) {
    return;
  }

  emit(state.copyWith(data: data));
}
```

注意：

- `guard` 返回 `null` 可能代表 provider 已 dispose。
- `guard` 返回 `null` 也可能代表请求失败且错误已被 `onError` 处理。
- 如果业务数据本身允许 `null`，应避免用返回值区分状态，推荐在 action 内完成 `emit`。

## 9. 为什么 guard 不能阻止 action 内所有 ref 使用

`guard` 可以在 action 开始前、完成后、catch 时检查 `ref.mounted`。

但如果 action 内部在 `await` 之后直接使用 `ref`，仍然可能有风险：

```dart
await guard(() async {
  final data = await repository.fetch();

  final other = ref.read(otherProvider); // 有风险

  emit(state.copyWith(data: data, other: other));
});
```

更稳妥的写法是在 `await` 前提前读取依赖：

```dart
await guard(() async {
  final otherRepository = ref.read(otherRepositoryProvider);

  final data = await repository.fetch();
  final other = await otherRepository.fetchOther();

  emit(state.copyWith(data: data, other: other));
});
```

如果确实需要在 `await` 后使用 `ref`，应显式判断：

```dart
await guard(() async {
  final data = await repository.fetch();

  if (!isAlive) {
    return;
  }

  final other = ref.read(otherProvider);

  emit(state.copyWith(data: data, other: other));
});
```

因此本轻量方案的推荐规则是：

```text
await 之后更新 state 用 emit。
await 之后尽量不要再直接使用 ref。
确实需要使用 ref 时，先判断 isAlive。
```

## 10. 和自动取消请求方案的区别

轻量方案：

```text
请求继续跑
请求回来后不更新已 dispose provider 的 state
不取消 Dio 请求
不改网络层
实现简单
```

生命周期绑定取消方案：

```text
provider dispose 时取消 Dio 请求
需要 CancelToken / Zone / Dio interceptor
实现复杂
适合强生命周期绑定诉求
```

本方案适合当前目标：

```text
只处理页面销毁后请求返回，避免继续更新 state 或使用 ref 造成错误。
```

## 11. 容易出问题的地方

### 11.1 继续直接写 state =

不要这样：

```dart
state = state.copyWith(data: data);
```

统一改为：

```dart
emit(state.copyWith(data: data));
```

### 11.2 await 后继续直接使用 ref

有风险：

```dart
final data = await repository.fetch();
final value = ref.read(otherProvider);
```

推荐提前读取依赖，或在使用前判断：

```dart
if (!isAlive) {
  return;
}
```

### 11.3 provider 不是 autoDispose

如果 provider 是 keepAlive 或全局 provider，它不会随页面销毁而 dispose。

这时 `ref.mounted` 仍然可能是 true，请求返回后仍会更新 state。

页面级状态 provider 应默认使用 autoDispose。

### 11.4 guard 不会取消网络请求

本方案不会中断请求。页面销毁后，请求仍会继续到完成，只是完成后不会更新已销毁 provider。

如果请求成本很高，例如上传、下载、大文件、长轮询，应使用更完整的 CancelToken 生命周期绑定方案。

### 11.5 不要把生命周期放进 State

不要让 State 持有：

```dart
Ref
BuildContext
CancelToken
ProviderSubscription
```

State 应保持可测试、可复制、可序列化的纯数据对象。

### 11.6 onError 内也要用 emit

错误处理里不要直接写：

```dart
state = state.copyWith(error: error);
```

应该写：

```dart
emit(state.copyWith(error: error));
```

## 12. 测试建议

### 12.1 emit/update

- mounted 时 `emit` 能更新 state。
- provider dispose 后 `emit` 不抛错。
- mounted 时 `update` 能基于当前 state 更新。
- provider dispose 后 `update` 不读取 state、不抛错。

### 12.2 guard

- action 正常完成时返回结果。
- action 抛错且 provider mounted 时调用 `onError`。
- action 抛错但 provider disposed 时忽略错误。
- action await 后 provider dispose，再调用 `emit` 不抛错。

### 12.3 业务约束

- Controller 异步方法使用 `guard`。
- Controller 状态更新使用 `emit/update`。
- Repository / AppApi 不需要改动。

## 13. 建议文件落点

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

## 14. 最终原则

- 轻量方案只防止 dispose 后更新 state/ref，不取消网络请求。
- 页面级异步方法用 `guard`。
- 状态更新用 `emit` / `update`。
- State / `copyWith` 保持纯数据。
- Repository / AppApi 保持无感。
- `await` 后尽量不要直接使用 `ref`。
- 确实需要在 `await` 后使用 `ref` 时，先判断 `isAlive`。
- 高成本请求如需真正取消，再使用 CancelToken 生命周期绑定方案。
