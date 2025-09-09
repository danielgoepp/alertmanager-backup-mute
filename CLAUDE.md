# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview
Automated CPU alert silencing for Alertmanager during backup windows. Creates temporary silences in Alertmanager to suppress CPU alerts during daily backup operations (4:00 AM - 4:15 AM).

## Common Commands

### Local Development
```bash
# Test scripts locally
python3 backup_cpu_alert_silence.py 15
python3 backup_cpu_alert_manager.py monitor 15
python3 backup_cpu_alert_manager.py status

# Install dependencies
pip install -r requirements.txt
```

### Docker Operations
```bash
# Build and deploy (full deployment to K3s)
./build-and-deploy.sh

# Build multi-platform image manually
docker buildx build --platform linux/amd64,linux/arm64 -t danielgoepp/backup-cpu-alert:latest --push .

# Test container locally
docker build -t backup-cpu-alert:latest .
docker run --rm backup-cpu-alert:latest python3 backup_cpu_alert_silence.py 1
```

### Kubernetes Operations
```bash
# Deploy CronJob
kubectl apply -f backup-cpu-alert-cronjob.yaml

# Monitor deployment
kubectl get cronjobs -n management
kubectl get jobs --selector=app=backup-cpu-alert -n management
kubectl logs -l app=backup-cpu-alert -n management --tail=50

# Test immediately
kubectl create job --from=cronjob/backup-cpu-alert-silence backup-test -n management
kubectl logs job/backup-test -n management
kubectl delete job backup-test -n management
```

## Architecture

### Core Components
- **backup_cpu_alert_silence.py**: Simple silence creation script (containerized version)
- **backup_cpu_alert_manager.py**: Full-featured management script with start/stop/status/monitor modes
- **BackupCPUAlertManager class**: Main orchestration class with methods:
  - `create_silence(duration_minutes)`: Creates Alertmanager silence
  - `remove_cpu_silences()`: Removes active CPU silences  
  - `show_status()`: Displays silence status
  - `monitor_mode(duration)`: Full cycle with cleanup

### Deployment Architecture
- **Container**: Python 3.11 slim with non-root user
- **K3s CronJob**: Scheduled daily at 4:00 AM EST in `management` namespace
- **Docker Hub**: Public image `danielgoepp/backup-cpu-alert:latest` (multi-platform)
- **Resources**: 128Mi memory limit, 100m CPU limit

### API Integration
- **Target**: Alertmanager at `https://alertmanager-prod.goepp.net/api/v2`
- **Authentication**: None required (public API)
- **Matching**: Regex pattern `.*CPU.*|.*cpu.*|.*Cpu.*` on alertname field
- **Duration**: 15 minutes (configurable)

## Configuration
- **Schedule**: `0 4 * * *` (4:00 AM daily)
- **Timezone**: America/New_York  
- **Namespace**: management
- **Concurrency**: Forbidden (no overlapping runs)
- **Default Duration**: 15 minutes

## Environment Variables
- **TZ**: America/New_York (container timezone)

## Dependencies
- **Python**: 3.11+
- **External**: requests==2.31.0
- **Tools**: Docker with buildx, kubectl
- **Access**: Alertmanager API, Docker Hub (danielgoepp), K3s cluster

## Current Deployment Status
- **Active**: CronJob `backup-cpu-alert-silence` in management namespace
- **Last Updated**: August 18, 2025
- **Image**: danielgoepp/backup-cpu-alert:latest
- **Schedule**: Daily 4:00 AM EST