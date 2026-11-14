# Comandos executados (em ordem, com horário)

Caminho A (Docker). O SQL foi rodado no DBeaver (conexões `primario` em localhost:5442 e `replica` em localhost:5443), e a carga de pedidos no psql interativo do primário.


| # | Horário (local) | Onde | Comando | Resultado | Evidência |
|---|---|---|---|---|---|
| 1 | 10:27:04 | DBeaver `replica` (5443) | `SELECT pg_is_in_recovery();` | `[v]` = **true** (é réplica) | `evidencias/01.png` |
| 2 | 10:27:33 | DBeaver `replica` (5443) | `SELECT count(*) FROM clientes;` | **1.000** | `evidencias/02.png` |
| 3 | 10:52 | CMD 1 | `docker compose exec primary psql -U postgres` | psql aberto no primário | — |
| 4 | 10:52:50 | CMD 1 (psql) | `INSERT INTO eventos DEFAULT VALUES RETURNING id;` + `\watch 0.1` | ids subindo, cerca de 10 por segundo (primeiro id = 1 às 13:52:50 UTC) | `evidencias/03-saida-watch-T0.txt` |
| 5 | **10:53:02 = T0** | CMD 2 | `docker compose kill primary` | `Container incidenteb-primary-1 Killed` | `evidencias/12.png` (topo) |
| 6 | 10:53:02 | CMD 1 (psql) | — | sessão encerrada; **último id impresso = 118** (13:53:02 UTC) | `evidencias/03.png`, `evidencias/03-saida-watch-T0.txt` |
| 7 | 10:54:26 | DBeaver `replica` (5443) | `SELECT pg_promote();` | `[v]` = **true** (levou 3,453 s) | `evidencias/04.png` |
| 8 | 10:55:45 | DBeaver `replica` (5443) | `SELECT pg_is_in_recovery();` | `[ ]` = **false** (virou primário) | `evidencias/05.png` |
| 9 | 10:58:40 | DBeaver `replica` (5443) | `SELECT max(id), max(ts) FROM eventos;` | **max(id) = 118**, max(ts) = 10:53:02.233 | `evidencias/07.png` |
| 10 | **11:01:28 = T1** | DBeaver `replica` (5443) | `INSERT INTO eventos DEFAULT VALUES RETURNING id, clock_timestamp();` | escrita aceita: id = 125, clock_timestamp = 11:01:28.554 −0300 | `evidencias/08.png` |
| 11 | ~11:11 | DBeaver `primario` (5442) | `select * from CLIENTES` | `Erro SQL [08001]: Connection to localhost:5442 refused` (antes) | `evidencias/09.png` |
| 12 | 11:12:00 | DBeaver `replica` (5443) | `INSERT INTO clientes (nome, email) VALUES ('pos-failover', 'pf@loja.com') RETURNING id;` | escrita aceita, id = 1.024 (depois) | `evidencias/10.png` |
| 13 | ~11:14 | CMD 2 | `docker compose start primary` | `Container incidenteb-primary-1 Started` | `evidencias/11.png`, `evidencias/12.png` |
| 14 | 11:14:39 | DBeaver `primario` (5442) | `SELECT pg_is_in_recovery()` | `[ ]` = **false**: antigo primário voltou como primário = **split-brain** | `evidencias/11.png` |
| 15 | ~11:17 | CMD 2 | `docker compose stop primary` | `Stopped` | `evidencias/12.png` |
| 16 | ~11:17 | CMD 2 | `docker compose rm -f primary` | `Removed` | `evidencias/12.png` |
| 17 | ~11:17 | CMD 2 | `docker volume ls` e `docker volume rm incidenteb_primary_data` | volume do antigo primário (o "disco cifrado") destruído | `evidencias/12.png` |

## Cálculos feitos com estes horários

- **RTO = T1 − T0 = 11:01:28 − 10:53:02 = 8 min 26 s = 506 s.** A promoção em si terminou às 10:54:26, 1 min 24 s depois da queda. O resto foram conferências e capturas manuais.
- **RPO = último id confirmado − max(id) no novo primário = 118 − 118 = 0 linhas perdidas.** O último pedido replicado (`max(ts)` = 10:53:02.233) é do mesmo segundo da queda.
- **Salto de ids (118 → 125 em `eventos`, 1.000 → 1.024 em `clientes`):** não é perda de dados. O PostgreSQL registra as sequências no WAL em blocos adiantados, e o novo primário continua a partir do último valor registrado.
