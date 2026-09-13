# DOCUMENTO TÉCNICO DE ARQUITECTURA HTTP Y RED

**Asignatura:** Desarrollo Móvil Avanzado  
**Estudiante:** AleQr01  
**Enlace al Repositorio:** https://github.com/AleQr01/SEMANA-13_APLICACIONES-MOVILES  

---

## 1. Selección y Configuración del Cliente HTTP

### Justificación de la Elección
Se ha seleccionado **Dio** como cliente HTTP para el proyecto debido a las siguientes razones técnicas:
* **Soporte de `QueuedInterceptor`:** Permite detener y encolar solicitudes concurrentes mientras se ejecuta la renovación asíncrona del token tras un error HTTP 401.
* **Configuración Centralizada:** Facilita la definición global de *timeouts* explícitos y cabeceras predeterminadas.
* **Manejo Estructurado de Excepciones:** Clasifica de forma nativa los tipos de errores de red (`DioExceptionType`).

```dart
// lib/core/network/http_client_factory.dart
import 'package:dio/dio.dart';

enum Environment {
  dev('[http://10.0.2.2:3000/api/v1](http://10.0.2.2:3000/api/v1)'), // IP local para emulador Android
  qa('[https://qa-api.midominio.com/api/v1](https://qa-api.midominio.com/api/v1)'),
  prod('[https://api.midominio.com/api/v1](https://api.midominio.com/api/v1)');

  final String baseUrl;
  const Environment(this.baseUrl);
}

class HttpClientFactory {
  static Dio create(Environment env) {
    // Exigencia de HTTPS en producción
    if (env == Environment.prod && !env.baseUrl.startsWith('https://')) {
      throw Exception('La configuración de producción exige protocolo HTTPS');
    }

    final options = BaseOptions(
      baseUrl: env.baseUrl,
      connectTimeout: const Duration(seconds: 10),
      receiveTimeout: const Duration(seconds: 10),
      sendTimeout: const Duration(seconds: 10),
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
      },
    );

    return Dio(options);

## 2. Cadena de Interceptores y Orden de Ejecución

Los interceptores se registran en una secuencia estricta para garantizar que la autenticación y la gestión de errores se apliquen correctamente:

1. **`AuthInterceptor`:** Lee el *Access Token* del almacenamiento cifrado (`FlutterSecureStorage`) e inyecta la cabecera `Authorization: Bearer <token>` en cada petición saliente.
2. **`TokenRefreshInterceptor` (`QueuedInterceptor`):** Escucha las respuestas de error `401 Unauthorized`. Pausa la cola de peticiones, llama al endpoint `/auth/refresh` y reintenta la petición fallida.
   * **Protección contra bucles infinitos:** Marca la petición reintentada con `requestOptions.extra['is_retry'] = true`. Si una solicitud reintentada vuelve a devolver 401, se fuerza el cierre de sesión y no se intenta renovar nuevamente.
3. **`LogInterceptor`:** Muestra información de depuración en consola durante el desarrollo.

```dart
// lib/core/network/interceptors/token_refresh_interceptor.dart
class TokenRefreshInterceptor extends QueuedInterceptor {
  final Dio dio;
  final SecureStorageService storage;

  TokenRefreshInterceptor({required this.dio, required this.storage});

  @override
  Future<void> onError(DioException err, ErrorInterceptorHandler handler) async {
    if (err.response?.statusCode == 401) {
      final requestOptions = err.requestOptions;

      // Validación para evitar bucles infinitos de reintentos
      final isAlreadyRetried = requestOptions.extra['is_retry'] == true;
      final isRefreshEndpoint = requestOptions.path.contains('/auth/refresh');

      if (isAlreadyRetried || isRefreshEndpoint) {
        await storage.clearAll();
        return handler.next(err);
      }

      try {
        final refreshToken = await storage.getRefreshToken();
        if (refreshToken == null) {
          await storage.clearAll();
          return handler.next(err);
        }

        final refreshDio = Dio(BaseOptions(baseUrl: dio.options.baseUrl));
        final response = await refreshDio.post('/auth/refresh', data: {'refresh_token': refreshToken});

        if (response.statusCode == 200) {
          final newAccessToken = response.data['access_token'];
          final newRefreshToken = response.data['refresh_token'];

          await storage.saveTokens(accessToken: newAccessToken, refreshToken: newRefreshToken);

          // Marca explícita para evitar bucles
          requestOptions.extra['is_retry'] = true;
          requestOptions.headers['Authorization'] = 'Bearer $newAccessToken';

          final clonedResponse = await dio.fetch(requestOptions);
          return handler.resolve(clonedResponse);
        }
      } catch (e) {
        await storage.clearAll();
      }
    }
    handler.next(err);
  }
}
