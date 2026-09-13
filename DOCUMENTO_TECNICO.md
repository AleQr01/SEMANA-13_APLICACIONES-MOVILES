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
  }
}
