# Flutter Integration Guide

Complete implementation for a Flutter app connecting to a Total.js API Routing backend.
Stack: **Flutter + Dart + http + flutter_secure_storage + Provider**.

Adapt `totaljsbackend.com` and the example resource `posts` to your project.

---

## Setup

Add to `pubspec.yaml`:

```yaml
dependencies:
  flutter:
    sdk: flutter
  http: ^1.2.1
  flutter_secure_storage: ^9.0.0
  provider: ^6.1.2
  web_socket_channel: ^3.0.1
```

```bash
flutter pub get
```

### Platform setup for flutter_secure_storage

**Android** — `android/app/build.gradle`:
```gradle
android {
  defaultConfig {
    minSdkVersion 18   // required by secure storage
  }
}
```

**iOS** — no additional setup required.

---

## Project structure

```
lib/
  api/
    token_storage.dart     ← SecureStorage wrapper
    api_client.dart        ← http client
    api_request.dart       ← apiRequest() function
  services/
    auth_service.dart
    posts_service.dart
  providers/
    auth_provider.dart
  utils/
    error_handler.dart
  screens/
    login_screen.dart
    register_screen.dart
    posts_screen.dart
  main.dart
```

---

## Token storage — `lib/api/token_storage.dart`

```dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

class TokenStorage {
  static const _storage = FlutterSecureStorage(
    aOptions: AndroidOptions(encryptedSharedPreferences: true),
  );
  static const _key = 'totaljs_session_token';

  static Future<String?> get() => _storage.read(key: _key);
  static Future<void> set(String token) => _storage.write(key: _key, value: token);
  static Future<void> clear() => _storage.delete(key: _key);
}
```

---

## Exceptions — `lib/api/api_client.dart`

```dart
class UnauthorizedException implements Exception {
  const UnauthorizedException();
  @override
  String toString() => 'Session expired. Please log in again.';
}

class ApiException implements Exception {
  final String message;
  final int? statusCode;
  const ApiException(this.message, {this.statusCode});
  @override
  String toString() => message;
}
```

---

## API Client — `lib/api/api_client.dart` (continued)

```dart
import 'dart:convert';
import 'package:http/http.dart' as http;
import 'token_storage.dart';

const String _baseUrl = 'https://totaljsbackend.com';

class ApiClient {
  static Future<Map<String, String>> _headers() async {
    final headers = <String, String>{'Content-Type': 'application/json'};
    final token = await TokenStorage.get();
    if (token != null) headers['x-token'] = token;
    return headers;
  }

  static Future<dynamic> post(String path, Map<String, dynamic> body) async {
    final uri = Uri.parse('$_baseUrl$path');
    final headers = await _headers();

    final response = await http.post(uri, headers: headers, body: jsonEncode(body));

    if (response.statusCode == 401) {
      await TokenStorage.clear();
      throw const UnauthorizedException();
    }

    if (response.statusCode >= 500) {
      throw ApiException('Server error (${response.statusCode})', statusCode: response.statusCode);
    }

    // Total.js always returns JSON
    return jsonDecode(response.body);
  }
}
```

---

## API Request function — `lib/api/api_request.dart`

```dart
import 'api_client.dart';

/// The single function for all Total.js API calls.
/// [schema] — e.g. "posts_list", "posts_read/abc123", "posts_list?page=2"
/// [data]   — optional request payload
Future<dynamic> apiRequest(String schema, [Map<String, dynamic>? data]) {
  final body = <String, dynamic>{'schema': schema};
  if (data != null) body['data'] = data;
  return ApiClient.post('/api/', body);
}

/// Helper to build a schema string with query parameters.
String buildSchema(String base, [Map<String, dynamic>? params]) {
  if (params == null || params.isEmpty) return base;
  final filtered = Map.fromEntries(
    params.entries.where((e) => e.value != null),
  );
  if (filtered.isEmpty) return base;
  final qs = filtered.entries.map((e) => '${e.key}=${Uri.encodeComponent(e.value.toString())}').join('&');
  return '$base?$qs';
}
```

---

## Error handler — `lib/utils/error_handler.dart`

```dart
import '../api/api_client.dart';

String extractErrorMessage(dynamic error) {
  // Total.js array response: [{ "error": "..." }]
  if (error is List && error.isNotEmpty) {
    final first = error[0];
    if (first is Map) return (first['error'] ?? first['value'] ?? 'An error occurred').toString();
  }

  // Total.js object response: { "error": "..." }
  if (error is Map) return (error['error'] ?? error['message'] ?? 'An error occurred').toString();

  if (error is UnauthorizedException) return error.toString();
  if (error is ApiException) return error.message;
  if (error is Exception) return error.toString().replaceFirst('Exception: ', '');
  if (error is String) return error;

  return 'An unexpected error occurred';
}
```

---

## Auth service — `lib/services/auth_service.dart`

```dart
import '../api/api_request.dart';
import '../api/token_storage.dart';

class AuthService {
  // Login — normalizes array or object response
  Future<Map<String, dynamic>> login(String email, String password) async {
    final raw = await apiRequest('account_login', {'email': email, 'password': password});
    final item = (raw is List) ? raw[0] as Map<String, dynamic> : raw as Map<String, dynamic>;

    if (item['success'] != true) throw item['error'] ?? 'Login failed';

    final token = (item['token'] ?? item['value'])?.toString();
    if (token != null) await TokenStorage.set(token);
    return item;
  }

  Future<Map<String, dynamic>> register(String name, String email, String password) async {
    final res = await apiRequest('account_create', {'name': name, 'email': email, 'password': password})
        as Map<String, dynamic>;
    if (res['success'] == true && res['value'] != null) {
      await TokenStorage.set(res['value'].toString());
    }
    return res;
  }

  Future<Map<String, dynamic>> getProfile() async =>
      await apiRequest('account') as Map<String, dynamic>;

  Future<void> logout() async {
    await apiRequest('account_logout').catchError((_) {});
    await TokenStorage.clear();
  }

  Future<void> changePassword(String current, String next) =>
      apiRequest('account_password', {'current_password': current, 'new_password': next});

  Future<void> requestPasswordReset(String email) =>
      apiRequest('account_reset', {'email': email});

  Future<void> verifyAccount(String token) =>
      apiRequest('account_verify', {'token': token});

  // Mobile OAuth — direct token exchange
  Future<Map<String, dynamic>> loginWithGoogle(String idToken) async {
    final res = await apiRequest('account_login_google', {'token': idToken}) as Map<String, dynamic>;
    final token = (res['value'] as Map?)?['token']?.toString() ?? res['token']?.toString();
    if (token != null) await TokenStorage.set(token);
    return res;
  }

  Future<Map<String, dynamic>> loginWithGithub(String accessToken) async {
    final res = await apiRequest('account_login_github', {'token': accessToken}) as Map<String, dynamic>;
    final token = (res['value'] as Map?)?['token']?.toString() ?? res['token']?.toString();
    if (token != null) await TokenStorage.set(token);
    return res;
  }

  // 2FA
  Future<dynamic> generate2FA() => apiRequest('account_2fa_generate');
  Future<dynamic> enable2FA(String token) => apiRequest('account_2fa_enable', {'token': token});
  Future<dynamic> disable2FA() => apiRequest('account_2fa_disable');
  Future<dynamic> verify2FA(String token) => apiRequest('account_2fa_verify', {'token': token});
}
```

---

## Auth provider — `lib/providers/auth_provider.dart`

```dart
import 'package:flutter/foundation.dart';
import '../api/token_storage.dart';
import '../api/api_client.dart';
import '../services/auth_service.dart';
import '../utils/error_handler.dart';

class AuthProvider extends ChangeNotifier {
  final _service = AuthService();

  Map<String, dynamic>? _user;
  bool _isLoading = true;
  String? _error;

  Map<String, dynamic>? get user => _user;
  bool get isAuthenticated => _user != null;
  bool get isLoading => _isLoading;
  String? get error => _error;

  /// Call once on app start (before runApp or in main())
  Future<void> initialize() async {
    final token = await TokenStorage.get();
    if (token == null) { _isLoading = false; notifyListeners(); return; }

    try {
      final res = await _service.getProfile();
      if (res['success'] == true) _user = res['value'] as Map<String, dynamic>;
    } on UnauthorizedException {
      // Token cleared by ApiClient
    } catch (_) {
      await TokenStorage.clear();
    } finally {
      _isLoading = false;
      notifyListeners();
    }
  }

  Future<bool> login(String email, String password) async {
    _error = null;
    try {
      final item = await _service.login(email, password);
      _user = (item['value'] as Map<String, dynamic>?) ?? {'email': email};
      notifyListeners();
      return true;
    } catch (e) {
      _error = extractErrorMessage(e);
      notifyListeners();
      return false;
    }
  }

  Future<bool> register(String name, String email, String password) async {
    _error = null;
    try {
      await _service.register(name, email, password);
      final res = await _service.getProfile();
      if (res['success'] == true) _user = res['value'] as Map<String, dynamic>;
      notifyListeners();
      return true;
    } catch (e) {
      _error = extractErrorMessage(e);
      notifyListeners();
      return false;
    }
  }

  Future<void> logout() async {
    await _service.logout();
    _user = null;
    notifyListeners();
  }

  void clearError() { _error = null; notifyListeners(); }
}
```

### Wire into app — `lib/main.dart`

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  final auth = AuthProvider();
  await auth.initialize();

  runApp(
    ChangeNotifierProvider.value(
      value: auth,
      child: const MyApp(),
    ),
  );
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: Consumer<AuthProvider>(
        builder: (_, auth, __) {
          if (auth.isLoading) return const SplashScreen();
          return auth.isAuthenticated ? const PostsScreen() : const LoginScreen();
        },
      ),
    );
  }
}
```

---

## Example service — `lib/services/posts_service.dart`

```dart
import '../api/api_request.dart';

class PostsService {
  Future<List<dynamic>> list({int? page, int? limit, String? status}) async {
    final schema = buildSchema('posts_list', {
      if (page != null) 'page': page,
      if (limit != null) 'limit': limit,
      if (status != null) 'status': status,
    });
    final res = await apiRequest(schema) as Map<String, dynamic>;
    return (res['value'] as List?) ?? [];
  }

  Future<Map<String, dynamic>> read(String id) async {
    final res = await apiRequest('posts_read/$id') as Map<String, dynamic>;
    return res['value'] as Map<String, dynamic>;
  }

  Future<Map<String, dynamic>> create(String title, String body, String status) async {
    final res = await apiRequest('posts_create', {
      'title': title,
      'body': body,
      'status': status,
    }) as Map<String, dynamic>;
    return res['value'] as Map<String, dynamic>;
  }

  Future<Map<String, dynamic>> update(String id, Map<String, dynamic> data) async {
    final res = await apiRequest('posts_update/$id', data) as Map<String, dynamic>;
    return res['value'] as Map<String, dynamic>;
  }

  Future<void> remove(String id) => apiRequest('posts_remove/$id');
}
```

---

## Login screen — `lib/screens/login_screen.dart`

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import '../providers/auth_provider.dart';

class LoginScreen extends StatefulWidget {
  const LoginScreen({super.key});
  @override
  State<LoginScreen> createState() => _LoginScreenState();
}

class _LoginScreenState extends State<LoginScreen> {
  final _emailCtrl = TextEditingController();
  final _passwordCtrl = TextEditingController();
  bool _loading = false;

  Future<void> _submit() async {
    setState(() => _loading = true);
    final auth = context.read<AuthProvider>();
    final ok = await auth.login(_emailCtrl.text.trim(), _passwordCtrl.text);
    if (mounted) setState(() => _loading = false);
    // On success, AuthProvider.isAuthenticated → true → MyApp rebuilds to PostsScreen
    // On failure, auth.error is set — read it from the provider
  }

  @override
  Widget build(BuildContext context) {
    final error = context.select<AuthProvider, String?>((a) => a.error);

    return Scaffold(
      body: SafeArea(
        child: Padding(
          padding: const EdgeInsets.all(24),
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              Text('Sign in', style: Theme.of(context).textTheme.headlineMedium),
              const SizedBox(height: 32),
              TextField(
                controller: _emailCtrl,
                decoration: const InputDecoration(labelText: 'Email', border: OutlineInputBorder()),
                keyboardType: TextInputType.emailAddress,
                autocorrect: false,
              ),
              const SizedBox(height: 16),
              TextField(
                controller: _passwordCtrl,
                decoration: const InputDecoration(labelText: 'Password', border: OutlineInputBorder()),
                obscureText: true,
              ),
              if (error != null) ...[
                const SizedBox(height: 12),
                Text(error, style: const TextStyle(color: Colors.red)),
              ],
              const SizedBox(height: 24),
              SizedBox(
                width: double.infinity,
                child: FilledButton(
                  onPressed: _loading ? null : _submit,
                  child: _loading
                      ? const SizedBox(height: 20, width: 20,
                          child: CircularProgressIndicator(strokeWidth: 2, color: Colors.white))
                      : const Text('Sign in'),
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }

  @override
  void dispose() {
    _emailCtrl.dispose();
    _passwordCtrl.dispose();
    super.dispose();
  }
}
```

---

## File upload — two-step pattern

```dart
import 'dart:io';
import 'package:http/http.dart' as http;
import 'dart:convert';
import '../api/api_request.dart';

Future<Map<String, dynamic>> uploadAndRegisterDocument(
  File file, {
  required String uploadToken,
  String bucket = 'documents',
}) async {
  // Step 1: Upload to the file service
  final request = http.MultipartRequest(
    'POST',
    Uri.parse('https://fs.totaljsbackend.com/upload/$bucket/?token=$uploadToken&hostname=1'),
  );
  request.files.add(await http.MultipartFile.fromPath('file', file.path));

  final streamed = await request.send();
  if (streamed.statusCode != 200) {
    throw Exception('Upload failed with status ${streamed.statusCode}');
  }

  final body = await streamed.stream.bytesToString();
  final uploadData = jsonDecode(body) as Map<String, dynamic>;

  // Step 2: Register in the app via main API
  final res = await apiRequest('documents_create', {
    'id':   uploadData['id'],
    'url':  uploadData['url'],
    'name': uploadData['name'] ?? file.uri.pathSegments.last,
    'size': uploadData['size'],
    'type': uploadData['type'],
  }) as Map<String, dynamic>;

  return res['value'] as Map<String, dynamic>;
}
```

---

## WebSocket — `lib/api/backend_socket.dart`

```dart
import 'dart:async';
import 'dart:convert';
import 'package:web_socket_channel/web_socket_channel.dart';
import 'token_storage.dart';

class BackendSocket {
  final String path;           // e.g. "/ws/"
  final Map<String, String> params;
  final void Function(Map<String, dynamic>) onMessage;
  final void Function()? onConnected;
  final void Function()? onDisconnected;

  WebSocketChannel? _channel;
  StreamSubscription? _sub;
  Timer? _retryTimer;
  int _retries = 0;
  bool _disposed = false;

  BackendSocket({
    required this.path,
    this.params = const {},
    required this.onMessage,
    this.onConnected,
    this.onDisconnected,
  });

  Future<void> connect() async {
    if (_disposed) return;

    final token = await TokenStorage.get() ?? '';
    final allParams = Map<String, String>.from(params)..['token'] = token;
    final qs = allParams.entries.map((e) => '${e.key}=${Uri.encodeComponent(e.value)}').join('&');
    final uri = Uri.parse('wss://totaljsbackend.com$path?$qs');

    _channel = WebSocketChannel.connect(uri);
    onConnected?.call();

    _sub = _channel!.stream.listen(
      (raw) {
        try {
          final msg = jsonDecode(raw.toString()) as Map<String, dynamic>;
          if (!_disposed) onMessage(msg);
        } catch (_) {}
      },
      onDone: _reconnect,
      onError: (_) => _reconnect(),
      cancelOnError: true,
    );
  }

  void _reconnect() {
    if (_disposed) return;
    onDisconnected?.call();
    final delay = Duration(milliseconds: (1000 * (1 << _retries)).clamp(1000, 30000));
    _retries = (_retries + 1).clamp(0, 6);
    _retryTimer = Timer(delay, connect);
  }

  void disconnect() {
    _disposed = true;
    _retryTimer?.cancel();
    _sub?.cancel();
    _channel?.sink.close(1000);
  }
}
```

Usage in a StatefulWidget:

```dart
class NotificationsScreen extends StatefulWidget {
  const NotificationsScreen({super.key});
  @override
  State<NotificationsScreen> createState() => _NotificationsScreenState();
}

class _NotificationsScreenState extends State<NotificationsScreen> {
  late final BackendSocket _socket;
  final List<Map<String, dynamic>> _events = [];

  @override
  void initState() {
    super.initState();
    _socket = BackendSocket(
      path: '/ws/',
      onMessage: (msg) {
        if (msg['type'] == 'notification' && mounted) {
          setState(() => _events.insert(0, msg['data'] as Map<String, dynamic>));
        }
      },
    );
    _socket.connect();
  }

  @override
  void dispose() {
    _socket.disconnect();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Notifications')),
      body: ListView.builder(
        itemCount: _events.length,
        itemBuilder: (_, i) => ListTile(
          title: Text(_events[i]['title']?.toString() ?? ''),
          subtitle: Text(_events[i]['body']?.toString() ?? ''),
        ),
      ),
    );
  }
}
```

---

## Key differences from React / React Native

| Concern | React (web) | React Native | Flutter |
|---------|-------------|--------------|---------|
| HTTP client | axios | axios | `http` package |
| Token storage | `localStorage` | `expo-secure-store` | `flutter_secure_storage` |
| State management | Context API | Zustand | Provider / Riverpod |
| 401 redirect | `window.location.href` | `authEvents` + `navigationRef` | `ChangeNotifier.notifyListeners()` → root rebuilds |
| WebSocket | `WebSocket` API | `WebSocket` API | `web_socket_channel` |
| File upload | `fetch` + `FormData` | `fetch` + `FormData` | `http.MultipartRequest` |
| Env variables | `import.meta.env.VITE_*` | `process.env.EXPO_PUBLIC_*` | hardcoded or `--dart-define` |

The API contract — schema strings, `{ schema, data }` body, `{ success, value, error }` response, `x-token` header — is **identical** across all three platforms. Only the tooling around it changes.
