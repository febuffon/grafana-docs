# DNS Monitoring — Grafana + ClickHouse

Stack local de observabilidade de DNS: Grafana lendo telemetria de queries DNS armazenada em
ClickHouse (`dns_telemetry.dns_queries`), rodando em `network_mode: host` na porta `3001`.

## Estrutura

```
dns-monitoring/
├── docker-compose.yml              # serviço Grafana (imagem oficial, plugin ClickHouse)
├── .env.example                    # variáveis obrigatórias (copiar para .env)
├── provisioning/
│   ├── datasources/clickhouse.yml  # datasource ClickHouse-DNS provisionado automaticamente
│   └── dashboards/dns.yml          # provider de dashboards via arquivo (auto-reload a cada 5min)
└── dashboards/
    └── dns-overview.json           # dashboard "Monitoramento de Rede DNS" (26 painéis)
```

## Subir a stack

```bash
cp .env.example .env
$EDITOR .env   # defina CLICKHOUSE_PASSWORD e GRAFANA_ADMIN_PASSWORD
docker compose up -d
```

Requer um ClickHouse já rodando em `localhost:8123` com o banco `dns_telemetry` e a tabela
`dns_queries` populada (fora do escopo deste repositório — normalmente alimentado por um coletor
DNS separado, ex. `dnstap`).

Acesso: `http://localhost:3001` (usuário `admin`, senha definida em `GRAFANA_ADMIN_PASSWORD`).

## Dashboard "Monitoramento de Rede DNS"

26 painéis cobrindo, entre outros: volume de queries, cache hit rate, tuning de TTL/serve-expired,
distribuição IPv4 vs IPv6, latência de resolução. Provisionado automaticamente via arquivo — editar
`dashboards/dns-overview.json` e o Grafana recarrega em até 5 minutos (ou reiniciar o container para
aplicar na hora).
