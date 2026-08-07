# cronet_http 1.6.0 — patches (JNI global ref leak + quicHints)

Bản vá trên nền `cronet_http` **1.6.0**, giữ trên nhánh `patch/cronet_http-1.6.0`.
Dùng qua `dependency_overrides` (git hoặc path) ở **workspace root**, nên override
phủ toàn bộ workspace và không cần sửa source của từng package.

Tìm mọi thay đổi viết tay bằng comment `LOCAL PATCH`.

## Changelog

Mới nhất ở trên, mỗi generation một mục. Chi tiết kỹ thuật ở phần tương ứng bên dưới.

### AN-1 — giải phóng JNI ref của upload provider · 2026-08-06

Đường upload giữ JNI global ref của `uploadProvider` và body buffer mà không release,
nên mỗi request có body vẫn rò ref kể cả sau khi v1 đã vá 3 terminal callback.
Release chúng tại terminal, không đặt trong `finally`.

### v2 — detach callback proxy · 2026-08-06

Proxy của callback vẫn nằm lại trong registry `_$impls` sau khi request kết thúc, kéo
theo một `RawReceivePort` sống mãi cho mỗi request: entry chỉ được gỡ khi ART GC thu
Java proxy, việc gần như không xảy ra trong một phiên chạy thực tế. Đóng port và gỡ
entry ngay tại terminal callback, idempotent.

### v1 — release JNI global ref tại 3 terminal callback, và port `quicHints` · 2026-08-06

`CronetUrlRequest` không được release ở bất kỳ terminal callback nào, nên JNI global
reference table đầy dần rồi app abort:
`SIGABRT: JNI ERROR (app bug): global reference table overflow (max=51200)`.
Thêm một class holder giữ ref và release nó ở cả ba callback kết thúc
(succeeded / failed / canceled).

Cùng commit port tham số `quicHints` của `CronetEngine.build()` từ 1.8.0 về — binding
`addQuicHint` vốn đã có sẵn trong `jni_bindings.dart` của 1.6.0.

## ⚠️ Vì sao 1.6.0 (không phải 1.8.0)

Trước đây vendored 1.8.0 (jni `^0.15.2`). Khi tích hợp Datadog, `datadog_flutter_plugin`
3.x yêu cầu `jni ^0.14.2` (`<0.15.0`) → xung đột cứng với cronet 1.7.0+ (jni `^0.15.2`).
Hạ về **1.6.0** (jni `^0.14.2`) để khớp Datadog. Bản 1.6.0 thiếu tham số `quicHints`
ở `CronetEngine.build()` — nhưng binding `addQuicHint` ĐÃ có sẵn
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

Nếu upstream phát hành bản fix leak này, chỉ cần bỏ `dependency_overrides:
cronet_http` — quay về bản trên pub.dev.

Khi bump lên upstream version khác: cắt **nhánh mới** từ release commit của version
đó (`patch/cronet_http-<version>`) rồi re-apply 3 chỗ `LOCAL PATCH` lên source mới.
Không rebase nhánh này lên upstream mới — commit ở đây đang được pin theo SHA.

---

# Patch v2 — detach proxy registry `_$impls` tại terminal callback

## Vấn đề

Patch v1 release JNI global ref của `CronetUrlRequest` (hết crash overflow tức
thời) nhưng KHÔNG đụng registry phía Dart của callback proxy: mỗi request,
generated `jni_bindings.dart` đăng ký impl vào **static map `_$impls`** và mở
một **`RawReceivePort`** (cả hai là GC root). Entry chỉ được gỡ khi ART GC thu
Java proxy → `PortCleaner` (jni) gửi message `null` về port. Chuỗi dọn =
Dart-GC-finalizer → ART GC → port message: thực tế không chạy trong phiên →
**mỗi request leak vĩnh viễn trọn graph**: impl + 6 closures (capture
`responseCompleter`→`CronetStreamedResponse`, `UrlRequest` wrapper,
`JByteBuffer` 10KB direct, `_CronetRequestHandle`, `StreamController`) và — qua
`responseCompleter.future._zone` → custom Zone per-request của dio
(`createInterceptorZone`) — cả `InterceptorState`/`RequestOptions`/`Response`
của dio. Đã device-verify bằng VM Service `getRetainingPath` (report:
`$SPECS_ROOT/agent-docs/reports/chat/thread-open-transition-jank-by-leak-ram/`).
Upstream 1.9.0 (jni 1.0.0) vẫn nguyên pattern — bump version không tự khỏi.

## Bản vá (tìm theo comment `LOCAL PATCH (v2)`)

`lib/src/jni/jni_bindings.dart` — block `UrlRequestCallbackProxy$UrlRequestCallbackInterface`:
1. Thêm static map `_$ports: Map<int, RawReceivePort>` song song `_$impls`.
2. `implementIn`: đăng ký `_$ports[$a] = $p;` + nhánh null-message remove cả `_$ports`.
3. Thêm static `detachImpl(impl)`: tìm port theo `identical`, remove 2 map +
   `port.close()`. Idempotent; null-message của PortCleaner đến muộn = no-op.

`lib/src/cronet_client.dart`:
4. `_urlRequestCallbacks`: tách `impl` ra biến `late final`, helper `detach()`;
   gọi `detach()` ngay sau `requestHandle.release()` ở CẢ 3 terminal callback
   (`onSucceeded`/`onFailed`/`onCanceled` — Cronet đảm bảo đúng một terminal,
   không callback nào sau đó). Return type đổi thành record
   `(interfaceWrapper, detachFn)` để `send()` dọn được khi fail đồng bộ.
5. `send()`: bọc builder→`start()` trong try/catch/finally —
   catch (sync-throw, terminal callback sẽ không bao giờ bắn):
   `requestHandle.release()` + `detachCallbacks()` + rethrow;
   finally: `.release()` 2 Dart wrapper (`UrlRequestCallbackProxy` + interface)
   — Java builder/UrlRequest đã giữ ref riêng, nhả sớm để ART GC không bị ghim.

## Bất biến

- Không đổi hành vi HTTP: redirect/read/cancel/abort giữ nguyên; detach chỉ chạy
  sau terminal (hoặc sync-fail), khi Cronet đã cam kết không callback nữa.
- PortCleaner backstop giữ nguyên cho mọi đường sót (double remove/close vô hại).

## Khi bump version

Re-apply 5 vị trí trên (grep `LOCAL PATCH (v2)`); nếu upstream đổi
codegen (jni ≥1.0 vẫn cùng shape `_$impls`/RawReceivePort tính đến 1.9.0), map
tương ứng theo block class của interface.

---

# Patch AN-1 (upload-ref) — release JNI global ref của upload provider + body buffer

## Vấn đề

Heap-diff (Pixel 8, UAT, 54 lần gửi) cho thấy mỗi POST có body làm leak các class
Cronet đồng loạt: `CronetUploadDataStream +938`, `DirectByteBuffer +1586`,
`CronetUrlRequest +945`, `CronetMetrics +945`. Trong `send()`, nhánh upload tạo
`data = body.toJByteBuffer()` (JNI global ref #1) rồi
`builder.setUploadDataProvider(jb.UploadDataProviders.create$2(data), _executor)`
(ref #2, tạo inline, không giữ, không release). v1 chỉ release `UrlRequest`, v2
release thêm response `jByteBuffer` + detach registry — nhưng **cả hai không đụng
2 ref phía upload** → provider ghim trọn request graph → leak mỗi lần gửi.

## Bản vá (tìm theo comment `LOCAL PATCH`; không có tag v2)

`lib/src/cronet_client.dart`:
- `_CronetRequestHandle`: thêm `JObject? _uploadProvider` + `JByteBuffer? _uploadData`
  và `attachUpload(provider, data)`; trong `release()` (sau `_request?.release()`)
  release + null hoá 2 ref này. `cancel()` KHÔNG đụng — chỉ terminal `release()` nhả.
- `send()` nhánh `if (body.isNotEmpty)`: bắt `final uploadProvider =
  jb.UploadDataProviders.create$2(data)!` (create$2 trả `JObject?`),
  `requestHandle.attachUpload(uploadProvider, data)`, rồi truyền CHÍNH object đó
  vào `setUploadDataProvider` (không gọi `create$2` lần hai).

## Compose với v2 (không sửa dòng v2)

`cleanupTerminal()` đã gọi `requestHandle.release()` ở MỌI đường terminal
(onSucceeded/onFailed/onCanceled) → 2 ref upload nhả sau khi Cronet đọc xong body
(an toàn). Nhánh `catch` fail đồng bộ cũng gọi `requestHandle.release()` → an toàn
(request chưa chạy). KHÔNG release trong `finally` (chạy ngay sau `start()`, Cronet
còn đang đọc body → use-after-free). GET/no-body: 2 field null → `release()` no-op.

## Khi bump version

Re-apply 2 chỗ trên (grep `LOCAL PATCH`, phần `_uploadProvider`/`attachUpload`/
`uploadProvider`); giữ release ở terminal (`release()`), không đưa vào `finally`.
