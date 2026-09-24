# Qwen 3.8 27B Q5: 29,7 tok/s con dos GPU

**Qwen 3.8 27B Q5 en una RTX 5070 de 12 GB + una RTX 5060 Ti de 16 GB: 8.192 tokens en 4 min 36 s, con MTP y un contexto configurado de 131.072.**

![GPU](https://img.shields.io/badge/GPU-RTX%205070%20%2B%205060%20Ti-76B900?logo=nvidia&logoColor=white)
![VRAM](https://img.shields.io/badge/VRAM-12%20GB%20%2B%2016%20GB-2563EB)
![MTP](https://img.shields.io/badge/MTP-activado-7C3AED)
![llama.cpp](https://img.shields.io/badge/runtime-llama.cpp-111827)

Las líneas protagonistas del preset:

```ini
split-mode = layer
tensor-split = 12,16
spec-type = draft-mtp
```

`tensor-split` expresa una proporción. El orden debe coincidir con el que muestra `llama-server --list-devices`: primero la RTX 5070 de 12 GB y después la RTX 5060 Ti de 16 GB.

## El resultado

| Ajuste | Resultado |
| --- | ---: |
| Prompt processing | **618,63 tok/s** |
| Generación sostenida | **≈ 29,7 tok/s** |
| Salida | **8.192 tokens** |
| Duración | **4 min 36 s** |
| Contexto configurado | **131.072 tokens** |
| GPU | **RTX 5070 12 GB + RTX 5060 Ti 16 GB** |
| MTP | **Activado** |

El cociente `8192 / 276` da aproximadamente `29,68 tok/s`, coherente con los `~29,7 tok/s` observados.

> Son las cifras de la ejecución original aportadas por su autor. No se ha repetido la prueba desde este repositorio y no se conserva aquí el log bruto. El contexto configurado es la capacidad máxima, no significa que el prompt ocupara 131.072 tokens.

## Qué hace esta configuración

- **Multi-GPU:** reparte las capas entre las dos NVIDIA en proporción `12,16`.
- **MTP (Multi Token Prediction):** el propio modelo propone varios tokens futuros y verifica cuáles acepta. El GGUF debe conservar los pesos MTP.
- **Q5:** reduce el tamaño del modelo frente a precisiones superiores. Conviene registrar la variante exacta —por ejemplo `Q5_K_M`— al repetir la prueba.
- **Contexto 131K:** reserva un contexto máximo muy grande. Su coste de memoria depende también del tipo de caché KV y de la build.

MTP no se añade mágicamente con una línea del INI: si el GGUF no contiene las capas necesarias o la build no admite `draft-mtp`, no se activará.

## Qué se conoce de la prueba

| Ajuste | Valor |
| --- | --- |
| Modelo | Qwen 3.8 27B |
| Cuantización | Q5; variante exacta no registrada |
| Runtime | llama.cpp; build exacta no registrada |
| GPU 0 | NVIDIA GeForce RTX 5070, 12 GB |
| GPU 1 | NVIDIA GeForce RTX 5060 Ti, 16 GB |
| Contexto | `ctx-size = 131072` |
| Salida | 8.192 tokens |
| Reparto | Multi-GPU por capas, proporción `12,16` |
| MTP | Activado |

No se inventan los datos que faltan: versión del driver, build exacta de `llama.cpp`, nombre y SHA-256 del GGUF, CPU, RAM, caché KV y consumo de VRAM por tarjeta.

## 1. Instalar o comprobar llama.cpp

Descarga una [release oficial de llama.cpp](https://github.com/ggml-org/llama.cpp/releases) para Windows con CUDA y extrae el paquete completo, incluidas sus DLL.

En PowerShell, cambia la ruta de ejemplo por la ubicación real:

```powershell
$llamaServer = '<RUTA_A_LLAMA_CPP>\llama-server.exe'
Test-Path $llamaServer
& $llamaServer --version
& $llamaServer --list-devices
& $llamaServer --help | Select-String 'split-mode|tensor-split|draft-mtp|spec-draft-n-max|ctx-size'
```

`Test-Path` debe devolver `True`. La lista de dispositivos debe enseñar las dos GPU en el mismo orden que usa el preset.

## 2. Preparar el modelo

Necesitas un GGUF Q5 de Qwen 3.8 27B que conserve los pesos MTP. El repositorio no incluye modelos.

Abre [`configs/qwen38-27b-q5-multigpu-mtp.ini`](configs/qwen38-27b-q5-multigpu-mtp.ini) y sustituye:

```ini
m = <RUTA_AL_MODELO_GGUF_Q5>
```

por la ruta real de tu GGUF. No publiques esa ruta si contiene nombres de usuario o carpetas privadas.

Si `--list-devices` enseña primero la RTX 5060 Ti y después la RTX 5070, cambia el reparto a:

```ini
tensor-split = 16,12
```

## 3. Arrancar llama-server

Desde la carpeta del repositorio:

```powershell
& $llamaServer `
  --models-preset '.\configs\qwen38-27b-q5-multigpu-mtp.ini' `
  --models-max 1 `
  --host 127.0.0.1 `
  --port 8080
```

Abre **http://127.0.0.1:8080**, selecciona el preset `qwen3.8-27b-Q5-MULTIGPU-MTP` y espera a que cargue. Comprueba en la consola que aparecen las dos GPU y que MTP queda activo.

Si tu build no admite presets, los ajustes equivalentes más importantes son:

```powershell
& $llamaServer `
  --model '<RUTA_AL_MODELO_GGUF_Q5>' `
  --ctx-size 131072 `
  --n-gpu-layers all `
  --split-mode layer `
  --tensor-split 12,16 `
  --spec-type draft-mtp `
  --spec-draft-n-max 3 `
  --host 127.0.0.1 `
  --port 8080
```

## 4. Repetir el benchmark

1. Cierra otras aplicaciones que estén usando las GPU.
2. Arranca el servidor y confirma el reparto entre las dos tarjetas.
3. Haz una respuesta corta de calentamiento y no la contabilices.
4. Abre un chat nuevo, sin historial, herramientas, adjuntos ni instrucciones de sistema personalizadas.
5. Copia completo [`prompts/benchmark.txt`](prompts/benchmark.txt).
6. Fija el máximo de salida en **8.192 tokens** y espera a que termine.
7. Guarda `prompt eval time`, `eval time`, tokens generados, duración total y uso de VRAM por GPU.
8. Repite tres veces y usa la mediana, conservando también las tres mediciones.

No mezcles `prompt eval time` con `eval time`: el primero produce los **618,63 tok/s** de procesamiento del prompt; el segundo corresponde a la generación sostenida de **~29,7 tok/s**.

### Medir las dos GPU

En otra ventana de PowerShell:

```powershell
nvidia-smi --query-gpu=index,name,memory.used,memory.total,utilization.gpu --format=csv -l 1
```

Detén el seguimiento con `Ctrl+C`. Anota el máximo observado en cada tarjeta y no solo la suma.

### Plantilla para compartir resultados

```text
Modelo GGUF / cuantización exacta / SHA-256:
Build de llama.cpp / CUDA / driver:
CPU / RAM:
GPU 0 / VRAM usada:
GPU 1 / VRAM usada:
Contexto / caché KV / tensor-split:
Aceptación MTP:
Prompt processing: ___ tok/s
Generación: ___ / ___ / ___ tok/s; mediana ___
Tokens de salida / duración:
```

## Si algo falla

| Problema | Qué comprobar |
| --- | --- |
| Solo se usa una GPU | Revisa `--list-devices`, `split-mode`, `tensor-split` y que ambas GPU sean visibles. |
| Falta memoria | Cierra otros modelos, baja contexto o cuantiza la caché KV; documenta cualquier cambio. |
| MTP no carga | El GGUF debe conservar sus pesos MTP y la build debe reconocer `draft-mtp`. |
| Opción desconocida | Comprueba `--version` y usa una release reciente compatible. |
| Orden de GPU invertido | Cambia `12,16` por `16,12`. |
| El modelo no aparece | Revisa la ruta del GGUF y reinicia el servidor. |
| Velocidad diferente | Registra build, driver, térmicas, potencia, caché, cuantización y reparto real. |

## Archivos

```text
README.md
configs/qwen38-27b-q5-multigpu-mtp.ini
prompts/benchmark.txt
results/resultados.csv
LICENSE
```

Fuentes técnicas: [llama.cpp](https://github.com/ggml-org/llama.cpp), [multi-GPU](https://github.com/ggml-org/llama.cpp/blob/master/docs/multi-gpu.md) y [decodificación especulativa](https://github.com/ggml-org/llama.cpp/blob/master/docs/speculative.md).

**¿Tienes otra combinación de GPU? Repite la prueba y comparte el reparto, la VRAM usada y tus tok/s.**
