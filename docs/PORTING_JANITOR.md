# Guía para integrar la navegación de JanitorAI en tu App Flutter

Para poder navegar por los personajes de JanitorAI en tu propia aplicación, necesitas implementar un sistema que pueda saltar la protección de Cloudflare. JanitorAI utiliza **Cloudflare Turnstile**, que bloquea peticiones directas (vía Dio o Http) porque detecta que no vienen de un navegador real.

La solución que usa Glaze es un **Proxy mediante un WebView oculto (Headless)**.

## 1. Dependencias Necesarias

Agrega esto a tu `pubspec.yaml`:

```yaml
dependencies:
  flutter_inappwebview: ^6.1.5 # Muy importante para el proxy
  shared_preferences: ^2.3.2  # Para guardar cookies de sesión
  dio: ^5.7.0                 # Para descargar imágenes
```

## 2. Arquitectura del Proxy

El truco consiste en:
1.  Tener un `InAppWebView` oculto que siempre esté en `https://janitorai.com`.
2.  Para buscar personajes, NO haces un `dio.get()`. En su lugar, le pides al WebView que ejecute un `fetch()` en JavaScript **dentro** de la página de Janitor.
3.  Como el WebView ya pasó el reto de Cloudflare, la petición `fetch()` heredará todas las cookies y el "visto bueno" del navegador.

## 3. Código del Proxy (`janitor_proxy.dart`)

Este archivo contiene el motor que hace las peticiones.

```dart
import 'dart:async';
import 'dart:convert';
import 'package:flutter/foundation.dart';
import 'package:flutter_inappwebview/flutter_inappwebview.dart';

class JanitorProxy {
  static final JanitorProxy instance = JanitorProxy._();
  JanitorProxy._();

  HeadlessInAppWebView? _webView;
  InAppWebViewController? _controller;
  Completer<void>? _starting;
  Completer<void>? _loadStop;

  Future<void> _ensureStarted() async {
    if (_controller != null) return;
    if (_starting != null) return _starting!.future;

    _starting = Completer<void>();
    _loadStop = Completer<void>();

    _webView = HeadlessInAppWebView(
      initialUrlRequest: URLRequest(url: WebUri('https://janitorai.com')),
      initialSettings: InAppWebViewSettings(
        javaScriptEnabled: true,
        domStorageEnabled: true,
        thirdPartyCookiesEnabled: true,
      ),
      onWebViewCreated: (c) => _controller = c,
      onLoadStop: (c, url) {
        if (!_loadStop!.isCompleted) _loadStop!.complete();
      },
    );

    await _webView!.run();
    await _loadStop!.future;
    _starting!.complete();
  }

  /// Ejecuta una petición HTTP dentro del WebView para saltar Cloudflare
  Future<String> fetch(String url) async {
    await _ensureStarted();

    // Ejecutamos fetch de JS dentro del WebView
    final result = await _controller!.callAsyncJavaScript(
      functionBody: '''
        const r = await fetch(${jsonEncode(url)}, {
          headers: { "Accept": "application/json" },
          credentials: "include",
        });
        return { status: r.status, body: await r.text() };
      ''',
    );

    if (result == null || result.error != null) {
      throw Exception('Error en el proxy: ${result?.error}');
    }

    final value = result.value;
    if (value['status'] >= 400) {
      throw Exception('HTTP Error ${value['status']}');
    }

    return value['body'];
  }

  void dispose() {
    _webView?.dispose();
    _webView = null;
    _controller = null;
  }
}
```

## 4. Servicio de búsqueda (`janitor_service.dart`)

Usa el proxy para obtener los datos de la API de Janitor.

```dart
import 'dart:convert';
import 'janitor_proxy.dart';

class JanitorService {
  static const String _hampterUrl = 'https://janitorai.com/hampter/characters';

  Future<List<dynamic>> searchCharacters({String query = '', int page = 1}) async {
    final params = 'sort=trending&page=$page&mode=all';
    final url = '$_hampterUrl?$params${query.isNotEmpty ? "&search=$query" : ""}';

    final body = await JanitorProxy.instance.fetch(url);
    final data = jsonDecode(body);

    if (data is List) return data;
    return (data['characters'] as List?) ?? [];
  }

  Future<Map<String, dynamic>> getCharacterDetails(String id) async {
    final url = '$_hampterUrl/$id';
    final body = await JanitorProxy.instance.fetch(url);
    return jsonDecode(body);
  }

  String resolveAvatar(String? avatarPath) {
    if (avatarPath == null) return '';
    if (avatarPath.startsWith('http')) return avatarPath;
    return 'https://ella.janitorai.com/bot-avatars/$avatarPath?width=400';
  }
}
```

## 5. Modelos de datos (`models.dart`)

```dart
class JanitorCharacter {
  final String id;
  final String name;
  final String? avatar;
  final String description;

  JanitorCharacter({
    required this.id,
    required this.name,
    this.avatar,
    required this.description,
  });

  factory JanitorCharacter.fromJson(Map<String, dynamic> json) {
    return JanitorCharacter(
      id: json['id'] ?? '',
      name: json['name'] ?? 'Unknown',
      avatar: json['avatar'] ?? json['image'],
      description: json['description'] ?? '',
    );
  }
}
```

## 6. Consideraciones para Windows e iOS

Si tu app corre en **Windows**, debes inicializar el entorno de WebView2 antes de usarlo. En Glaze lo hacemos así:

```dart
// En tu main() o antes de usar el proxy
if (defaultTargetPlatform == TargetPlatform.windows) {
  await WebViewEnvironment.create();
}
```

## 7. Retos de Cloudflare (Turnstile)

A veces Cloudflare pide un reto interactivo (marcar la casilla de "Soy humano"). Si el proxy falla con un error 403, debes mostrar un WebView **visible** al usuario solo una vez para que resuelva el reto. Una vez resuelto, las cookies se guardan y el proxy vuelve a funcionar en segundo plano.

Puedes detectar esto si `value['status'] == 403` en el `fetch` del proxy.

---
**Nota:** JanitorAI cambia su API (llamada internamente "hampter") ocasionalmente. Si deja de funcionar, revisa las URLs en la pestaña de Network de tu navegador mientras usas JanitorAI.com.
