name: super-monitor
description: Monitor de tareas de investigación con IA con seguimiento, análisis e informes automatizados
version: 2.1.0
author: SMOUJBOT Research Team
tags:
  - research
  - ai
  - automation
  - monitoring
  - experiments
  - mlops
maintainer: ops@openclaw.io
dependencies:
  - python>=3.9
  - tensorboard>=2.12
  - pandas>=2.0
  - numpy>=1.24
  - psutil>=5.9
  - watchdog>=3.0
  - pyyaml>=6.0
  - requests>=2.31
  - rich>=13.0
  - graphene>=3.0
system_dependencies:
  - nvidia-smi (opcional, para monitoreo GPU)
  - docker (opcional, para monitoreo de contenedores)
  - prometheus-client (opcional, para exportación de métricas)
license: MIT
repository: https://github.com/smouj/openclaw-skills
documentation: https://docs.openclaw.io/skills/super-monitor
entrypoint: bin/super-monitor
```

# Super Monitor

Monitor con IA para seguimiento automatizado de tareas de investigación, análisis de rendimiento y alertas inteligentes.

## Propósito

Super Monitor automatiza la supervisión de flujos de trabajo de investigación al:
- Rastrear experimentos de ML en tiempo real con registro automático de métricas
- Monitorear utilización de recursos (CPU, GPU, memoria, disco, red)
- Detectar anomalías en curvas de entrenamiento, patrones de convergencia y picos de recursos
- Generar informes completos de experimentos con insights impulsados por IA
- Gestionar ciclo de vida de artefactos de investigación (checkpoints, logs, configs)
- Proveer dashboards en vivo para experimentos distribuidos/paralelos
- Alertar sobre fallas de hardware, fugas de memoria o degradación de rendimiento

Casos de uso reales:
1. **Investigación en Deep Learning**: Monitorear 50+ experimentos GPU concurrentes, rastrear curvas loss/accuracy, detectar patrones de sobreajuste
2. **Monitoreo de Data Pipelines**: Observar procesos ETL, detectar data drift, registrar métricas de throughput
3. **Optimización de Hiperparámetros**: Rastrear rendimiento de trials, detección de early-stopping, eficiencia de recursos
4. **Simulaciones de Larga Duración**: Monitorear progreso, intervalos de checkpoint, debug de stalls/hangs
5. **Investigación Computacional**: Rastrear salud de jobs HPC, cuellos de botella I/O, fugas de memoria en simulaciones

## Alcance

### Comandos Principales

```
super-monitor start [EXPERIMENT] [OPTIONS]
  --config FILE              Cargar config YAML (predeterminado: .super-monitor.yml)
  --track-dir DIR           Directorio a monitorear (predeterminado: .)
  --metrics METRICS         Métricas separadas por coma: loss,accuracy,f1,precision,recall
  --interval SEC            Intervalo de polling en segundos (predeterminado: 5)
  --log-level LEVEL         DEBUG, INFO, WARNING, ERROR (predeterminado: INFO)
  --alerts FILE             Reglas de alerta YAML (predeterminado: alerts.yml)
  --export-format FORMAT    json, csv, prometheus (predeterminado: json)
  --dashboard PORT          Iniciar dashboard web en puerto (predeterminado: deshabilitado)
  --container ID            Monitorear contenedor Docker por ID
  --gpu                    Habilitar monitoreo GPU (requiere nvidia-smi)
  --auto-checkpoint N       Auto-checkpoint cada N minutos

super-monitor stop [EXPERIMENT]
  --force                   Forzar detención sin shutdown graceful
  --save-artifacts          Guardar todos los artefactos antes de detener (predeterminado: true)
  --keep-days DAYS          Período de retención para artefactos (predeterminado: 30)

super-monitor status [EXPERIMENT]
  --live                    Vista de actualización en vivo (como top)
  --format FORMAT           json, table, compact (predeterminado: table)
  --filter PATTERN          Filtrar experimentos por patrón de nombre

super-monitor list
  --sort FIELD              name, start_time, duration, status
  --status STATUS           running, completed, failed, stopped
  --date DATE              Filtrar por fecha (YYYY-MM-DD)

super-monitor report [EXPERIMENT]
  --output FILE             Escribir informe a archivo (predeterminado: stdout)
  --template NAME          Usar plantilla de informe: standard, minimal, detailed
  --include-artifacts       Incluir referencias de artefactos (predeterminado: true)
  --ai-summary             Generar insights con IA (requiere OPENAI_API_KEY)

super-monitor dashboard [OPTIONS]
  --port PORT               Puerto del dashboard web (predeterminado: 8080)
  --host HOST               Dirección de enlace (predeterminado: 0.0.0.0)
  --auth CREDENTIALS        Auth básico: user:pass
  --open                    Abrir navegador automáticamente

super-monitor alerts add RULE
  --metric METRIC           Nombre de métrica a vigilar
  --condition COND          >, <, ==, !=, >=, <=
  --threshold VALUE         Valor de umbral
  --window N                Ventana de evaluación (puntos de datos)
  --cooldown MIN           Coolentre entre alertas (predeterminado: 10)
  --action CMD             Comando a ejecutar al activarse

super-monitor alerts list
  --active                  Mostrar solo alertas activas

super-monitor export [EXPERIMENT]
  --format FORMAT           json, csv, parquet, prometheus
  --output PATH             Archivo/directorio de salida
  --since TIMEFRAME         Exportar datos desde timeframe: 1h, 1d, 1w, all

super-monitor cleanup
  --older-than DAYS         Remover artefactos más antiguos que N días
  --status STATUS           Limpiar solo experimentos con este estado
  --dry-run                 Mostrar qué se eliminaría sin eliminar

super-monitor validate-config FILE
  --strict                  Fallar en claves de configuración desconocidas
```

## Proceso de Trabajo Detallado

### 1. Configuración Inicial

```bash
# Generar configuración predeterminada
super-monitor init --force

# Crea .super-monitor.yml en directorio actual
```

**.super-monitor.yml predeterminado**:
```yaml
monitor:
  name: "experiment-${USER}-${DATE}"
  track_dir: "."
  metrics: ["loss", "accuracy"]
  interval: 5
  auto_checkpoint: 30
  
logging:
  level: "INFO"
  rotate: "1 day"
  retention: "30 days"
  
alerts:
  enabled: true
  config_file: "alerts.yml"
  
export:
  format: "json"
  destination: "./experiments"
  
dashboard:
  enabled: false
  port: 8080
  
monitored_paths:
  - "**/metrics.csv"
  - "**/checkpoint_*.pth"
  - "**/logs/*.log"
  exclude:
    - "**/tmp/**"
    - "**/.git/**"

resource_thresholds:
  memory_percent: 90
  gpu_memory_percent: 95
  disk_percent: 95
  cpu_percent: 95

anomaly_detection:
  enabled: true
  sensitivity: 0.05
  min_data_points: 10
```

### 2. Iniciar una Sesión de Investigación

```bash
# Uso básico - monitorear directorio actual
super-monitor start --track-dir ./training --metrics loss,accuracy,f1 --interval 2 --dashboard

# Con monitoreo de contenedor y GPU
super-monitor start experiment-001 \
  --container $(docker ps -q -f name=training) \
  --gpu \
  --auto-checkpoint 15

# Cargar desde config
super-monitor start --config custom-monitor.yml

# Con alertas personalizadas
super-monitor start --alerts critical-alerts.yml
```

**Qué sucede**:
1. Crea ID de experimento (UUID4) y directorio de trabajo `./.super-monitor/experiments/<id>`
2. Inicializa base de datos SQLite `monitor.db` para almacenamiento de métricas
3. Inicia hilos de vigilancia en segundo plano:
   - File watcher: vigila `monitored_paths` para cambios
   - Resource poller: muestrea CPU/GPU/memoria/disco cada `interval` segundos
   - Metric parser: extrae valores numéricos de archivos de log que coinciden con `regex_patterns`
4. Carga reglas de alerta desde `alerts.yml`
5. Si `--dashboard` habilitado, inicia servidor FastAPI en puerto especificado
6. Comienza logging en vivo a `./experiments/<id>/monitor.log`

### 3. Configuración de Alertas

**alerts.yml**:
```yaml
alerts:
  - name: "High GPU Memory"
    metric: "gpu_memory_percent"
    condition: ">"
    threshold: 95
    window: 3
    cooldown: 5
    severity: "critical"
    action: "super-monitor checkpoint --emergency"
    
  - name: "Loss Not Decreasing"
    metric: "loss"
    condition: ">"
    threshold: 0.1
    window: 20
    cooldown: 30
    severity: "warning"
    action: "echo 'Potential convergence issue' | mail -s 'Alert' researcher@lab.edu"
    
  - name: "Stalled Training"
    metric: "training_step"
    condition: "=="
    threshold:  # Comparar contra valor anterior
    window: 10
    cooldown: 60
    severity: "critical"
    action: "super-monitor kill --reason='stalled'"
```

### 4. Dashboard en Vivo

Acceder `http://localhost:8080`:
- Gráficos de métricas en tiempo real (Ventana móvil: últimos 1000 puntos)
- Gauges de utilización de recursos
- Lista de experimentos con indicadores de estado
- Historial de alertas
- Botones de descarga CSV/JSON

Autenticación:
```
http://user:password@localhost:8080
```

### 5. Ciclo de Vida de Experimentos

```bash
# Ver estado durante ejecución
super-monitor status --live
# Salida:
# EXP-ID           STATUS    DURATION  GPU%  MEM%   LOSS    ACC
# abc123           RUNNING   2h 15m    87    64     0.234   0.89
# def456           COMPLETED 4h 03m    -     -      0.012   0.97

# Generar resumen con IA
OPENAI_API_KEY=sk-... super-monitor report experiment-001 --ai-summary
# Salida:
# ===== AI Summary =====
# - Training converged successfully after 15,000 steps
# - Learning rate schedule effectively prevented overfitting
# - GPU utilization averaged 78% - consider batch size increase
# - Memory leak detected after 12h (increase 2MB every 1000 steps)
# Recommendations:
# 1. Increase batch size from 64 to 128 for 15% faster training
# 2. Investigate memory leak in data loader
# 3. Current checkpoint frequency optimal

# Exportar datos para análisis
super-monitor export experiment-001 --format csv --output ./analysis/data.csv --since 1d

# Detener y archivar gracefulmente
super-monitor stop experiment-001
# Crea: experiments/abc123/artifacts/checkpoints/, logs/, metrics.json
```

### 6. Checkpoints Automatizados

```bash
# Configurar auto-checkpoint en .super-monitor.yml:
auto_checkpoint: 30  # Cada 30 minutos

# Al activarse checkpoint:
# 1. Guarda estado actual del modelo en experiments/<id>/artifacts/checkpoints/checkpoint_<timestamp>.pth
# 2. Comprime logs más antiguos de 1h a .gz
# 3. Sube a S3 si s3_bucket configurado:
#    aws s3 cp experiments/<id>/artifacts/ s3://bucket/experiments/<id>/ --recursive
```

### 7. Integración con Scripts de Entrenamiento

```python
# En tu código de entrenamiento (ejemplo PyTorch):
import json
from super_monitor import SuperMonitor

monitor = SuperMonitor(experiment_id="auto-generated if None")
monitor.log_metric("loss", loss.item(), step=global_step)
monitor.log_metric("accuracy", acc, step=global_step)
monitor.log_artifact("model", model.state_dict(), step=global_step)

# O usar variables de entorno (sin cambios de código):
# export SUPER_MONITOR_EXPERIMENT_ID=abc123
# export SUPER_MONITOR_LOG_METRICS=loss,accuracy,f1
# El programa escribe métricas a stdout como líneas "metric=value":
# [Train] loss=0.423 accuracy=0.78 step=1000
```

## Reglas de Oro

1. **Umbrales de Recursos**: Configurar `memory_percent` en 85% (no 90%) para permitir 5 minutos de tiempo de reacción antes de OOM
2. **Selección de Intervalo**: Usar intervalos 1-2s para experimentos cortos (<1h), 5-10s para experimentos largos (>24h)
3. **Coolentre de Alertas**: Siempre establecer cooldown ≥ 2× ventana de evaluación para prevenir tormentas de alertas
4. **Frecuencia de Checkpoint**: Auto-checkpoint cada 30min o 10% del tiempo total esperado de ejecución, lo que sea menor
5. **Seguridad del Dashboard**: Siempre habilitar `--auth` si dashboard expuesto más allá de localhost
6. **Política de Retención**: Ejecutar `super-monitor cleanup --older-than 30` via cron diariamente
7. **Nomenclatura de Métricas**: Usar lowercase_with_underscores; nunca cambiar nombres de métricas a mitad del experimento
8. **Monitoreo de Recursos**: Siempre habilitar monitoreo GPU para experimentos de ML; es el indicador de fallo más temprano
9. **Detección de Anomalías**: Configurar `min_data_points: 20` mínimo; falsos positivos altos con <10 puntos
10. **Formato de Exportación**: Usar `parquet` para datasets >1GB; `csv` para compatibilidad; `prometheus` para integración con Grafana

## Ejemplos

### Ejemplo 1: Monitoreo de Entrenamiento Distribuido

```bash
# Lanzar entrenamiento de 4 nodos con monitoreo coordinado
super-monitor start distributed-bert \
  --config bert-config.yml \
  --dashboard \
  --export-format prometheus

# bert-config.yml:
master_node: "node01"
worker_nodes: ["node02", "node03", "node04"]
metrics:
  - "train_loss"
  - "eval_accuracy"
  - "gradient_norm"
resource_aggregation: "average"  # Agregar entre nodos

# En cada script de nodo:
export SUPER_MONITOR_NODE_ID=$(hostname)
export SUPER_MONITOR_MASTER=node01
# Métricas se agregan automáticamente en dashboard
```

### Ejemplo 2: Alertas Automatizadas para Ejecuciones Nocturnas

```bash
# alerts-night.yml:
alerts:
  - name: "Night Run Complete"
    metric: "training_complete"
    condition: "=="
    threshold: 1
    window: 1
    action: "curl -X POST https://hooks.slack.com/... -d '{\"text\":\"Experiment done\"}'"
  
  - name: "Power Outage Detected"
    metric: "last_update_seconds"
    condition: ">"
    threshold: 600  # 10 minutos
    window: 1
    action: "super-monitor stop --force; aws s3 sync artifacts/ s3://backup/"

super-monitor start night-experiment \
  --alerts alerts-night.yml \
  --interval 60 \
  --log-level WARNING
```

### Ejemplo 3: Integración con CI/CD

```yaml
# .github/workflows/train.yml:
- name: Start monitoring
  run: |
    super-monitor start ci-run-${{ github.run_id }} \
      --track-dir . \
      --metrics loss,accuracy \
      --interval 2
    echo "MONITOR_ID=$(super-monitor list --json | jq -r '.[0].id')" >> $GITHUB_ENV
    
- name: Run training
  run: python train.py --epochs 10
  
- name: Generate report
  if: always()
  run: |
    super-monitor report $MONITOR_ID --output report.md
    super-monitor export $MONITOR_ID --format json --output metrics.json
    super-monitor stop $MONITOR_ID
```

### Ejemplo 4: Monitoreo en Cluster HPC

```bash
# Script de job SLURM:
#!/bin/bash
#SBATCH --gres=gpu:4
#SBATCH --output=%x-%j.log

module load python/3.9
super-monitor start hpc-run-${SLURM_JOB_ID} \
  --gpu \
  --metrics train_loss,val_loss,learning_rate \
  --interval 1 \
  --auto-checkpoint 60 \
  --export-format parquet

python distributed_train.py --nodes $SLURM_NNODES --gpus-per-node 4

# Colectar automáticamente al completar:
super-monitor report hpc-run-${SLURM_JOB_ID} \
  --output results/${SLURM_JOB_ID}.md \
  --ai-summary
```

## Comandos de Rollback

### Intervenciones de Experimentos

```bash
# 1. Detener experimento desbordado (data leak, OOM)
super-monitor stop experiment-id --force
# Rollback: Restaurar desde último checkpoint
super-monitor restore experiment-id --checkpoint latest

# 2. Deshabilitar alerta problemática
super-monitor alerts disable "High GPU Memory"
# Rollback: Re-habilitar
super-monitor alerts enable "High GPU Memory"

# 3. Revertir config de metric tracking (causando alto CPU)
# Editar .super-monitor.yml, cambiar interval de 1s a 10s
# Rollback: Restaurar config anterior desde backup:
cp .super-monitor.yml.bak .super-monitor.yml
super-monitor restart experiment-id

# 4. Liberar espacio en disco reduciendo retención
super-monitor cleanup --older-than 7 --dry-run  # Preview
super-monitor cleanup --older-than 7           # Execute
# Rollback: Si se eliminó artefacto necesario, restaurar desde S3:
aws s3 cp s3://bucket/experiments/id/artifacts/ ./restore/ --recursive

# 5. Detener exposición del dashboard
# Si se inició con --auth faltante, inmediatamente:
super-monitor dashboard --stop
# O matar proceso: pkill -f "super-monitor dashboard"
# Rollback: Reiniciar con auth:
super-monitor dashboard --auth admin:$(openssl rand -base64 12)

# 6. Recuperar de corrupción de base de datos
# .super-monitor/experiments/<id>/monitor.db corrupto
super-monitor recover-db experiment-id --from-metrics ./backup_metrics.json
# Rollback: Si recovery falla, usar backup anterior:
cp .super-monitor/experiments/<id>/monitor.db.bak .super-monitor/experiments/<id>/monitor.db

# 7. Detener todo monitoreo (emergencia)
super-monitor stop-all --force
# Rollback: Revisar experiments/, reiniciar selectivamente los necesarios
super-monitor start experiment-id --config backup-config.yml

# 8. Deshabilitar resumen IA (control de costos)
unset OPENAI_API_KEY
# O editar .super-monitor.yml: ai_summary: false
# Rollback: Re-exportar variable, set a true
```

### Rollbacks de Configuración

```bash
# Mantener historial de config:
super-monitor config snapshot --message "Increased thresholds"
# Lista: .super-monitor/configs/20260101_120000.yml

# Rollback a anterior:
super-monitor config apply 20260101_120000.yml

# O manual:
cp .super-monitor.yml.backup .super-monitor.yml
super-monitor validate-config .super-monitor.yml
```

## Verificación

Después de cualquier modificación:

```bash
# 1. Validar sintaxis de configuración
super-monitor validate-config .super-monitor.yml

# 2. Verificar monitoreo activo
super-monitor status | grep RUNNING

# 3. Verificar métricas siendo capturadas
super-monitor tail experiment-id --lines 10
# Debería mostrar: "INFO Recorded metric loss=0.423 step=1000"

# 4. Probar alertas
super-monitor alerts test --rule "High GPU Memory" --value 96

# 5. Probar acceso a dashboard
curl -u user:pass http://localhost:8080/api/health
# Esperado: {"status":"ok","experiments":X}

# 6. Verificar escritura de artefactos
ls -la .super-monitor/experiments/<id>/artifacts/
# Debería contener: checkpoints/, logs/, metrics/

# 7. Precisión de monitoreo de recursos
super-monitor status experiment-id --format json | jq '.gpu_usage'
# Comparar con: nvidia-smi --query-gpu=utilization.gpu --format=csv
# Valores deberían coincidir dentro de 2% (latencia intervalo polling)

# 8. Confirmar políticas de limpieza efectivas
find .super-monitor/experiments/ -mtime +30 | wc -l
# Debería ser 0 si limpieza corrió
```

## Solución de Problemas

### Síntoma: Alto uso de CPU (super-monitor > 50%)
**Causa**: Regex de parsing de métricas muy agresivo o intervalo muy corto
**Fix**:
1. Incrementar `--interval` a 10s
2. Simplificar regex de métricas en config: `metric_patterns: ["loss:\s*(\d+\.?\d*)"]` (evitar lookaheads complejos)
3. Excluir archivos de log ruidosos: agregar a patrones `exclude`

### Síntoma: Métricas no aparecen en dashboard
**Causa**: File watcher con patrones incorrectos
**Debug**:
```bash
super-monitor diagnose --experiment-id <id> --issue "missing_metrics"
# Outputs:
# - Watched paths: ./logs/**/*.log
# - Recent file events: 0 in last 5m
# - Log tail: muestra si hay errores de parser
```
**Fix**: Actualizar `monitored_paths` en config para incluir ubicaciones reales de archivos de log

### Síntoma: Alertas de umbral de memoria GPU falsos positivos
**Causa**: Formato de salida de nvidia-smi cambió (actualización de driver)
**Fix**:
```bash
nvidia-smi --query-gpu=memory.used,memory.total --format=csv
# Si headers cambiaron, actualizar parser en config:
gpu_memory_pattern: "(\d+) MiB / (\d+) MiB"
```

### Síntoma: Experimentos se acumulan, disco lleno
**Causa**: Política de retención no aplicada
**Fix**:
```bash
# Habilitar limpieza via cron:
0 2 * * * super-monitor cleanup --older-than 30 --status completed
# O set en config:
retention:
  completed: "7 days"
  failed: "30 days"
  running: "0 days"  # No auto-borrar running
```

### Síntoma: Acciones de alerta no se disparan
**Causa**: Comando de acción no ejecutable o entorno diferente
**Debug**:
```bash
super-monitor alerts test --rule "Loss Not Decreasing" --dry-run
# Muestra comando exacto a ejecutar
```
**Fix**: Usar paths absolutos en acciones; source entorno:
```yaml
action: ". /home/user/.bashrc && python /path/to/alert.py"
```

### Síntoma: Dashboard muestra "Disconnected"
**Causa**: Timeout de WebSocket detrás de reverse proxy
**Fix**: Incrementar timeout en config Nginx/Caddy:
```
proxy_read_timeout 3600s;
proxy_send_timeout 3600s;
```
O deshabilitar WebSocket, usar polling: `dashboard: { polling_interval: 5 }`

### Síntoma: `super-monitor start` se cuelga
**Causa**: Base de datos bloqueada por instancia previa que falló
**Fix**:
```bash
rm -f .super-monitor/experiments/<id>/monitor.db.lock
# O forzar reset:
super-monitor reset --experiment <id> --force
```

### Síntoma: Export falla con error de memoria
**Causa**: Dataset grande (multi-GB) cargado en memoria
**Fix**: Usar export streaming:
```bash
super-monitor export experiment-id \
  --format parquet \
  --output - | gzip > data.parquet.gz
# O exportar en chunks:
super-monitor export --since 1d --until 12h
super-monitor export --since 12h --until now
```

## Ajuste de Rendimiento

- **Huella de memoria**: ~50MB base + 2MB por flujo de métrica monitoreada
- **CPU**: ~1% por directorio monitoreado (con intervalo 5s)
- **Disk I/O**: ~100KB/s para 10 métricas a 1s interval (JSON append)
- **Red**: Dashboard ~5KB/s por cliente (WebSocket push)

Para 1000+ experimentos concurrentes en nodo único:
```yaml
monitor:
  interval: 30  # Reducir sampling
  metrics: ["loss"]  # Set mínimo
dashboard:
  enabled: false  # Deshabilitar para batch runs
logging:
  rotate: "6 hours"
  retention: "7 days"
```

## Consideraciones de Seguridad

- Dashboard: Siempre usar `--auth` si no es localhost
- Archivos de config: `chmod 600 .super-monitor.yml` (contiene paths, potencialmente sensible)
- Base de datos: Archivos SQLite en `.super-monitor/` deberían ser legibles solo por owner
- Acciones de alerta: Validar comandos; YAML suministrado por usuario podría inyectar comandos
- Subidas S3: Usar roles IAM, no credenciales root; bucket policies restringir a prefijo de experimento
- Tokens de API: Nunca commit `OPENAI_API_KEY` a repo; usar entorno o HashiCorp Vault

## Soporte

Issues: https://github.com/smouj/openclaw-skills/issues?label=super-monitor
Docs: https://docs.openclaw.io/skills/super-monitor
```