# 📡 ESP32 WiFi Scanner Profesional 

Escáner avanzado de redes WiFi para ESP32 con sincronización NTP, análisis de canales y múltiples opciones de configuración.

### Funcionalidades Principales

- ✅ **Escaneo automático de redes WiFi** con intervalo configurable
- ⏰ **Sincronización de hora NTP** para timestamps precisos
- 📊 **Análisis de distribución de canales** con gráficos ASCII
- 🎯 **Filtrado de redes** por intensidad de señal
- 📱 **Detección de duplicados** (mismo SSID, diferentes AP)
- 🔒 **Identificación de tipos de seguridad** (WPA, WPA2, WPA3, etc.)
- 📄 **Exportación a formato JSON** (opcional)
- 🎨 **Interfaz visual mejorada** con indicadores de calidad

### Indicadores Visuales

- 🟢 **Señal excelente** (≥80%)
- 🟡 **Señal buena** (60-79%)
- 🟠 **Señal regular** (40-59%)
- 🔴 **Señal débil** (<40%)

---

## 🔧 Requisitos

### Hardware

- **ESP32** (cualquier modelo compatible)
- Cable USB para programación
- Computadora con Arduino IDE

### Software

- **Arduino IDE** 1.8.x o superior (o PlatformIO)
- **ESP32 Board Support Package** instalado
- Librerías incluidas (no requiere instalación adicional):
  - `WiFi.h`
  - `time.h`

---

## 📥 Instalación

### 1. Configurar Arduino IDE para ESP32

Si aún no tienes soporte para ESP32:

1. Abre Arduino IDE
2. Ve a **Archivo → Preferencias**
3. En "Gestor de URLs Adicionales de Tarjetas", agrega:
   ```
   https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
   ```
4. Ve a **Herramientas → Placa → Gestor de Tarjetas**
5. Busca "ESP32" e instala **esp32 by Espressif Systems**

### 2. Cargar el Código

1. Copia el código completo en un nuevo sketch de Arduino
2. Guarda el proyecto con un nombre (ej: `ESP32_WiFi_Scanner`)
3. Selecciona tu placa ESP32 en **Herramientas → Placa**
4. Selecciona el puerto COM correcto en **Herramientas → Puerto**

---

## ⚙️ Configuración

### Parámetros Principales

Edita estas constantes al inicio del código según tus necesidades:

```cpp
// ========== CONFIGURACIÓN ==========

// Sincronización de hora (NTP)
const char* ssid_sync = "TU_RED_WIFI";        // Tu red WiFi
const char* password_sync = "TU_CONTRASEÑA";  // Tu contraseña

// Intervalo de escaneo
const int SCAN_INTERVAL_MS = 10000;      // 10 segundos entre escaneos

// Filtro de señal
const int MIN_RSSI_FILTER = -100;        // -100 = mostrar todas las redes
                                         // -80 = solo redes con buena señal

// Opciones de visualización
const bool SHOW_DUPLICATES = true;       // true = mostrar APs duplicados
                                         // false = solo un SSID único

const bool EXPORT_JSON = false;          // true = exportar en JSON
                                         // false = solo tabla visual
```

### Configuración de Zona Horaria

Para ajustar tu zona horaria, modifica:

```cpp
const long gmtOffset_sec = -21600;       // UTC-6 para México
                                         // UTC-5 = -18000
                                         // UTC-3 = -10800
                                         // UTC+1 = 3600
```

---

## 🚀 Uso

### Inicio Rápido

1. **Configura tus credenciales WiFi** (líneas 11-12)
2. **Sube el código** a tu ESP32
3. **Abre el Monitor Serial** a **115200 baud**
4. Observa los escaneos automáticos

### Modos de Operación

#### Modo con Sincronización NTP (Recomendado)

Proporciona credenciales WiFi válidas para obtener timestamps precisos:

```cpp
const char* ssid_sync = "MiRedWiFi";
const char* password_sync = "MiContraseña123";
```

**Proceso:**
1. El ESP32 se conecta a tu red WiFi
2. Sincroniza la hora con servidor NTP
3. Se desconecta y comienza a escanear
4. Muestra fecha/hora en cada escaneo

#### Modo Sin Sincronización

Deja las credenciales por defecto:

```cpp
const char* ssid_sync = "TU_RED_WIFI";
const char* password_sync = "TU_CONTRASEÑA";
```

El escáner funcionará sin mostrar timestamps.

---

## 📝 Funciones del Código

### `void setup()`

**Propósito:** Inicialización del sistema

**Proceso:**
1. Inicia comunicación serial (115200 baud)
2. Muestra banner de bienvenida
3. Intenta conectar a WiFi (si hay credenciales válidas)
4. Sincroniza hora con servidor NTP
5. Desconecta WiFi y configura modo escáner
6. Prepara el sistema para escaneos continuos

**Salida típica:**
```
╔════════════════════════════════════════╗
║   ESP32 WiFi Scanner Profesional      ║
║   Versión 2.0 con mejoras             ║
╚════════════════════════════════════════╝

🌐 Conectando a WiFi para sincronizar hora...
   Red: MiRedWiFi
✅ Conectado a WiFi
   IP: 192.168.1.100
🕒 Hora NTP sincronizada correctamente
   Hora actual: 22/10/2025  14:30:45
```

---

### `void loop()`

**Propósito:** Escaneo continuo y presentación de resultados

**Flujo de ejecución:**

1. **Mostrar encabezado del escaneo**
   - Número de escaneo actual
   - Fecha y hora (si disponible)

2. **Realizar escaneo WiFi**
   ```cpp
   int numRedes = WiFi.scanNetworks();
   ```

3. **Filtrar redes**
   - Aplicar filtro de señal mínima (`MIN_RSSI_FILTER`)
   - Contar redes válidas

4. **Ordenar por señal**
   - Algoritmo de ordenamiento por intensidad RSSI
   - De mayor a menor señal

5. **Generar JSON** (si está activado)
   - Llamar a `exportJSON()`

6. **Mostrar tabla de resultados**
   - Iterar por cada red encontrada
   - Aplicar filtros configurados
   - Formatear y mostrar información

7. **Análisis de canales**
   - Llamar a `analyzeChannels()`

8. **Limpieza y espera**
   ```cpp
   WiFi.scanDelete();  // Liberar memoria
   delay(SCAN_INTERVAL_MS);
   ```

---

### `void analyzeChannels(int numRedes, int indices[])`

**Propósito:** Analizar y visualizar la distribución de canales WiFi

**Parámetros:**
- `numRedes`: Número total de redes encontradas
- `indices[]`: Array con índices ordenados por señal

**Funcionamiento:**

1. **Crear contador de canales**
   ```cpp
   int channelCount[14] = {0};  // Canales 1-13 (WiFi 2.4GHz)
   ```

2. **Contar redes por canal**
   ```cpp
   for (int i = 0; i < numRedes; i++) {
     int channel = WiFi.channel(indices[i]);
     if (channel >= 1 && channel <= 13) {
       channelCount[channel]++;
     }
   }
   ```

3. **Mostrar gráfico ASCII**
   ```
   📡 Distribución de canales:
   Canal │ Redes │ Gráfico
   ──────┼───────┼─────────────────────────────
    1    │   5   │ ▓▓▓▓▓
    6    │   8   │ ▓▓▓▓▓▓▓▓
   11    │   3   │ ▓▓▓
   ```

**Utilidad:** Identifica canales congestionados para optimizar tu router.

---

### `void exportJSON(int numRedes, int indices[])`

**Propósito:** Exportar datos de redes en formato JSON estándar

**Parámetros:**
- `numRedes`: Número total de redes
- `indices[]`: Array ordenado de índices

**Formato de salida:**

```json
{
  "networks": [
    {
      "ssid": "MiRedWiFi",
      "rssi": -45,
      "channel": 6,
      "bssid": "AA:BB:CC:DD:EE:FF",
      "encryption": 3
    },
    {
      "ssid": "OtraRed",
      "rssi": -67,
      "channel": 11,
      "bssid": "11:22:33:44:55:66",
      "encryption": 4
    }
  ]
}
```

**Códigos de encriptación:**
- `0` = WIFI_AUTH_OPEN (Abierta)
- `1` = WIFI_AUTH_WEP
- `2` = WIFI_AUTH_WPA_PSK
- `3` = WIFI_AUTH_WPA2_PSK
- `4` = WIFI_AUTH_WPA_WPA2_PSK
- `5` = WIFI_AUTH_WPA2_ENTERPRISE
- `6` = WIFI_AUTH_WPA3_PSK
- `7` = WIFI_AUTH_WPA2_WPA3_PSK

---

## 📊 Salida del Monitor Serial

### Ejemplo de Escaneo Completo

```
╔══════════════════════════════════════════════╗
║  ESCANEO #1                                  ║
╚══════════════════════════════════════════════╝
🕒 Fecha y hora: 22/10/2025  14:35:12
🔍 Escaneando redes WiFi...

✅ Escaneo completado en 2847 ms
📊 Redes encontradas: 12 (Total acumulado: 12)

╔═══════════════════════════════════════════════════════════════════════════╗
║ #  SSID                   RSSI  Señal  Barras     Seguridad     Canal BSSID ║
╠═══════════════════════════════════════════════════════════════════════════╣
║ 1  MiRedPrincipal         -42   🟢 95%  █████      WPA2          6    AA:BB:CC:DD:EE:FF ║
║ 2  VecinoJuan             -58   🟡 73%  ███░░      WPA2/WPA3     11   11:22:33:44:55:66 ║
║ 3  IZZI-ABCD              -65   🟡 63%  ███░░      WPA2          1    22:33:44:55:66:77 ║
║ 4  Totalplay-5678         -72   🟠 50%  ██░░░      WPA2          6    33:44:55:66:77:88 ║
║ 5  WiFi_Publico           -78   🟠 43%  ██░░░      Abierta       3    44:55:66:77:88:99 ║
║ 6  RedMovilOficina        -85   🔴 30%  █░░░░      WPA2          11   55:66:77:88:99:AA ║
╚═══════════════════════════════════════════════════════════════════════════╝

📡 Distribución de canales:
Canal │ Redes │ Gráfico
──────┼───────┼─────────────────────────────
 1    │   2   │ ▓▓
 3    │   1   │ ▓
 6    │   5   │ ▓▓▓▓▓
 11   │   4   │ ▓▓▓▓

--- Escaneo completo ---
⏱️  Próximo escaneo en 10 segundos
```

### Interpretación de Resultados

#### Columnas de la Tabla

| Columna | Descripción |
|---------|-------------|
| **#** | Número de orden (ordenado por señal) |
| **SSID** | Nombre de la red WiFi |
| **RSSI** | Intensidad de señal en dBm (-30 a -90) |
| **Señal** | Indicador visual + porcentaje de calidad |
| **Barras** | Representación gráfica (5 barras máximo) |
| **Seguridad** | Tipo de encriptación |
| **Canal** | Canal WiFi (1-13 en 2.4GHz) |
| **BSSID** | Dirección MAC del punto de acceso |

#### Valores RSSI

| Rango | Calidad | Descripción |
|-------|---------|-------------|
| **-30 a -50 dBm** | 🟢 Excelente | Señal muy fuerte, conexión óptima |
| **-50 a -60 dBm** | 🟡 Buena | Señal fuerte, buena velocidad |
| **-60 a -70 dBm** | 🟡 Regular | Señal aceptable, puede haber pérdidas |
| **-70 a -80 dBm** | 🟠 Débil | Señal baja, conexión inestable |
| **-80 a -90 dBm** | 🔴 Muy débil | Señal mínima, difícil conectar |

---

## 🛠️ Solución de Problemas

### Problema: No se sincroniza la hora NTP

**Síntomas:**
```
⚠️ No se pudo sincronizar la hora NTP
```

**Soluciones:**

1. **Verifica las credenciales WiFi**
   ```cpp
   const char* ssid_sync = "TU_RED_CORRECTA";
   const char* password_sync = "CONTRASEÑA_CORRECTA";
   ```

2. **Verifica que tu router permita conexiones**
   - Desactiva filtrado MAC temporalmente
   - Asegúrate de que DHCP esté activo

3. **Prueba con otro servidor NTP**
   ```cpp
   const char* ntpServer = "time.google.com";
   // o "time.cloudflare.com"
   ```

4. **Aumenta el tiempo de espera**
   ```cpp
   while (!getLocalTime(&timeinfo) && ntpAttempts < 20) {  // Cambia 10 a 20
   ```

---

### Problema: No detecta ninguna red

**Síntomas:**
```
❌ No se encontraron redes. Reintentando...
```

**Soluciones:**

1. **Verifica la antena del ESP32**
   - Algunos modelos requieren antena externa
   - Asegúrate de que esté bien conectada

2. **Acércate a un router WiFi**
   - El ESP32 tiene menor alcance que un smartphone

3. **Verifica el modo WiFi**
   - El código debe tener `WiFi.mode(WIFI_STA)`
   - No usar modo AP simultáneamente

4. **Reinicia el ESP32**
   - Presiona el botón RESET
   - Re-sube el código

---

### Problema: Caracteres extraños en el Monitor Serial

**Síntomas:**
```
            
```

**Solución:**

Configura el Monitor Serial a **115200 baud**:
1. Abre Monitor Serial (Ctrl+Shift+M)
2. Selecciona "115200 baud" en el menú inferior
3. Reinicia el ESP32

---

### Problema: El ESP32 se reinicia constantemente

**Síntomas:**
- Bootloop continuo
- Mensajes de "Brownout detector"

**Soluciones:**

1. **Usa un cable USB de datos**
   - No todos los cables USB transmiten datos
   - Prueba con otro cable

2. **Fuente de alimentación insuficiente**
   - Conecta a un puerto USB 2.0/3.0 directo
   - No uses hubs USB sin alimentación
   - Considera usar fuente externa de 5V/1A

3. **Reduce el consumo**
   ```cpp
   const int SCAN_INTERVAL_MS = 15000;  // Aumenta el intervalo
   ```

---

### Problema: Errores de compilación

**Error:** `WiFi.h: No such file or directory`

**Solución:**
1. Instala el soporte para ESP32 en Arduino IDE
2. Selecciona una placa ESP32 en Herramientas → Placa
3. Reinicia Arduino IDE

**Error:** `'WIFI_AUTH_WPA3_PSK' was not declared`

**Solución:**
- Actualiza el paquete ESP32 a la última versión
- Ve a Herramientas → Placa → Gestor de Tarjetas
- Busca "esp32" y actualiza

---

## 📈 Casos de Uso

### 1. Análisis de Redes en el Hogar

**Objetivo:** Optimizar tu red WiFi doméstica

**Pasos:**
1. Ejecuta varios escaneos en diferentes ubicaciones
2. Observa el análisis de canales
3. Identifica el canal menos congestionado
4. Configura tu router en ese canal

**Ejemplo:**
```
📡 Distribución de canales:
 1    │  2   │ ▓▓           ← Poco uso
 6    │  12  │ ▓▓▓▓▓▓▓▓▓▓▓▓ ← Muy congestionado (evitar)
 11   │  5   │ ▓▓▓▓▓        ← Opción media
```
**Recomendación:** Usar canal 1

---

### 2. Auditoría de Seguridad WiFi

**Objetivo:** Identificar redes inseguras

**Configuración:**
```cpp
const bool SHOW_DUPLICATES = false;  // Solo SSIDs únicos
const int MIN_RSSI_FILTER = -70;     // Solo redes cercanas
```

**Buscar:**
- Redes con seguridad "Abierta" 🔓
- Redes con WEP (obsoleto e inseguro)
- Redes duplicadas (posibles rogue APs)

---

### 3. Site Survey Profesional

**Objetivo:** Mapeo de cobertura WiFi

**Configuración:**
```cpp
const int SCAN_INTERVAL_MS = 5000;   // Escaneos frecuentes
const bool EXPORT_JSON = true;       // Exportar datos
```

**Proceso:**
1. Camina por el área con el ESP32
2. Copia los datos JSON del Monitor Serial
3. Procesa los datos en Excel/Python
4. Crea mapas de cobertura

---

### 4. Monitoreo de Interferencias

**Objetivo:** Detectar dispositivos que causan interferencia

**Configuración:**
```cpp
const int SCAN_INTERVAL_MS = 3000;   // Escaneo rápido
const bool SHOW_DUPLICATES = true;   // Ver todos los APs
```

**Observar:**
- Aparición/desaparición súbita de redes
- Cambios bruscos en RSSI
- Canales con muchas redes simultáneas

---

## 🔬 Datos Técnicos

### Especificaciones del Escaneo

| Parámetro | Valor |
|-----------|-------|
| **Tiempo de escaneo típico** | 2-4 segundos |
| **Redes máximas detectables** | Limitado por memoria RAM (~50-100) |
| **Precisión RSSI** | ±2 dBm |
| **Canales soportados** | 1-13 (2.4 GHz) |
| **Frecuencia de escaneo** | Configurable (default 10s) |

### Consumo Energético

| Fase | Consumo |
|------|---------|
| **Idle** | ~80 mA |
| **Escaneando WiFi** | ~120-160 mA |
| **Conectado WiFi (NTP)** | ~100-130 mA |

**Autonomía estimada:**
- Con batería 1000mAh: ~6-8 horas (escaneo continuo)
- Con power bank 10000mAh: ~60-80 horas

---

## 📚 Referencias

### Librerías Utilizadas

- **WiFi.h**: [Documentación oficial ESP32](https://docs.espressif.com/projects/arduino-esp32/en/latest/api/wifi.html)
- **time.h**: Librería estándar de C para manejo de tiempo
