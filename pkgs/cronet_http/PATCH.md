# cronet_http 1.6.0 — LOCAL PATCH (JNI global ref leak + quicHints)

Bản vendored từ `cronet_http` **1.6.0** (pub.dev) + bản vá, dùng qua
`dependency_overrides` trong `pubspec.yaml` ở **workspace root**.
Override ở root phủ **toàn bộ** workspace, gồm cả `app` lẫn another workspace package
(dùng cronet_http qua `NativeHttpAdapterFactory`) — nên không cần sửa source chúng.

## ⚠️ Vì sao 1.6.0 (không phải 1.8.0)

Trước đây vendored 1.8.0 (jni `^0.15.2`). Khi tích hợp Datadog, `datadog_flutter_plugin`
3.x yêu cầu `jni ^0.14.2` (`<0.15.0`) → xung đột cứng với cronet 1.7.0+ (jni `^0.15.2`).
Hạ về **1.6.0** (jni `^0.14.2`) để khớp Datadog. Bản 1.6.0 thiếu tham số `quicHints`
ở `CronetEngine.build()` (that package cần) — nhưng binding `addQuicHint` ĐÃ có sẵn
trong `jni_bindings.dart` của 1.6.0, nên đã **port tham số `quicHints` từ 1.8.0** vào
`build()` (đánh dấu `LOCAL PATCH`). Kết quả: jni 0.14.2 (Datadog OK) + fix leak + quicHints.

Khi `datadog_flutter_plugin` hỗ trợ jni 0.15.x, có thể nâng lại cronet ≥1.7 và bỏ port quicHints.

Lưu ý: callback trong 1.6.0 KHÔNG bọc `using((arena){...})` như 1.8.0; ở 1.6.0 chỉ cần
thêm `requestHandle.release()` ở cuối mỗi terminal callback. Mô tả arena bên dưới là theo
1.8.0. Tìm mọi thay đổi bằng comment `LOCAL PATCH`.

## Vấn đề

Crash `SIGABRT: JNI ERROR (app bug): global reference table overflow (max=51200)`,
áp đảo bởi `org.chromium.net.impl.CronetUrlRequest` (quan sát: 45522/51200 ref),
crashing stack đi qua `libdartjni.so`.

Gốc trong `lib/src/cronet_client.dart` `CronetClient.send()`: object
`cronetRequest` (JNI global ref tới `UrlRequest`/`CronetUrlRequest`) **cố ý
không release**, chỉ neo sống qua `abortTrigger.whenComplete(cronetRequest.cancel)`
và trông chờ GC thu hồi. Với conversion layer của `native_dio_adapter`,
`abortTrigger` (= `Future.any([cancelFuture, timeoutCompleter.future])`) **chỉ
complete khi cancel/timeout** — request thành công bình thường thì KHÔNG complete
→ ref chỉ được nhả khi GC thu cả chuỗi future. Dưới tải request cao, bảng JNI
global ref (51200) đầy nhanh hơn GC → tràn → crash. Mỗi request thành công leak
1 ref ⇒ con số ≈ tổng số request trong phiên.

Bản mới nhất trên pub.dev là 1.8.0; changelog không có fix → bug chưa vá upstream.

## Bản vá

`lib/src/cronet_client.dart`:
- Thêm `_CronetRequestHandle`: giữ `cronetRequest`, `release()` nhả JNI global
  ref đúng 1 lần (idempotent), `cancel()` no-op sau khi đã release.
- `_urlRequestCallbacks(...)` nhận thêm `requestHandle`; gọi `requestHandle.release()`
  ở cả 3 terminal callback `onSucceeded` / `onFailed` / `onCanceled` (Cronet đảm
  bảo đúng một terminal callback; mọi đường hủy/redirect-limit đều dẫn về
  `onCanceled`).
- `send()`: tạo `requestHandle`, `attach(cronetRequest)` sau `build()`, và đổi
  `abortTrigger.whenComplete(cronetRequest.cancel)` → `...whenComplete(requestHandle.cancel)`
  (tránh cancel trên ref đã release).

Tìm các thay đổi bằng comment `LOCAL PATCH`.

## Chi tiết thay đổi code

Tất cả ở 1 file: `lib/src/cronet_client.dart`. Gồm 5 thay đổi logic (6 dấu
`LOCAL PATCH`). Mọi dòng/khu vực còn lại giữ NGUYÊN so với upstream 1.8.0.

### Thay đổi 1 — Thêm class holder `_CronetRequestHandle` (mới)

Thêm mới ngay sau `const _bufferSize = ...;` (trước `class _StreamedResponseWithUrl`).
Không có ở upstream.

```dart
class _CronetRequestHandle {
  jb.UrlRequest? _request;
  var _released = false;

  void attach(jb.UrlRequest request) => _request = request;

  void cancel() {            // wired vào abortTrigger; no-op sau khi đã release
    if (_released) return;
    _request?.cancel();
  }

  void release() {           // nhả JNI global ref đúng 1 lần (idempotent)
    if (_released) return;
    _released = true;
    _request?.release();
    _request = null;
  }
}
```

Ý nghĩa: `release()` được gọi tại terminal callback (deterministic), `cancel()`
thay cho `cronetRequest.cancel` ở nhánh abort và tự bảo vệ không cancel/đụng vào
ref đã release (tránh use-after-release).

### Thay đổi 2 — Thêm tham số `requestHandle` cho `_urlRequestCallbacks`

```dart
// BEFORE
jb.UrlRequestCallbackProxy$UrlRequestCallbackInterface _urlRequestCallbacks(
    BaseRequest request,
    Completer<CronetStreamedResponse> responseCompleter,
    HttpClientRequestProfile? profile) {

// AFTER
jb.UrlRequestCallbackProxy$UrlRequestCallbackInterface _urlRequestCallbacks(
    BaseRequest request,
    Completer<CronetStreamedResponse> responseCompleter,
    HttpClientRequestProfile? profile,
    _CronetRequestHandle requestHandle) {   // ← thêm
```

### Thay đổi 3,4,5 — Release tại 3 terminal callback

Thêm `requestHandle.release();` ngay sau khối `using((arena){...})` của mỗi
callback `onSucceeded`, `onFailed`, `onCanceled`. Mẫu giống nhau ở cả 3:

```dart
// onSucceeded — AFTER
onSucceeded: (urlRequest, responseInfo) {
  using((arena) {
    ...                                    // (nguyên gốc, không đổi)
    profile?.responseData.close();
  });
  requestHandle.release(); // LOCAL PATCH   // ← thêm
},
```

```dart
// onFailed — AFTER
onFailed: (urlRequest, responseInfo, cronetException) {
  using((arena) {
    ...                                    // (nguyên gốc)
    jByteBuffer?.release();
  });
  requestHandle.release(); // LOCAL PATCH   // ← thêm
},
```

```dart
// onCanceled — AFTER (callback cuối cùng Cronet đảm bảo gọi)
onCanceled: (urlRequest, urlResponseInfo) {
  using((arena) {
    ...                                    // (nguyên gốc)
    jByteBuffer?.release();
  });
  requestHandle.release(); // LOCAL PATCH   // ← thêm
},
```

Lý do đủ: Cronet đảm bảo đúng MỘT trong ba terminal callback chạy cho mỗi
request; mọi đường hủy (consumer cancel stream, `followRedirects=false`,
redirect-limit, content-length lỗi) đều dẫn về `onCanceled`. `release()`
idempotent nên double-call vô hại.

### Thay đổi 6 — Wiring trong `CronetClient.send()`

a) Tạo holder (ngay sau `final responseCompleter = ...;`):

```dart
final responseCompleter = Completer<CronetStreamedResponse>();
final requestHandle = _CronetRequestHandle();   // ← thêm
```

b) Truyền holder vào `_urlRequestCallbacks`:

```dart
// BEFORE
jb.UrlRequestCallbackProxy(
    _urlRequestCallbacks(request, responseCompleter, profile)),

// AFTER
jb.UrlRequestCallbackProxy(
    _urlRequestCallbacks(request, responseCompleter, profile,
        requestHandle)),                          // ← thêm tham số
```

c) Attach sau `build()` + đổi nhánh abort sang holder:

```dart
// BEFORE
// Not releasing `cronetRequest` as it's used in `whenComplete` callback.
final cronetRequest = builder.build()!;
if (request case Abortable(:final abortTrigger?)) {
  unawaited(abortTrigger.whenComplete(cronetRequest.cancel));
}
cronetRequest.start();
return responseCompleter.future;

// AFTER
final cronetRequest = builder.build()!;
requestHandle.attach(cronetRequest);                       // ← thêm
if (request case Abortable(:final abortTrigger?)) {
  unawaited(abortTrigger.whenComplete(requestHandle.cancel)); // ← đổi: cronetRequest.cancel → requestHandle.cancel
}
cronetRequest.start();
return responseCompleter.future;
```

`attach()` chạy TRƯỚC `start()` nên ref luôn có mặt trước khi bất kỳ callback nào
bắn ra → không race.

### Bất biến giữ nguyên (không phá vỡ hành vi)

- `cronetRequest` vẫn KHÔNG `releasedBy(arena)` (arena đóng sớm khi response
  bắt đầu; ref do `requestHandle` sở hữu, sống tới terminal callback).
- Mỗi callback vẫn release ref `urlRequest` riêng của nó qua `releasedBy(arena)`
  như cũ — patch không đụng phần này.
- Khả năng abort (cancel/timeout) giữ nguyên ngữ nghĩa; chỉ định tuyến qua
  `requestHandle.cancel` để an toàn với ref đã release.

## Quan hệ với `native_dio_adapter` PR #2517 (vì sao VẪN cần patch này)

Root `pubspec.yaml` đã pin `native_dio_adapter` vào git PR #2517 ("Android
memory leak fix", merge 2026-05-19). #2517 sửa `ConversionLayerAdapter`:
complete `abortTrigger` (timeoutCompleter) khi response stream kết thúc / khi
send lỗi (`onStreamDone: completeTimeout` + `catch`). Điều này làm
`abortTrigger.whenComplete(...cancel)` trong cronet_http bắn → **bỏ retention** →
`cronetRequest` trở thành **GC-eligible**.

NHƯNG #2517 chỉ đưa object về trạng thái GC-eligible — JNI global ref chỉ thực sự
được nhả khi GC finalize Dart wrapper. **Bằng chứng thực địa:** build prod ĐÃ có
#2517 vẫn crash `global reference table overflow` với `CronetUrlRequest` ≈
45522/51200 → dưới tải request cao, GC KHÔNG theo kịp tốc độ cấp ref → vẫn tràn.

Patch này release ref **deterministic** ngay tại terminal callback (không chờ
GC) → chặn đúng chỗ #2517 còn hở. Hai fix **bổ trợ và coexist an toàn**:
- #2517 complete abortTrigger → cronet_http gọi `requestHandle.cancel` → no-op
  vì đã `release()` ở terminal callback (guard `_released`).
- Thứ tự nào xảy ra trước cũng an toàn (cả `cancel`/`release` đều idempotent).

Khi `native_dio_adapter`/`cronet_http` có bản release chính thức xử lý dứt điểm
(release deterministic, không phụ thuộc GC), cân nhắc gỡ patch này.

## Khi nâng cấp / gỡ

Nếu upstream phát hành bản fix leak này, có thể xoá `packages/cronet_http` và
bỏ `dependency_overrides: cronet_http` trong `pubspec.yaml` (workspace root). Khi
bump cronet_http version khác, re-apply 3 chỗ `LOCAL PATCH` lên source mới.
