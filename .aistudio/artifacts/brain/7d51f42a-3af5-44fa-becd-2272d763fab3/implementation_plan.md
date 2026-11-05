# Control-Tareas: Plan Técnico y Arquitectura de Viabilidad

Plan de ingeniería y análisis de viabilidad técnica para la aplicación **Control-Tareas** en Google AI Studio Build, adoptando un enfoque dual: un simulador interactivo de alta fidelidad con diseño Material 3 para previsualización inmediata en el entorno web, acompañado de la estructura y código fuente nativo de Android en Kotlin y Jetpack Compose listo para compilar en Android Studio.

---

## Decisiones Críticas y Confirmaciones

> [!IMPORTANT]
> **Decisiones confirmadas por el usuario en Fase 1:**
> 1. **Estructura del Proyecto en AI Studio**: Enfoque dual. Un simulador interactivo web Material 3 en la raíz (`/src`) para validación inmediata de UX y lógica en el iFrame de AI Studio, y una carpeta estructurada de código fuente nativo (`/android`) con Gradle, Kotlin, Jetpack Compose, Room y SQLCipher listo para abrir en Android Studio.
> 2. **Cifrado de Base de Datos**: Generación y custodia de clave maestra de 256 bits en **Android Keystore** vinculada a autenticación biométrica (`BiometricPrompt` con `BIOMETRIC_STRONG` o credencial del dispositivo PIN/Patrón) para inicializar SQLCipher.
> 3. **Migración por QR**: Mecanismo de **códigos QR secuenciales (multipart) y comprimidos** (Gzip/Deflate + codificación Base45/Base64) que permite transferir bases de datos o colecciones de tareas completas entre dispositivos sin conexión a Internet y sin depender de redes locales.

---

## 1. Evaluación de Viabilidad Técnica en Google AI Studio Build

### 1.1 Diagnóstico del Entorno de Ejecución
* **Contenedor del Agente**: Entorno Linux (gVisor) con Node.js 22, npm y servidor Vite en puerto 3000 con renderizado en iFrame web.
* **Herramientas Nativas Disponibles**: No se cuenta con JDK (javac), Gradle, Android SDK Platform Tools (`adb`, `aapt2`) ni emulador de Android nativo dentro del contenedor.
* **Veredicto de Viabilidad**:
  - **Ejecución directa de APK en AI Studio**: No es viable compilar ni ejecutar un emulador Android dentro del contenedor de AI Studio.
  - **Generación de código fuente nativo Kotlin/Compose**: 100% viable. Se generarán los archivos fuente (`build.gradle.kts`, `AndroidManifest.xml`, entidades Room, DAOs, ViewModels, Composables y tests) con sintaxis e importaciones de producción.
  - **Simulador Web Interactivo de Alta Fidelidad**: 100% viable y complementario. Permite probar inmediatamente en el navegador el flujo de creación de tareas, el cifrado (vía WebCrypto AES-GCM), el generador/lector de QR secuenciales (usando WebCam y canvas), y la simulación de notificaciones.

---

## 2. Arquitectura del Sistema (Enfoque Dual)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       REPOSITORIO / PROYECTO DUAL                           │
├──────────────────────────────────────┬──────────────────────────────────────┤
│    A. SIMULADOR WEB (AI Studio)      │  B. CÓDIGO NATIVO ANDROID (Studio)   │
│       React 19 + TypeScript + Vite   │     Kotlin 2.0 + Jetpack Compose     │
├──────────────────────────────────────┼──────────────────────────────────────┤
│ • UI Material You (M3) en Tailwind   │ • Material 3 Compose UI + NavHost    │
│ • WebCrypto AES-GCM (Simula Keystore)│ • Android Keystore + BiometricPrompt │
│ • LocalStorage / IndexedDB Cifrado   │ • Room Database + SQLCipher SQLite   │
│ • QR Multipart Generator + Scanner   │ • CameraX + ZXing / ML Kit Scanner   │
│ • Web Notifications + Audio Alerts   │ • WorkManager + AlarmManager Exact   │
│ • Exportador de Respaldos .json.enc  │ • Storage Access Framework (SAF) URI │
│ • Asistente Gemini AI Integrado      │ • Google GenAI SDK / Remote API      │
└──────────────────────────────────────┴──────────────────────────────────────┘
```

### 2.1 Mapeo de Componentes entre Ambos Entornos

| Capa Funcional | Entorno Web (Simulador AI Studio) | Entorno Nativo Android (Android Studio) |
| :--- | :--- | :--- |
| **Interfaz de Usuario** | Componentes React con diseño Material 3 (FAB, BottomSheet, TopAppBar) | Jetpack Compose (`Scaffold`, `ModalBottomSheet`, `FloatingActionButton`) |
| **Cifrado de Datos** | Web Crypto API (`SubtleCrypto` AES-GCM 256 bits, PBKDF2) | `EncryptedSharedPreferences`, Android Keystore (`AES/GCM/NoPadding`), SQLCipher |
| **Base de Datos** | Repositorio IndexedDB / Dexie con serialización cifrada | Room Database 2.6+ con `SupportOpenHelper` de SQLCipher |
| **Migración QR** | Canvas QR secuencial a 10 fps + `html5-qrcode` / `jsQR` | CameraX ImageAnalysis + ZXing Barcode Scanner multi-frame |
| **Notificaciones** | Web Notification API (`Notification.requestPermission`) | `NotificationManagerCompat`, `NotificationChannel`, `AlarmManager` |
| **Copias de Seguridad** | File System Access API / Blob Download cifrado `.enc` | Storage Access Framework (`Intent.ACTION_CREATE_DOCUMENT`) |

---

## 3. Análisis de Riesgos Técnicos y Estrategia de Mitigación

### 3.1 Cifrado de Base de Datos y Gestión de Claves
* **Riesgo 1**: Incompatibilidad de binarios de SQLCipher con arquitecturas de CPU (`armeabi-v7a`, `arm64-v8a`, `x86_64`).
  * *Mitigación*: Utilizar la dependencia oficial `net.zetetic:sqlcipher-android:4.5.4` configurando `pickFirst` en `packagingOptions` para evitar colisiones de `.so`.
* **Riesgo 2**: Pérdida de clave si el usuario cambia credenciales biométricas o bloqueo de pantalla.
  * *Mitigación*: En Android Keystore, configurar `setUserAuthenticationRequired(true)` con fallback a PIN del dispositivo. Proporcionar además una frase de recuperación (Seed Phrase o clave de emergencia) de 12 palabras para restaurar la base de datos en caso de invalidación de la clave biométrica.

### 3.2 Migración de Datos mediante Códigos QR Secuenciales
* **Riesgo 1**: Densidad excesiva de datos en un solo QR que impida su lectura por cámaras de gama baja.
  * *Mitigación*: Fragmentar el payload en bloques ("chunks") de máximo 400 bytes. Cada bloque contendrá una cabecera binaria `[id_sesion: 2B][indice: 1B][total: 1B][checksum_crc32: 4B]`.
* **Riesgo 2**: Compresión insuficiente en bases de datos con cientos de tareas.
  * *Mitigación*: Comprimir el payload con algoritmo Gzip/Deflate antes de la fragmentación y codificar en Base45 (más eficiente para códigos QR alfanuméricos que Base64, reduciendo el tamaño del símbolo en un 45%).
* **Riesgo 3**: Pérdida de frames durante la visualización secuencial.
  * *Mitigación*: Mostrar un carrusel dinámico en bucle a 4–6 fps indicando el progreso de lectura del receptor (ej. `Recibidos: 8/12`). El receptor almacena los fragmentos por índice hasta completar el 100% y verificar el hash SHA-256 total.

### 3.3 Copias de Seguridad (Backup)
* **Riesgo 1**: Restricciones de almacenamiento de Android (Scoped Storage en Android 11+).
  * *Mitigación*: Uso obligatorio de `ActivityResultContracts.CreateDocument("application/octet-stream")` mediante Storage Access Framework (SAF), permitiendo al usuario guardar el archivo en cualquier carpeta local o Google Drive sin requerir permisos peligrosos de almacenamiento (`MANAGE_EXTERNAL_STORAGE`).
* **Riesgo 2**: Corrupción o manipulación del archivo de respaldo.
  * *Mitigación*: Encriptar el JSON exportado con AES-256-GCM. El archivo tendrá la estructura: `[Magic Header 4B: 'CTAR'][Salt 16B][IV 12B][Ciphertext + Auth Tag 16B]`. Se verificará la autenticidad antes de iniciar cualquier proceso de restauración.

### 3.4 Notificaciones Locales y Alarmas Exactas
* **Riesgo 1**: Cancelación o retraso de alarmas por modos de ahorro de batería (Doze Mode).
  * *Mitigación*: Usar `AlarmManager.setExactAndAllowWhileIdle()` para recordatorios con hora crítica. En Android 12+ (API 31+), solicitar permiso `SCHEDULE_EXACT_ALARM` o verificar `canScheduleExactAlarms()`.
* **Riesgo 2**: Notificaciones bloqueadas en Android 13+ (API 33+).
  * *Mitigación*: Solicitar en tiempo de ejecución el permiso `android.permission.POST_NOTIFICATIONS` con una pantalla explicativa previa (Rationale) que eduque al usuario sobre su necesidad para recordatorios.
* **Riesgo 3**: Reinicio del dispositivo borra las alarmas registradas en `AlarmManager`.
  * *Mitigación*: Registrar un `BroadcastReceiver` para `ACTION_BOOT_COMPLETED` y `ACTION_MY_PACKAGE_REPLACED` que reconsulte la base de datos Room y reprograme automáticamente todas las tareas pendientes.

---

## 4. Etapas Verificables de Desarrollo

```
Fase 1: Núcleo de Datos y Cifrado
├── Entidades de Tareas (id, título, descripción, fecha, prioridad, estado, etiquetas)
├── Motor de Cifrado (Keystore / WebCrypto AES-GCM)
└── Base de Datos (Room + SQLCipher / IndexedDB Cifrado)

Fase 2: Interfaz Material 3 y Gestión de Tareas
├── Listado de Tareas con filtros (Pendientes, En progreso, Completadas)
├── Creación y edición con Bottom Sheet / Modal accesible
└── Indicadores visuales de prioridad y fecha de vencimiento

Fase 3: Recordatorios y Notificaciones
├── Programación de alarmas exactas con AlarmManager / Web Notifications
├── Canales de notificación con sonido y prioridad alta
└── Receptor de reinicio (BootReceiver) para persistencia de alarmas

Fase 4: Migración QR Secuencial y Respaldos Cifrados
├── Algoritmo de compresión Gzip + Base45 + Chunking
├── Generador de QR dinámicos en bucle y escáner lector con progreso
└── Exportación e importación de respaldos cifrados con SAF (.ctar)

Fase 5: Asistente Inteligente de Productividad y Refinamiento
├── Integración de Gemini con Google Search Grounding para sugerencias de tareas
├── Exportador completo del árbol de archivos Android Studio (.zip / carpeta /android)
└── Pruebas de compilación, verificación de linting y empaquetado
```

---

## 5. Árbol de Dependencias de Android

### 5.1 Dependencias Clave (`app/build.gradle.kts`)
* **Jetpack Compose**:
  - `androidx.compose.ui:ui:1.7.0`
  - `androidx.compose.material3:material3:1.3.0`
  - `androidx.navigation:navigation-compose:2.8.0`
* **Persistencia y Cifrado**:
  - `androidx.room:room-runtime:2.6.1`
  - `androidx.room:room-ktx:2.6.1`
  - `net.zetetic:sqlcipher-android:4.5.4`
  - `androidx.security:security-crypto:1.1.0-alpha06`
  - `androidx.biometric:biometric:1.2.0-alpha05`
* **Segundo Plano y Alarmas**:
  - `androidx.work:work-runtime-ktx:2.9.1`
* **Cámara y Escaneo QR**:
  - `androidx.camera:camera-camera2:1.3.4`
  - `androidx.camera:camera-lifecycle:1.3.4`
  - `androidx.camera:camera-view:1.3.4`
  - `com.google.zxing:core:3.5.3`

---

## 6. Verificación y Siguientes Pasos

Una vez aprobado este plan:
1. Procederemos a desarrollar progresivamente cada fase según los requisitos funcionales específicos que proporciones.
2. Cada fase será verificable y reproducible directamente en el visor web interactivo de AI Studio y en el código exportable para Android Studio.
