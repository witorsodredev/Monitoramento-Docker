# Stack de monitoramento de containers (Podman Compose + Ubuntu Server)

Monitora os containers e o host de uma maquina **8 GB / 4 vCPU**, com **Grafana**
para dashboards e alertas.

## Componentes

| Servico | Porta | Funcao |
|---|---|---|
| Grafana | 3000 | Dashboards + alertas |
| Prometheus | 9090 | Coleta/armazena metricas |
| cAdvisor | 8080 | Metricas por container |
| Node Exporter | 9100 | Metricas do host |

## Pre-requisitos no Ubuntu Server

```bash
# Podman + podman-compose
sudo apt update
sudo apt install -y podman podman-compose
```

## Subir a stack

```bash
cd monitoring
# TROQUE a senha do Grafana no docker-compose.yml (GF_SECURITY_ADMIN_PASSWORD) antes!
podman-compose up -d
podman ps
```

Acesse o Grafana em `http://<ip-do-host>:3000` (admin / senha definida).
O datasource Prometheus ja vem provisionado automaticamente.

## Especificidades do Podman (ler se o cAdvisor nao mostrar containers)

1. **cgroups v2**: Ubuntu moderno usa cgroups v2. O cAdvisor roda com `privileged: true`
   e monta `/sys`, `/var/run` e `/var/lib/containers` justamente por isso.
2. **Path do runtime**: no Docker seria `/var/lib/docker`; no Podman e
   `/var/lib/containers` (ja ajustado no compose).
3. **Podman rootless**: se os 7 containers rodam em modo *rootless* (usuario comum,
   nao root), o cAdvisor privilegiado pode nao enxergar os cgroups do usuario.
   Opcoes:
   - Rodar a stack de monitoramento tambem no mesmo usuario rootless, ou
   - Usar o endpoint de metricas do proprio Podman:
     ```bash
     systemctl --user enable --now podman.socket
     # metricas em: podman system service + /metrics
     ```
   Para dashboards ricos por container, o cAdvisor root (containers em modo root)
   e o caminho mais simples.
4. **SELinux**: os volumes usam a flag `:z` / `:ro,z` para relabel automatico.
   Se houver "permission denied" em volume, confirme com `getenforce`.

## Validar que esta coletando

```bash
# Targets devem aparecer UP
curl -s http://localhost:9090/api/v1/targets | grep -o '"health":"[a-z]*"'

# cAdvisor expondo metricas de container
curl -s http://localhost:8080/metrics | grep container_memory_usage_bytes | head
```

## Alertas

As regras de alerta estao versionadas em `prometheus/alerts.yml` (host + containers:
memoria, CPU, disco, swap, container down, OOM iminente).

Voce pode:
- **Usar o Grafana Alerting** (recomendado): crie as regras na UI do Grafana apontando
  para o datasource Prometheus, e configure contact points (e-mail, Slack, webhook).
- Ou manter as regras no Prometheus (arquivo acima) e adicionar um Alertmanager.

## Dashboards sugeridos (importar por ID no Grafana)

- **893** / **14282** - cAdvisor / container metrics
- **1860** - Node Exporter Full (host)

## Notas de dimensionamento (8 GB / 4 vCPU)

- Prometheus com retencao de **15 dias** e teto de **2 GB** de TSDB.
- `mem_limit` definido em todos os servicos para nao competir com os 7 containers de producao.
- Se precisar de historico longo, use **remote_write** para um Prometheus gerenciado
  (ex.: Amazon Managed Prometheus) e mantenha so o curto prazo localmente.
