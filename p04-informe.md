# Práctica 4 — “El servidor está lento”: respuestas de bitácora

## Punto 1 — La foto del antes

### Foto del antes (hora, nproc, load averages, memoria disponible)
6:52, nproc 4, 1 min: 0, 5 min: 0, 15 min: 0, mem used 251Mi swap used 0, vmstat ( si 0 so 0 bi 0 bo 0 wa 0)

### Punto 1: ¿cuál de los tres load averages dice si mejora o empeora?
> ¿Cuál de los tres números te dice si el problema está empeorando o mejorando?

foto de antes: 6:52, nproc 4, 1 min: 0, 5 min: 0, 15 min: 0, mem used 251Mi swap used 0, vmstat ( si 0 so 0 bi 0 bo 0 wa 0 in (0, 50) cs (0, 50))

revisaré si empeora o mejora visualizando lo siguiente:
comandos uptime comparando los tiempos pasados y actual + vmstat con I/O visualizacion de columma bi/bo, CPU con wa y SWAP con si/so

## Punto 4 — Aislar el recurso

### Mediciones (con hora) y recurso saturado
top -b -n 1 | head -20: PID 541 CPU: 93.8% MEM: 16.3% COMMAND indexador_catalogo
vmstat: nada raro. wa 0 si/so 0, a nivel SO se visualiza in/cs (50;100) 
free -h -s 1 -c 5: muestra que el uso de memoria pasa de 646Mi a 1.8Gi en diferentes segundos
pmap -x 541: 541:   /bin/bash /usr/local/bin/indexador-catalogo.sh
strace -p 541: visualicé llamadas al sistema y no entendí nada
logs del healthcheack:
2026-09-22 17:20:04 srv1 OK disco=3% mem=15% puerto22=escucha
2026-09-22 17:25:04 srv1 OK disco=3% mem=33% puerto22=escucha
2026-09-22 17:30:04 srv1 OK disco=3% mem=16% puerto22=escucha
2026-09-22 17:35:04 srv1 OK disco=3% mem=16% puerto22=escucha
2026-09-22 17:40:04 srv1 OK disco=3% mem=37% puerto22=escucha

ps -eo pid,ppid,user,%cpu,%mem,etime,comm --sort=-%cpu | head -10
etime (tiempo trasncurrido): desde hace 25 minutos
ppid: 1 (proceso padre es decir systemd?)

## Punto 5 — Encontrar al responsable

### Proceso responsable (PID, etime, número medido) y causa
PID    PPID USER     %CPU %MEM     ELAPSED COMMAND
    541       1 root     26.3 30.6       26:43 indexador-catal

El cuello de botella es la CPU. El proceso está calculando no esperando debido a wa 0

## Punto 6 — Intervenir, con criterio

### Punto 6: qué espero que pase antes de intervenir
Esperaba que sucediera un uso excesivo de disco que mi sistema se rentelizara completamente per la maquina virtual seguia funcionando fluidamente. Lo voy a volver a correr al escenario.sh

### Antes y después de la intervención (con hora) y si coincidió con lo esperado
Bajar la prioridad con renice no sirvió se mantuvo al 100%
load average: 0.23, 0.33, 0.18 --> load average: 0.87, 0.34, 0.15
vmstat 1 3
Mem used: 256Mi --> Mem used: 3.5Gi
vmstat wa 0 a vmstat wa 0

## Preguntas de cierre

### 1. Load average de 1 min = 4 y de 15 min = 1
> El load average de 1 minuto es 4 y el de 15 es 1. ¿El problema está empezando o terminando? ¿Y si fuera al revés?

Está terminando. Si fuera al revés el problema está ocurriendo en este momento.

### 2. renice cuando el problema es de disco
> ¿Por qué renice puede no cambiar nada cuando el problema es de disco?

Porque modifica la prioridad de uso del CPU (tiempo de procesador) y no prioridad de E/S

### 3. Permiso de reinicio: usos fuera del espíritu del ticket
> Le diste al pasante permiso para reiniciar un servicio. ¿Qué podría hacer con ese permiso que no esté en el espíritu del ticket? (Pensá mal: es el ejercicio.)

Reiniciar un servicio que no esté fallando y que sea de uso critico. Al menos el servicio que probé lo levantaba reiniciaba y levantaba rapidamente no sé como sera en otros tipos de servicios.

### 4. Healthcheck de P03: ¿lo habría detectado?
> Tu healthcheck de P03, ¿habría detectado esto? Si no, escribí la condición que le agregarías.

No porque no está midiendo CPU y con respecto a la memoria que perduró en 37% significa que a nunca cruzó el umbral en el momento de medir y tambien debido a que al ejecutar free -h -s 1 noté que el uso de memoria cambiaba de segundo a segundo. Pasar de un 6% a 37% de un proceso que no está haciendo parece que es poco y no es alarma suficiente.

### 5. Proceso que corría desde antes de las quejas
> El proceso que encontraste corría desde antes de ayer a la tarde, cuando empezaron las quejas. ¿Qué significa eso y cómo lo verificarías?

Que el proceso ya no está corriendo y verificaria los logs de los recursos de ese momento. Tambien revisaria los logs del sistema con journalctl.
