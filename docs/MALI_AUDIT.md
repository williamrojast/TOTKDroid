# Auditoría inicial: soporte Mali / PanVK

Fecha: 2026-11-12  
Repositorio: `williamrojast/TOTKDroid`  
Rama de auditoría: `audit/mali-support`  
Base: `main` en `a52423545aeddab5543b03930180dfa550510cd0`

## Estado verificado

- La rama de auditoría se creó desde `main`; no se modificó `main`.
- El parche `mali-support.patch` está en la raíz y el workflow `.github/workflows/build-mali.yml` lo aplica durante CI.
- GitHub Actions run #3 (`43325618156`) terminó correctamente. Pasaron checkout, aplicación del parche, dependencias, FFmpeg, `assembleDebug` y subida del artefacto.
- El artefacto `TOTKDroid-Mali-debug-apk` se generó (22,636,757 bytes).
- Run #1 también aparece como exitoso. La API consultada no devuelve una ejecución #2, por lo que no se atribuye aquí un resultado a esa ejecución.
- La compilación confirma que el parche compila en ese entorno; no confirma funcionamiento Vulkan ni rendimiento en el Realme GT7.

## Hallazgos críticos

### 1. El parche bloquea PanVK en Mali

En `runtime/nvn/vulkan/android_driver.c`, cualquier driver personalizado se ignora cuando no existe KGSL y el vendorID del driver del sistema no es Qualcomm. En Mali (Arm, vendorID `0x13B5`) esto impide cargar PanVK.

Además, `runtime/ui/launcher.mm` oculta el botón “Choose driver” cuando `vk_gpu_vendor() >= 2`. Aunque el cargador se corrigiera, la interfaz seguiría impidiendo seleccionar un driver personalizado en Mali.

**Corrección requerida:** distinguir explícitamente paquetes Turnip de controladores Vulkan alternativos compatibles con Mali. No permitir indiscriminadamente cualquier biblioteca; validar arquitectura, ABI, vendorID, versión Vulkan y carga de funciones. El selector debe permitir paquetes no-Turnip en Mali.

### 2. La prueba AHardwareBuffer no implementa todavía memoria de juego

La ruta nueva activa `VK_ANDROID_external_memory_android_hardware_buffer` y `VK_EXT_queue_family_foreign`, e importa AHardwareBuffer para una prueba. El propio comentario del parche dice que el juego no usa estos buffers. No hay una integración de AHB en el heap del juego ni evidencia de que esto mejore FPS.

**Conclusión:** es una prueba de diagnóstico, no una optimización de rendimiento implementada.

### 3. La prueba AHardwareBuffer puede reservar demasiada memoria

`vk_ahb_probe()` se ejecuta durante el arranque para GPU Arm cuando las extensiones están disponibles. Primero prueba 64 MiB y, si la RAM física total es de al menos 7 GiB, intenta asignar y mapear adicionalmente buffers de 512, 1024, 2048 y 3072 MiB.

El criterio usa RAM física total, no memoria disponible. Los buffers grandes se liberan uno por uno, pero cada intento puede ejercer presión fuerte sobre el sistema y provocar cierre de la app o del proceso.

**Corrección requerida:** no ejecutar por defecto; habilitarlo sólo por una opción de diagnóstico explícita, limitar el tamaño por defecto a 64–256 MiB y comprobar memoria disponible antes de cualquier prueba grande. No usar el total de RAM como sustituto de memoria libre.

### 4. Secuencia DMA-BUF de sincronización incorrecta

En la prueba AHB, el código llama primero a `DMA_BUF_IOCTL_SYNC(END | RW)` sin un `START | RW` previo; después de la operación GPU llama a `START | RW`, pero no cierra ese acceso CPU con `END | RW`.

La secuencia debe ser: START antes del acceso CPU, END antes de que la GPU acceda, esperar a la GPU, START antes de leer desde CPU y END al terminar. También deben comprobarse y registrarse los resultados de cada ioctl; actualmente los códigos se imprimen, pero no invalidan el resultado de la prueba.

### 5. Keystore de depuración dentro del parche

El parche agrega `android/mali-debug.keystore` y el workflow lo copia como `~/.android/debug.keystore`. Esto permite una firma de desarrollo estable, pero la clave privada queda distribuida públicamente en el repositorio. No debe usarse para una versión de producción. Conviene decidir entre una clave CI protegida como secreto o una clave efímera de debug, según si se necesita actualizar la misma instalación de prueba.

## Workflow de compilación

- El workflow usa `workflow_dispatch`, por lo que no se ejecuta automáticamente al hacer push.
- Aplica el parche con `git apply --check` antes de modificar el árbol de trabajo.
- Compila dependencias nativas, FFmpeg y el APK debug.
- Run #3 terminó con `BUILD SUCCESSFUL`; los warnings observados pertenecen a dependencias/archivos ajenos a los cambios Mali inspeccionados.
- El APK generado es un artefacto de CI, no una prueba de instalación o funcionamiento en el dispositivo objetivo.

## Próximos cambios propuestos

1. Corregir la política de carga de drivers para que Turnip siga restringido a Qualcomm, pero PanVK pueda seleccionarse en Mali mediante una identificación explícita del paquete.
2. Cambiar el AHB probe a opt-in y reducir el límite por defecto.
3. Corregir el protocolo de sincronización DMA-BUF y validar sus resultados.
4. Separar la prueba diagnóstica de AHB de cualquier futura integración del heap del juego.
5. Revisar el manejo de la clave de firma de desarrollo.
6. Ejecutar CI sobre esta rama y, después, probar en el Realme GT7 con posibilidad de volver al driver del sistema.

## Límite de esta auditoría

No se ha probado PanVK en el Realme GT7. Tampoco se ha demostrado que el driver PanVK pueda cargarse mediante el mecanismo actual de adrenotools. Esa compatibilidad requiere verificar el formato real del paquete y la ABI de sus bibliotecas antes de implementar el selector.
