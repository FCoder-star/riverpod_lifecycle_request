# Riverpod 生命周期绑定网络请求方案

## 1. 核心痛点

页面级网络请求常见写法如下：

```dart
final data = await repository.fetch();
state = state.copyWith(data: data);
```

当页面已经销毁、对应 provider 已经 dispose 后，网络请求才返回，此时继续更新
`state` 或继续使用 `ref`，可能触发 Riverpod 的 disposed/ref mounted 相关错误。

直接在业务中到处写判断虽然能规避问题，但不适合作为长期规范：

```dart
if (!ref.mounted) {
  return;
}

state = state.copyWith(data: data);
```

这种写法有几个问题：

- 判断散落在每个异步方法中，容易遗漏。
- 业务代码被生命周期细节污染。
- 只能避免销毁后更新 state，不能取消仍在进行的 Dio 请求。
- 如果每个 Repository 或 AppApi 方法都显式传 `CancelToken`，会污染业务接口。

本方案要解决的核心问题是：**像 Android `viewModelScope` / `lifecycleScope` 一样，让页面级请求自动绑定当前状态 provider 的生命周期。**

目标效果：

- 页面级 provider dispose 时，自动取消当前 provider 发起的未完成 Dio 请求。
- Repository 和 AppApi 不暴露 `CancelToken`。
- 业务只使用 `request(...)` 发起生命周期绑定请求。
- 业务只使用 `emit(...)` / `update(...)` 安全更新 state。
- State 和 `copyWith` 仍然保持纯数据，不感知 Riverpod 生命周期。

## 2. 设计目标

业务 Controller 目标写法：

```dart
Future<void> load() async {
  await request(
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

搜索、刷新、重复点击等场景目标写法：

```dart
Future<void> search(String keyword) async {
  await request(
    key: 'search',
    cancelPrevious: true,
    () async {
      emit(state.copyWith(searching: true));

      final result = await ref
          .read(searchRepositoryProvider)
          .search(keyword);

      emit(state.copyWith(
        searching: false,
        result: result,
      ));
    },
  );
}
```

明确不做：

- 不把 mounted 判断放进 State 或 `copyWith()`。
- 不让 Repository 或 AppApi 接收 `CancelToken`。
- 不通过关闭全局 Dio 取消页面请求。
- 不要求业务直接使用 `ProviderRequestScope` 或 `CancelToken`。

## 3. 设计思想

本方案按职责拆分，而不是把所有逻辑塞进一个类。

```text
Controller / Notifier
  -> 管页面状态和生命周期入口

Repository
  -> 管业务数据

AppApi
  -> 管接口定义

Dio Interceptor
  -> 管 CancelToken 注入

State
  -> 只管数据
```

最终原则：

- Controller 管生命周期。
- Repository 管业务数据。
- AppApi 保持纯接口。
- Dio interceptor 管 token 注入。
- RequestBinding 管异步上下文传递。
- State 只管数据。
- 每个 provider 实例一个 request scope。
- 每个 `request(...)` 一个 `CancelToken`。
- 同 key 可以取消旧请求。
- provider dispose 只取消当前 provider scope 内的请求。

## 4. 总体架构

方案包含 5 个核心组件。

```text
AppLifecycleNotifier
  -> 业务唯一入口，暴露 emit / update / request / cancelRequests

AppStateEmitter
  -> 安全更新 state

ProviderRequestScope
  -> 管理当前 provider 实例的 active requests

RequestBinding
  -> 把当前 request 的 CancelToken 绑定到异步调用链

LifecycleCancelInterceptor
  -> Dio 请求发出前自动注入当前 CancelToken
```

### AppLifecycleNotifier

业务 Controller 使用的 mixin。业务只接触这个 mixin 提供的 API：

```dart
emit(...)
update(...)
await request(...)
cancelRequests(...)
cancelAllRequests()
```

### AppStateEmitter

只负责 mounted 后安全更新 state：

- `emit(value)`：provider 仍 mounted 时才执行 `state = value`。
- `update(transform)`：provider 仍 mounted 时才读取当前 state 并更新。

### ProviderRequestScope

每个 provider 实例一个 scope。

职责：

- 构造时注册一次 `ref.onDispose(dispose)`。
- 管理当前 provider 发起的 active requests。
- provider dispose 时统一取消当前 scope 内未完成请求。
- 支持 `key` 和 `cancelPrevious`。

### RequestBinding

抽象“如何把 token 传给 Dio 拦截器”。

第一版使用 Dart Zone。Zone 只作为基础设施细节，不暴露给业务。

### LifecycleCancelInterceptor

Dio 请求发出前读取当前 request token，并注入到 `RequestOptions.cancelToken`。

## 5. 关键组件代码骨架

### 5.1 AppLifecycleNotifier

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';

typedef AppRequestErrorHandler =
    void Function(Object error, StackTrace stackTrace);

mixin AppLifecycleNotifier<StateT, ValueT>
    on AnyNotifier<StateT, ValueT> {
  AppStateEmitter<StateT, ValueT>? _emitter;
  ProviderRequestScope? _requestScope;

  AppStateEmitter<StateT, ValueT> get _stateEmitter {
    return _emitter ??= AppStateEmitter<StateT, ValueT>(
      notifier: this,
      isMounted: () => ref.mounted,
    );
  }

  ProviderRequestScope get _requests {
    return _requestScope ??= ProviderRequestScope(
      ref: ref,
      binding: appRequestBinding,
      isMounted: () => ref.mounted,
    );
  }

  void emit(StateT value) {
    _stateEmitter.emit(value);
  }

  void update(StateT Function(StateT current) transform) {
    _stateEmitter.update(transform);
  }

  Future<T?> request<T>(
    Future<T> Function() action, {
    Object? key,
    bool cancelPrevious = false,
    AppRequestErrorHandler? onError,
  }) {
    return _requests.run<T>(
      action,
      key: key,
      cancelPrevious: cancelPrevious,
      onError: onError,
    );
  }

  void cancelRequests(Object key) {
    _requests.cancel(key);
  }

  void cancelAllRequests() {
    _requests.cancelAll();
  }
}
```

### 5.2 AppStateEmitter

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

### 5.3 RequestBinding

```dart
import 'dart:async';

import 'package:dio/dio.dart';

abstract interface class RequestBinding {
  CancelToken? get currentToken;

  Future<T> bind<T>(
    CancelToken token,
    Future<T> Function() action,
  );
}

final Object _cancelTokenZoneKey = Object();

final RequestBinding appRequestBinding = ZoneRequestBinding();

final class ZoneRequestBinding implements RequestBinding {
  @override
  CancelToken? get currentToken {
    return Zone.current[_cancelTokenZoneKey] as CancelToken?;
  }

  @override
  Future<T> bind<T>(
    CancelToken token,
    Future<T> Function() action,
  ) {
    return runZoned(
      action,
      zoneValues: {
        _cancelTokenZoneKey: token,
      },
    );
  }
}
```

### 5.4 ProviderRequestScope

```dart
import 'package:dio/dio.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

final class ProviderRequestScope {
  ProviderRequestScope({
    required Ref ref,
    required RequestBinding binding,
    required bool Function() isMounted,
  })  : _ref = ref,
        _binding = binding,
        _isMounted = isMounted {
    _ref.onDispose(dispose);
  }

  final Ref _ref;
  final RequestBinding _binding;
  final bool Function() _isMounted;

  final Set<_TrackedRequest> _requests = <_TrackedRequest>{};
  var _disposed = false;

  Future<T?> run<T>(
    Future<T> Function() action, {
    Object? key,
    bool cancelPrevious = false,
    AppRequestErrorHandler? onError,
  }) async {
    if (_disposed || !_isMounted()) {
      return null;
    }

    if (key != null && cancelPrevious) {
      cancel(key);
    }

    final tracked = _TrackedRequest(
      key: key,
      token: CancelToken(),
    );

    _requests.add(tracked);

    try {
      return await _binding.bind<T>(tracked.token, action);
    } on DioException catch (error, stackTrace) {
      if (CancelToken.isCancel(error)) {
        return null;
      }

      if (!_isMounted()) {
        return null;
      }

      onError?.call(error, stackTrace);
      return null;
    } catch (error, stackTrace) {
      if (!_isMounted()) {
        return null;
      }

      onError?.call(error, stackTrace);
      return null;
    } finally {
      _requests.remove(tracked);
    }
  }

  void cancel(Object key) {
    for (final request in _requests.where((item) => item.key == key).toList()) {
      request.cancel('request cancelled: $key');
    }
  }

  void cancelAll() {
    for (final request in _requests.toList()) {
      request.cancel('all requests cancelled');
    }
  }

  void dispose() {
    if (_disposed) {
      return;
    }

    _disposed = true;

    for (final request in _requests.toList()) {
      request.cancel('provider disposed');
    }

    _requests.clear();
  }
}

final class _TrackedRequest {
  _TrackedRequest({
    required this.key,
    required this.token,
  });

  final Object? key;
  final CancelToken token;

  void cancel(Object reason) {
    if (!token.isCancelled) {
      token.cancel(reason);
    }
  }
}
```

### 5.5 LifecycleCancelInterceptor

```dart
import 'package:dio/dio.dart';

class LifecycleCancelInterceptor extends Interceptor {
  const LifecycleCancelInterceptor(this._binding);

  final RequestBinding _binding;

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    final token = _binding.currentToken;

    if (token != null && options.cancelToken == null) {
      options.cancelToken = token;
    }

    handler.next(options);
  }
}
```

Dio 注册顺序建议放在业务拦截器前：

```dart
dio.interceptors.add(LifecycleCancelInterceptor(appRequestBinding));

dio.interceptors.add(
  ApiInterceptor(
    headerProvider: ref.watch(httpHeaderProvider),
    crypto: ref.watch(httpCryptoProvider),
  ),
);
```

## 6. 请求执行流程

业务代码：

```dart
await request(() async {
  final data = await repository.fetch();
  emit(state.copyWith(data: data));
});
```

完整流程：

```text
Controller.load()
  -> AppLifecycleNotifier.request()
  -> AppLifecycleNotifier._requests
  -> 懒创建 ProviderRequestScope
  -> ProviderRequestScope 构造时注册 ref.onDispose(dispose)
  -> ProviderRequestScope.run()
  -> 创建 _TrackedRequest(token A, key null)
  -> _requests.add(tracked A)
  -> RequestBinding.bind(token A, action)
  -> Zone.current 保存 token A
  -> action 执行
  -> repository.fetch()
  -> AppApi.fetch()
  -> Retrofit 生成代码调用 Dio
  -> Dio 执行 LifecycleCancelInterceptor.onRequest()
  -> interceptor 读取 binding.currentToken
  -> 读取到 token A
  -> options.cancelToken = token A
  -> Dio 请求绑定 token A
  -> 请求成功返回
  -> emit(...) 安全更新 state
  -> finally: _requests.remove(tracked A)
  -> request Future 完成
```

关键点：

- `_requests` 集合只负责生命周期管理。
- Zone 负责让当前异步调用链能读取 token。
- Dio interceptor 负责把 token 写入 `RequestOptions.cancelToken`。

## 7. Provider Dispose 自动取消流程

页面不再监听 autoDispose provider 后：

```text
页面退出
  -> provider dispose
  -> Riverpod 执行 ref.onDispose listeners
  -> ProviderRequestScope.dispose()
  -> _disposed = true
  -> 遍历 _requests
  -> cancel token A / token B / ...
  -> Dio 对应请求抛 DioExceptionType.cancel
  -> ProviderRequestScope.run 捕获取消异常
  -> 返回 null，不进入 onError
  -> finally 清理 tracked request
```

`ref.onDispose` 可以注册多次，不会和业务自己的清理逻辑冲突：

```dart
@override
HealthState build() {
  ref.onDispose(() {
    textController.dispose();
  });

  return const HealthState();
}
```

请求 scope 只是额外注册了一个 dispose listener：

```dart
_ref.onDispose(dispose);
```

它不会覆盖业务注册的 listener。

## 8. 并发请求与 Key 机制

### 8.1 多个 request 并发

```dart
request(() async {
  await repository.fetchA();
});

request(() async {
  await repository.fetchB();
});
```

执行结果：

```text
request A -> token A -> fetchA 使用 token A
request B -> token B -> fetchB 使用 token B

当前 provider scope:
  _requests = { tracked A, tracked B }

provider dispose:
  cancel token A
  cancel token B
```

不会取消其他页面请求。其他页面使用的是另一个 provider 实例、另一个
`ProviderRequestScope`、另一组 tokens。

### 8.2 一个 request 内多个 Dio 请求

```dart
await request(() async {
  final result = await Future.wait([
    repository.fetchUser(),
    repository.fetchOrders(),
  ]);

  emit(state.copyWith(result: result));
});
```

执行结果：

```text
一个 request -> token A

fetchUser   -> token A
fetchOrders -> token A
```

这两个请求属于同一个生命周期任务，provider dispose 时一起取消。

### 8.3 key + cancelPrevious

搜索场景：

```dart
await request(
  key: 'search',
  cancelPrevious: true,
  () async {
    final result = await repository.search(keyword);
    emit(state.copyWith(result: result));
  },
);
```

第一次输入：

```text
search("a")
  -> key search
  -> token A
  -> _requests = { A }
```

第二次输入：

```text
search("ab")
  -> cancelPrevious == true
  -> cancel('search')
  -> cancel token A
  -> 创建 token B
  -> _requests = { B }
```

结果：

```text
旧请求 A 被取消
新请求 B 继续
只有 B 有机会更新 state
```

## 9. 业务使用规范

### 9.1 Controller 默认写法

```dart
@riverpod
class HealthController extends _$HealthController
    with AppLifecycleNotifier<HealthState, HealthState> {
  @override
  HealthState build() {
    return const HealthState();
  }

  Future<void> load() async {
    await request(
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

### 9.2 返回值写法

```dart
Future<void> load() async {
  final data = await request(() {
    return ref.read(appRepositoryProvider).health();
  });

  if (data == null) {
    return;
  }

  emit(state.copyWith(data: data));
}
```

注意：`request` 外更新 state 仍然必须使用 `emit`。

### 9.3 搜索写法

```dart
Future<void> search(String keyword) async {
  await request(
    key: 'search',
    cancelPrevious: true,
    () async {
      emit(state.copyWith(searching: true));

      final result = await ref
          .read(searchRepositoryProvider)
          .search(keyword);

      emit(state.copyWith(
        searching: false,
        result: result,
      ));
    },
    onError: (error, stackTrace) {
      emit(state.copyWith(
        searching: false,
        error: error,
      ));
    },
  );
}
```

### 9.4 手动取消某类请求

```dart
void cancelSearch() {
  cancelRequests('search');
}
```

### 9.5 取消当前 provider 所有请求

```dart
void cancelAll() {
  cancelAllRequests();
}
```

## 10. Repository 和 AppApi 保持无感

Repository 不接收 `CancelToken`：

```dart
class AppRepository {
  const AppRepository(this._api);

  final AppApi _api;

  Future<dynamic> health() async {
    final response = await _api.health();
    if (!response.isSuccess) {
      throw ApiException(
        code: response.code,
        message: response.message ?? 'Request failed',
        raw: response,
      );
    }

    return response.data;
  }
}
```

AppApi 不接收 `CancelToken`：

```dart
@RestApi()
abstract class AppApi {
  factory AppApi(Dio dio, {String? baseUrl}) = _AppApi;

  @GET(ApiUrl.health)
  Future<ApiResponse<dynamic>> health();
}
```

生命周期只属于 Controller / Notifier，不下沉到 Repository。

## 11. 错误处理策略

### 11.1 Cancel 异常

Dio 请求被取消时会抛 `DioException`，并且：

```dart
CancelToken.isCancel(error) == true
```

默认行为：

```text
吞掉
返回 null
不进入 onError
不更新 error state
```

取消不是业务失败，而是生命周期结束或新请求替换旧请求。

### 11.2 普通异常

普通异常流程：

```text
catch error
  -> 如果 provider 已 dispose，直接忽略
  -> 如果 provider 仍 mounted，调用 onError
```

如果没有传 `onError`，第一版建议返回 `null`，不重新抛出，保持业务调用简单。

如果后续需要强制暴露异常，可以扩展：

```dart
bool rethrowUnhandled = false
```

第一版不建议加入，避免 API 复杂。

## 12. 为什么 request 内必须 await

错误写法：

```dart
await request(() async {
  repository.fetchUser(); // 错误：没有 await
});
```

实际流程：

```text
request()
  -> 创建 token A
  -> _requests.add(token A)
  -> 进入 Zone A
  -> 调用 repository.fetchUser()
  -> action 立刻结束
  -> finally 执行
  -> _requests.remove(token A)
```

后果：

- provider dispose 时 `_requests` 里已经没有 token A，可能取消不到。
- 请求异常不会进入 `onError`。
- loading 状态可能提前结束。
- 后续 `.then(...)` 里的更新脱离 request 生命周期管理。

正确写法：

```dart
await request(() async {
  final user = await repository.fetchUser();
  emit(state.copyWith(user: user));
});
```

并发正确写法：

```dart
await request(() async {
  final results = await Future.wait([
    repository.fetchUser(),
    repository.fetchOrders(),
  ]);

  emit(state.copyWith(
    user: results[0],
    orders: results[1],
  ));
});
```

核心规则：

```text
request 管的是 action 返回的 Future。
没有 await 的网络 Future 不属于这个 request。
```

## 13. 容易出问题的地方

### 13.1 provider 不是 autoDispose

如果 provider 是 keepAlive 或全局 provider，页面退出不会 dispose provider，请求也不会自动取消。

页面级 Controller 默认使用 autoDispose。

### 13.2 多页面共享同一个 provider 实例

如果多个页面共享同一个 provider 实例，它们也共享同一个 request scope。

只有这个 provider 实例 dispose 时才会取消请求。

页面独立状态应使用 family 或页面级参数隔离 provider 实例。

### 13.3 fire-and-forget

禁止：

```dart
request(() async {
  repository.fetch();
});
```

必须：

```dart
request(() async {
  await repository.fetch();
});
```

### 13.4 Zone 不跨 isolate

`RequestBinding` 第一版使用 Zone。普通 `async/await`、`Future.wait` 可以保留 Zone。

跨 isolate 不会自动保留 Zone。如果某个请求是在 isolate 中创建的，它拿不到当前 token。

### 13.5 第三方库重置 Zone

大多数库不会这么做。如果某个库内部显式使用自己的 `runZoned` 并丢失外部
`zoneValues`，可能导致 token 不可见。

这类情况需要通过测试和拦截器日志排查。

### 13.6 不要用 dio.close 做页面取消

页面请求取消应该 cancel token，不应该关闭全局 Dio。

```dart
token.cancel();
```

只取消绑定该 token 的请求。

```dart
dio.close();
```

会影响所有使用这个 Dio 的请求，不适合页面生命周期取消。

### 13.7 State 不要感知生命周期

不要让 State 或 `copyWith` 依赖 Riverpod：

```dart
state.copyWith(ref: ref)
```

生命周期属于 Notifier / request scope，State 是纯数据。

## 14. 测试建议

### 14.1 AppStateEmitter

- mounted 时 `emit` 更新 state。
- disposed 后 `emit` 不抛错。
- `update` 使用当前 state 计算新 state。
- disposed 后 `update` 不读取 state、不抛错。

### 14.2 ProviderRequestScope

- `run` 创建 token 并在完成后移除。
- provider dispose 时取消所有 active request。
- 多个 active request 都被取消。
- `cancel(key)` 只取消匹配 key 的请求。
- `cancelAll()` 取消当前 scope 全部请求。
- cancel 异常不进入 `onError`。
- 普通异常在 mounted 时进入 `onError`。

### 14.3 RequestBinding

- `bind` 内可以读取当前 token。
- bind 外 `currentToken` 为 null。
- 并发 bind 各自读取各自 token。
- 嵌套 bind 时内层使用内层 token，内层结束后外层恢复外层 token。

### 14.4 LifecycleCancelInterceptor

- Zone 内请求自动注入 token。
- Zone 外请求不注入 token。
- options 已有 `cancelToken` 时不覆盖。
- 并发 Dio 请求 token 不串。

### 14.5 集成测试

- Controller request 发起 Dio 请求后 dispose provider，请求被取消。
- dispose 后请求完成不更新 state。
- `key + cancelPrevious` 只允许最后一次搜索更新 state。
- AppApi / Repository 方法不暴露 `CancelToken`。

## 15. 建议文件落点

```text
lib/core/riverpod/app_state_emitter.dart
lib/core/riverpod/request_binding.dart
lib/core/riverpod/provider_request_scope.dart
lib/core/riverpod/app_lifecycle_notifier.dart
lib/core/riverpod/app_riverpod.dart
lib/app/http/interceptors/lifecycle_cancel_interceptor.dart
```

Dio 注册修改：

```text
lib/app/http/providers.dart
```

## 16. 最终原则

- 页面级网络请求必须通过 `request(...)` 发起。
- 状态更新必须通过 `emit(...)` 或 `update(...)`。
- 请求取消绑定 provider 实例生命周期。
- provider dispose 只取消当前 provider scope 内请求。
- Repository / AppApi 不暴露 `CancelToken`。
- State / `copyWith` 不感知生命周期。
- `request(...)` 内部创建的网络 Future 必须 `await`、`return` 或进入 `Future.wait`。
- 搜索、刷新、重复点击等场景使用 `key + cancelPrevious`。
