# Guía para integrar la navegación de JanitorAI en Android (Kotlin)

Si estás desarrollando una app nativa en Android con Kotlin, puedes usar la misma técnica de Glaze para navegar por JanitorAI saltando la protección de Cloudflare.

## 1. Arquitectura

Necesitas un `WebView` (puede estar oculto) que cargue `janitorai.com`. Para hacer peticiones a la API:
1.  Cargas la URL base en el WebView.
2.  Inyectas un script que realiza un `fetch()`.
3.  Como `fetch` es asíncrono en JS, usamos una **`JavascriptInterface`** para enviar el resultado de vuelta a Kotlin cuando esté listo.

## 2. Código del Proxy (`JanitorProxy.kt`)

Este motor maneja la comunicación bidireccional entre Kotlin y el motor de JavaScript del WebView.

```kotlin
import android.annotation.SuppressLint
import android.content.Context
import android.webkit.JavascriptInterface
import android.webkit.WebView
import android.webkit.WebViewClient
import kotlinx.coroutines.CompletableDeferred
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import java.util.UUID

class JanitorProxy(context: Context) {
    private val pendingRequests = mutableMapOf<String, CompletableDeferred<String>>()

    @SuppressLint("SetJavaScriptEnabled")
    private val webView = WebView(context).apply {
        settings.javaScriptEnabled = true
        settings.domStorageEnabled = true
        addJavascriptInterface(ProxyInterface(), "KotlinProxy")

        webViewClient = object : WebViewClient() {
            override fun onPageFinished(view: WebView?, url: String?) {
                isPageLoaded = true
                loadDeferred.complete(Unit)
            }
        }
        loadUrl("https://janitorai.com")
    }

    private var isPageLoaded = false
    private val loadDeferred = CompletableDeferred<Unit>()

    inner class ProxyInterface {
        @JavascriptInterface
        fun onResult(requestId: String, result: String) {
            pendingRequests[requestId]?.complete(result)
            pendingRequests.remove(requestId)
        }

        @JavascriptInterface
        fun onError(requestId: String, error: String) {
            pendingRequests[requestId]?.completeExceptionally(Exception(error))
            pendingRequests.remove(requestId)
        }
    }

    suspend fun fetch(url: String): String = withContext(Dispatchers.Main) {
        if (!isPageLoaded) loadDeferred.await()

        val requestId = UUID.randomUUID().toString()
        val deferred = CompletableDeferred<String>()
        pendingRequests[requestId] = deferred

        val script = """
            (async function() {
                try {
                    const r = await fetch('$url', {
                        headers: { 'Accept': 'application/json' },
                        credentials: 'include'
                    });
                    const body = await r.text();
                    KotlinProxy.onResult('$requestId', body);
                } catch (e) {
                    KotlinProxy.onError('$requestId', e.message);
                }
            })()
        """.trimIndent()

        webView.evaluateJavascript(script, null)
        deferred.await()
    }
}
```

## 3. Servicio de Datos (`JanitorService.kt`)

```kotlin
import org.json.JSONObject

class JanitorService(private val proxy: JanitorProxy) {
    private val hampterUrl = "https://janitorai.com/hampter/characters"

    suspend fun searchCharacters(query: String = "", page: Int = 1): String {
        val url = "$hampterUrl?sort=trending&page=$page&mode=all" +
                  if (query.isNotEmpty()) "&search=$query" else ""

        return proxy.fetch(url)
    }

    fun resolveAvatar(avatarPath: String?): String {
        if (avatarPath == null) return ""
        if (avatarPath.startsWith("http")) return avatarPath
        return "https://ella.janitorai.com/bot-avatars/$avatarPath?width=400"
    }
}
```

## 4. Configuración Necesaria

Asegúrate de permitir el tráfico en tu `AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

## 5. Manejo de Turnstile (Cloudflare)

Si la respuesta del proxy es un error 403 o contiene el HTML de Cloudflare, significa que se requiere una acción manual:
1.  **Muestra el WebView** al usuario (puedes usar un Dialog o un Fragment).
2.  El usuario marca "Soy humano".
3.  Cloudflare deposita la cookie `cf_clearance`.
4.  Ocultas el WebView y el proxy volverá a funcionar automáticamente en segundo plano.

---
**Nota importante:** Debido a las restricciones de seguridad de Android, el WebView debe crearse y manipularse siempre en el **Hilo Principal (Main Thread)**. He incluido `withContext(Dispatchers.Main)` en el método `fetch` para asegurar esto.
