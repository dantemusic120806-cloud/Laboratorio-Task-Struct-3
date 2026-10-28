# Laboratorio-Task-Struct-3
# Informe de Laboratorio 2: Diagnóstico Avanzado de Procesos en Linux
## Decodificando Estados del Kernel, Runqueues, Planificador CFS y Laboratorio Forense

## 1. Introducción
En los sistemas operativos modernos basados en Linux, la gestión y planificación de procesos trasciende un ciclo de vida elemental. El kernel simplifica los estados teóricos del modelo académico clásico en categorías pragmáticas y observables dentro de la topografía del sistema[cite: 25]. Comprender la telemetría del núcleo, los estados críticos de suspensión e inactividad, los procesos zombie, así como la arquitectura interna de las colas por núcleo (`runqueues`) y el motor matemático del planificador *Completely Fair Scheduler* (CFS), resulta indispensable para el diagnóstico forense y la optimización del rendimiento en entornos de producción[cite: 26, 32, 34].

---

## 2. Matriz Clínica de Estados del Kernel
El kernel de Linux unifica los procesos listos y en ejecución bajo un mismo estado operativo, expandiendo los bloqueos en matices críticos[cite: 26]:

* **R (`TASK_RUNNING`):** Proceso activo ejecutándose en la CPU o preparado en la cola de ejecución (*runqueue*)[cite: 26].
* **S (`TASK_INTERRUPTIBLE`):** Proceso en suspensión a la espera de un evento (temporizador, E/S, red). Despierta de inmediato al recibir señales del sistema[cite: 26].
* **D (`TASK_UNINTERRUPTIBLE`):** Suspensión profunda de hardware (controladores de disco o NFS mediante DMA). **Ignora señales críticas como `kill -9`** para evitar la corrupción de datos o la destrucción de búferes[cite: 26, 28].
* **T (`TASK_STOPPED`):** Proceso congelado forzosamente por una señal externa (`SIGSTOP` o `Ctrl+Z`). Solo reanuda su actividad al recibir la orden explícita `SIGCONT`[cite: 26].
* **Z (`EXIT_ZOMBIE`):** Proceso finalizado cuyos recursos básicos y tabla de control (`PID`) se retienen porque el proceso padre aún no ha recolectado su código de salida mediante la llamada `wait()`.

---

## 3. Verificación Empírica: Laboratorio Forense de Procesos Zombie (`Z`)

Para comprobar de forma práctica el comportamiento del ciclo de vida y la retención de entradas en la tabla de procesos, se implementó y ejecutó un código en C que simula un proceso padre durmiente y un proceso hijo que finaliza de inmediato sin la recolección del padre.

### Código C Implementado (`zombie.c`):
```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

int main() {
    pid_t pid = fork();
    
    if (pid == 0) {
        _exit(0); // El hijo termina inmediatamente
    }
    
    sleep(100); // El padre duerme sin hacer wait()
    return 0;
}
