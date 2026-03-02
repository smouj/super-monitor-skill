---
name: super-monitor
description: AI-powered research task monitor with automated tracking, analysis, and reporting
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
  - nvidia-smi (optional, for GPU monitoring)
  - docker (optional, for container monitoring)
  - prometheus-client (optional, for metrics export)
license: MIT
repository: https://github.com/smouj/openclaw-skills
documentation: https://docs.openclaw.io/skills/super-monitor
entrypoint: bin/super-monitor
---

# Super Monitor

AI-powered monitor for automated research task tracking, performance analysis, and intelligent alerting.

## Purpose

Super Monitor automates research workflow oversight by:
- Tracking ML experiments in real-time with automatic metric logging
- Monitoring resource utilization (CPU, GPU, memory, disk, network)
- Detecting anomalies in training curves, convergence patterns, and resource spikes
- Generating comprehensive experiment reports with AI-powered insights
- Managing research artifact lifecycle (checkpoints, logs, configs)
- Providing live dashboards for distributed/parallel experiments
- Alerting on hardware failures, memory leaks, or performance degradation

Real use cases:
1. **Deep Learning Research**: Monitor 50+ concurrent GPU experiments, track loss/accuracy curves, detect overfitting patterns
2. **Data Pipeline Monitoring**: Watch ETL processes, detect data drift, log throughput metrics
3. **Hyperparameter Optimization**: Track trial performance, early-stopping detection, resource efficiency
4. **Long-running Simulations**: Monitor progress, checkpoint intervals, debug stalls/hangs
5. **Computational Research**: Track HPC job health, I/O bottlenecks, memory leaks in simulations

## Scope

### Core Commands

```
super-monitor start [EXPERIMENT] [OPTIONS]
  --config FILE              Load YAML config (default: .super-monitor.yml)
  --track-dir DIR           Directory to monitor (default: .)
  --metrics METRICS         Comma-separated metrics: loss,accuracy,f1,precision,recall
  --interval SEC            Polling interval in seconds (default: 5)
  --log-level LEVEL         DEBUG, INFO, WARNING, ERROR (default: INFO)
  --alerts FILE             Alert rules YAML (default: alerts.yml)
  --export-format FORMAT    json, csv, prometheus (default: json)
  --dashboard PORT          Start web dashboard on port (default: disabled)
  --container ID            Monitor Docker container by ID
  --gpu                    Enable GPU monitoring (requires nvidia-smi)
  --auto-checkpoint N       Auto-checkpoint every N minutes

super-monitor stop [EXPERIMENT]
  --force                   Force stop without graceful shutdown
  --save-artifacts          Save all artifacts before stopping (default: true)
  --keep-days DAYS          Retention period for artifacts (default: 30)

super-monitor status [EXPERIMENT]
  --live                    Live updating view (like top)
  --format FORMAT           json, table, compact (default: table)
  --filter PATTERN          Filter experiments by name pattern

super-monitor list
  --sort FIELD              name, start_time, duration, status
  --status STATUS           running, completed, failed, stopped
  --date DATE              Filter by date (YYYY-MM-DD)

super-monitor report [EXPERIMENT]
  --output FILE             Write report to file (default: stdout)
  --template NAME          Use report template: standard, minimal, detailed
  --include-artifacts       Include artifact references (default: true)
  --ai-summary             Generate AI insights (requires OPENAI_API_KEY)

super-monitor dashboard [OPTIONS]
  --port PORT               Web dashboard port (default: 8080)
  --host HOST               Bind address (default: 0.0.0.0)
  --auth CREDENTIALS        Basic auth: user:pass
  --open                    Open browser automatically

super-monitor alerts add RULE
  --metric METRIC           Metric name to watch
  --condition COND          >, <, ==, !=, >=, <=
  --threshold VALUE         Threshold value
  --window N                Evaluation window (data points)
  --cooldown MIN           Cooldown between alerts (default: 10)
  --action CMD             Command to execute on trigger

super-monitor alerts list
  --active                  Show only active alerts

super-monitor export [EXPERIMENT]
  --format FORMAT           json, csv, parquet, prometheus
  --output PATH             Output file/directory
  --since TIMEFRAME         Export data from timeframe: 1h, 1d, 1w, all

super-monitor cleanup
  --older-than DAYS         Remove artifacts older than N days
  --status STATUS           Clean only experiments with this status
  --dry-run                 Show what would be deleted without deleting

super-monitor validate-config FILE
  --strict                  Fail on unknown configuration keys
```

## Detailed Work Process

### 1. Initial Setup

```bash
# Generate default configuration
super-monitor init --force

# Creates .super-monitor.yml in current directory
```

**Default .super-monitor.yml**:
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

### 2. Starting a Research Session

```bash
# Basic usage - monitor current directory
super-monitor start --track-dir ./training --metrics loss,accuracy,f1 --interval 2 --dashboard

# With container and GPU monitoring
super-monitor start experiment-001 \
  --container $(docker ps -q -f name=training) \
  --gpu \
  --auto-checkpoint 15

# Load from config
super-monitor start --config custom-monitor.yml

# With custom alerts
super-monitor start --alerts critical-alerts.yml
```

**What happens**:
1. Creates experiment ID (UUID4) and working directory `./.super-monitor/experiments/<id>`
2. Initializes SQLite database `monitor.db` for metrics storage
3. Starts background watcher threads:
   - File watcher: watches `monitored_paths` for changes
   - Resource poller: samples CPU/GPU/memory/disk every `interval` seconds
   - Metric parser: extracts numeric values from log files matching `regex_patterns`
4. Loads alert rules from `alerts.yml`
5. If `--dashboard` enabled, starts FastAPI server on specified port
6. Begins live logging to `./experiments/<id>/monitor.log`

### 3. Alerts Configuration

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
    threshold:  # Compare against previous value
    window: 10
    cooldown: 60
    severity: "critical"
    action: "super-monitor kill --reason='stalled'"
```

### 4. Live Dashboard

Access `http://localhost:8080`:
- Real-time metric charts (Rolling window: last 1000 points)
- Resource utilization gauges
- Experiment list with status indicators
- Alert history log
- Download CSV/JSON buttons

Authentication:
```
http://user:password@localhost:8080
```

### 5. Experiment Lifecycle

```bash
# Check status during run
super-monitor status --live
# Output:
# EXP-ID           STATUS    DURATION  GPU%  MEM%   LOSS    ACC
# abc123           RUNNING   2h 15m    87    64     0.234   0.89
# def456           COMPLETED 4h 03m    -     -      0.012   0.97

# Generate AI-powered summary
OPENAI_API_KEY=sk-... super-monitor report experiment-001 --ai-summary
# Output:
# ===== AI Summary =====
# - Training converged successfully after 15,000 steps
# - Learning rate schedule effectively prevented overfitting
# - GPU utilization averaged 78% - consider batch size increase
# - Memory leak detected after 12h (increase 2MB every 1000 steps)
# Recommendations:
# 1. Increase batch size from 64 to 128 for 15% faster training
# 2. Investigate memory leak in data loader
# 3. Current checkpoint frequency optimal

# Export data for analysis
super-monitor export experiment-001 --format csv --output ./analysis/data.csv --since 1d

# Gracefully stop and archive
super-monitor stop experiment-001
# Creates: experiments/abc123/artifacts/checkpoints/, logs/, metrics.json
```

### 6. Automated Checkpoints

```bash
# Configure auto-checkpoint in .super-monitor.yml:
auto_checkpoint: 30  # Every 30 minutes

# On checkpoint trigger:
# 1. Saves current model state to experiments/<id>/artifacts/checkpoints/checkpoint_<timestamp>.pth
# 2. Compresses logs older than 1h to .gz
# 3. Uploads to S3 if s3_bucket configured:
#    aws s3 cp experiments/<id>/artifacts/ s3://bucket/experiments/<id>/ --recursive
```

### 7. Integration with Training Scripts

```python
# In your training code (PyTorch example):
import json
from super_monitor import SuperMonitor

monitor = SuperMonitor(experiment_id="auto-generated if None")
monitor.log_metric("loss", loss.item(), step=global_step)
monitor.log_metric("accuracy", acc, step=global_step)
monitor.log_artifact("model", model.state_dict(), step=global_step)

# Or use environment variables (no code changes):
# export SUPER_MONITOR_EXPERIMENT_ID=abc123
# export SUPER_MONITOR_LOG_METRICS=loss,accuracy,f1
# Program writes metrics to stdout as "metric=value" lines:
# [Train] loss=0.423 accuracy=0.78 step=1000
```

## Golden Rules

1. **Resource Thresholds**: Set `memory_percent` threshold at 85% (not 90%) to allow 5-minute reaction time before OOM
2. **Interval Selection**: Use 1-2s intervals for short experiments (<1h), 5-10s for long experiments (>24h)
3. **Alert Cooldown**: Always set cooldown ≥ 2× evaluation window to prevent alert storms
4. **Checkpoint Frequency**: Auto-checkpoint every 30min or 10% of total expected runtime, whichever is lower
5. **Dashboard Security**: Always enable `--auth` if dashboard exposed beyond localhost
6. **Retention Policy**: Run `super-monitor cleanup --older-than 30` via cron daily
7. **Metric Naming**: Use lowercase_with_underscores; never change metric names mid-experiment
8. **Resource Monitoring**: Always enable GPU monitoring for ML experiments; it's the earliest failure indicator
9. **Anomaly Detection**: Set `min_data_points: 20` minimum; false positives high with <10 points
10. **Export Format**: Use `parquet` for >1GB datasets; `csv` for compatibility; `prometheus` for Grafana integration

## Examples

### Example 1: Monitoring Distributed Training

```bash
# Launch 4-node training with coordinated monitoring
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
resource_aggregation: "average"  # Aggregate across nodes

# In each node's training script:
export SUPER_MONITOR_NODE_ID=$(hostname)
export SUPER_MONITOR_MASTER=node01
# Metrics auto-aggregated in dashboard
```

### Example 2: Automated Alerting for Night Runs

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
    threshold: 600  # 10 minutes
    window: 1
    action: "super-monitor stop --force; aws s3 sync artifacts/ s3://backup/"

super-monitor start night-experiment \
  --alerts alerts-night.yml \
  --interval 60 \
  --log-level WARNING
```

### Example 3: Integration with CI/CD

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

### Example 4: HPC Cluster Monitoring

```bash
# SLURM job script:
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

# Auto-collect on completion:
super-monitor report hpc-run-${SLURM_JOB_ID} \
  --output results/${SLURM_JOB_ID}.md \
  --ai-summary
```

## Rollback Commands

### Experiment Interventions

```bash
# 1. Stop a runaway experiment (data leak, OOM)
super-monitor stop experiment-id --force
# Rollback: Restore from last checkpoint
super-monitor restore experiment-id --checkpoint latest

# 2. Disable problematic alert
super-monitor alerts disable "High GPU Memory"
# Rollback: Re-enable
super-monitor alerts enable "High GPU Memory"

# 3. Revert metric tracking config (causing high CPU)
# Edit .super-monitor.yml, change interval from 1s to 10s
# Rollback: Restore previous config from backup:
cp .super-monitor.yml.bak .super-monitor.yml
super-monitor restart experiment-id

# 4. Free disk space by reducing retention
super-monitor cleanup --older-than 7 --dry-run  # Preview
super-monitor cleanup --older-than 7           # Execute
# Rollback: If deleted needed artifact, restore from S3:
aws s3 cp s3://bucket/experiments/id/artifacts/ ./restore/ --recursive

# 5. Stop dashboard exposure
# If started with --auth missing, immediately:
super-monitor dashboard --stop
# Or kill process: pkill -f "super-monitor dashboard"
# Rollback: Restart with auth:
super-monitor dashboard --auth admin:$(openssl rand -base64 12)

# 6. Recover from database corruption
# .super-monitor/experiments/<id>/monitor.db corrupted
super-monitor recover-db experiment-id --from-metrics ./backup_metrics.json
# Rollback: If recovery fails, use previous backup:
cp .super-monitor/experiments/<id>/monitor.db.bak .super-monitor/experiments/<id>/monitor.db

# 7. Stop all monitoring (emergency)
super-monitor stop-all --force
# Rollback: Review experiments/, selectively restart needed ones
super-monitor start experiment-id --config backup-config.yml

# 8. Disable AI summary (cost containment)
unset OPENAI_API_KEY
# Or edit .super-monitor.yml: ai_summary: false
# Rollback: Re-export variable, set to true
```

### Configuration Rollbacks

```bash
# Keep config history:
super-monitor config snapshot --message "Increased thresholds"
# Lists: .super-monitor/configs/20260101_120000.yml

# Rollback to previous:
super-monitor config apply 20260101_120000.yml

# Or manual:
cp .super-monitor.yml.backup .super-monitor.yml
super-monitor validate-config .super-monitor.yml
```

## Verification

After any modification:

```bash
# 1. Validate configuration syntax
super-monitor validate-config .super-monitor.yml

# 2. Check monitoring is active
super-monitor status | grep RUNNING

# 3. Verify metrics being captured
super-monitor tail experiment-id --lines 10
# Should show: "INFO Recorded metric loss=0.423 step=1000"

# 4. Test alerts
super-monitor alerts test --rule "High GPU Memory" --value 96

# 5. Test dashboard access
curl -u user:pass http://localhost:8080/api/health
# Expected: {"status":"ok","experiments":X}

# 6. Verify artifacts writing
ls -la .super-monitor/experiments/<id>/artifacts/
# Should contain: checkpoints/, logs/, metrics/

# 7. Resource monitoring accuracy
super-monitor status experiment-id --format json | jq '.gpu_usage'
# Compare with: nvidia-smi --query-gpu=utilization.gpu --format=csv
# Values should match within 2% (polling interval latency)

# 8. Confirm cleanup policies effective
find .super-monitor/experiments/ -mtime +30 | wc -l
# Should be 0 if cleanup ran
```

## Troubleshooting

### Symptom: High CPU usage (super-monitor > 50%)
**Cause**: Aggressive metric parsing regex or short interval
**Fix**:
1. Increase `--interval` to 10s
2. Simplify metric regex in config: `metric_patterns: ["loss:\\s*(\\d+\\.?\\d*)"]` (avoid complex lookaheads)
3. Exclude noisy log files: add to `exclude` patterns

### Symptom: Metrics not appearing in dashboard
**Cause**: File watcher missing patterns
**Debug**:
```bash
super-monitor diagnose --experiment-id <id> --issue "missing_metrics"
# Outputs:
# - Watched paths: ./logs/**/*.log
# - Recent file events: 0 in last 5m
# - Log tail: shows if parser errors
```
**Fix**: Update `monitored_paths` in config to include actual log file locations

### Symptom: GPU memory threshold alerts false positive
**Cause**: nvidia-smi output format changed (driver update)
**Fix**:
```bash
nvidia-smi --query-gpu=memory.used,memory.total --format=csv
# If headers changed, update parser in config:
gpu_memory_pattern: "(\d+) MiB / (\d+) MiB"
```

### Symptom: Experiments accumulate, disk full
**Cause**: Retention policy not enforced
**Fix**:
```bash
# Enable cleanup via cron:
0 2 * * * super-monitor cleanup --older-than 30 --status completed
# Or set in config:
retention:
  completed: "7 days"
  failed: "30 days"
  running: "0 days"  # Don't auto-delete running
```

### Symptom: Alert actions not firing
**Cause**: Action command not executable or environment differs
**Debug**:
```bash
super-monitor alerts test --rule "Loss Not Decreasing" --dry-run
# Shows exact command to be executed
```
**Fix**: Use absolute paths in actions; source environment:
```yaml
action: ". /home/user/.bashrc && python /path/to/alert.py"
```

### Symptom: Dashboard shows "Disconnected"
**Cause**: WebSocket timeout behind reverse proxy
**Fix**: Increase timeout in Nginx/Caddy config:
```
proxy_read_timeout 3600s;
proxy_send_timeout 3600s;
```
Or disable WebSocket, use polling: `dashboard: { polling_interval: 5 }`

### Symptom: `super-monitor start` hangs
**Cause**: Database locked from previous crashed instance
**Fix**:
```bash
rm -f .super-monitor/experiments/<id>/monitor.db.lock
# Or force reset:
super-monitor reset --experiment <id> --force
```

### Symptom: Export fails with memory error
**Cause**: Large dataset (multi-GB) loaded into memory
**Fix**: Use streaming export:
```bash
super-monitor export experiment-id \
  --format parquet \
  --output - | gzip > data.parquet.gz
# Or export in chunks:
super-monitor export --since 1d --until 12h
super-monitor export --since 12h --until now
```

## Performance Tuning

- **Memory footprint**: ~50MB base + 2MB per monitored metric stream
- **CPU**: ~1% per monitored directory (with 5s interval)
- **Disk I/O**: ~100KB/s for 10 metrics at 1s interval (JSON append)
- **Network**: Dashboard ~5KB/s per client (WebSocket push)

For 1000+ concurrent experiments on single node:
```yaml
monitor:
  interval: 30  # Reduce sampling
  metrics: ["loss"]  # Minimal set
dashboard:
  enabled: false  # Disable for batch runs
logging:
  rotate: "6 hours"
  retention: "7 days"
```

## Security Considerations

- Dashboard: Always use `--auth` if not localhost
- Config files: `chmod 600 .super-monitor.yml` (contains paths, potentially sensitive)
- Database: SQLite files in `.super-monitor/` should be readable only by owner
- Alert actions: Validate commands; user-supplied YAML could inject commands
- S3 uploads: Use IAM roles, not root credentials; bucket policies restrict to experiment prefix
- API tokens: Never commit `OPENAI_API_KEY` to repo; use environment or HashiCorp Vault

## Support

Issues: https://github.com/smouj/openclaw-skills/issues?label=super-monitor
Docs: https://docs.openclaw.io/skills/super-monitor
```