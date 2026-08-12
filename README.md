# grafana-docs

Documentação e exports de tudo que tenho de projeto sobre Grafana: stacks locais (docker-compose,
provisioning) e dashboards versionados como JSON, sanitizados (sem IP interno, domínio, senha ou
token — só placeholders/variáveis de ambiente).

## Projetos

| Pasta | O que é |
|---|---|
| [`dns-monitoring/`](dns-monitoring/) | Stack local Grafana + ClickHouse para observabilidade de DNS (docker-compose, provisioning de datasource/dashboard, export do dashboard "Monitoramento de Rede DNS"). |
| [`grafana-flex-noc/`](grafana-flex-noc/) | Exports dos dashboards do NOC da FLEX Internet: MAPA FLEX e MAPA FLEX - STATUS CTO's (mapa geográfico de queda/atenuação de CTO via IXC x RADIUS) e NOC — Visão Geral da Rede (topologia física em Node Graph via Zabbix). |

## Convenções

- Cada subpasta é um projeto autocontido, com seu próprio `README.md`.
- Segredos nunca são versionados: `.env.example` traz só o placeholder, cada ambiente preenche o
  seu `.env` local (ignorado pelo git).
- Dashboards Grafana ficam como export JSON puro (`GET /api/dashboards/uid/:uid`, campo
  `dashboard`), sem o wrapper `meta` (que contém metadados específicos da instância).
