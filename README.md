# Incidente B — Ransomware derruba o servidor principal

## 1. Identificação

| Campo | Valor |
|---|---|
| Aluno(a) | RAFAEL DE AVIZ CONSERVA |
| Cenário | B — Ransomware derruba o servidor principal |
| Data | 14/11/2026 |
| Caminho usado | **A — Docker** (docker compose), SQL executado no **DBeaver** (conexões `primario-5442` e `replica-5443`) e no psql interativo |
| Ambiente | PostgreSQL 16. Primário em `localhost:5442`, réplica em `localhost:5443`. Replicação física por streaming, **assíncrona**, com o slot `replica1_slot`. |

## 2. Resumo do incidente

Em horário de pico, um ransomware criptografou o disco do servidor primário e ele parou de responder na hora. A simulação foi feita com `docker compose kill primary`, que manda SIGKILL: o processo morre sem desligamento limpo, igual a uma queda real. A aplicação estava gravando cerca de 10 pedidos por segundo na tabela `eventos`.

A réplica foi promovida manualmente com `pg_promote()` e passou a ser o novo primário. O serviço voltou em **8 min 26 s = 506 segundos  (11:01:28 − 10:53:02)(RTO)  obs: Esses horarios estão diferentes do print devido as 3h descontadas do UTC  ** e **[0] registros (RPO)** foram perdidos.

## 3. Linha do tempo

| Horário | Ação | Resultado |
|---|---|---|
| [10:27:04] | Conferência do ambiente na `replica-5443` | `pg_is_in_recovery() = true`, `count(*) = 1000` (`evidencias/01`) | 
| [10:52:10] | Carga no psql do primário (`INSERT ... \watch 0.1`) | ids subindo (~10 por segundo) |
| **[10:53:02]** | **T0:** `docker compose kill primary` | primário derrubado (`evidencias/03-saida-watch-T0.txt`) |
| [10:53:02] | psql do primário | sessão encerrada; último id impresso = **118** (`evidencias/03`) |
| [10:54:26] | `SELECT pg_promote();` na `replica-5443` | `true` (`evidencias/04`) |
| [10:55:45] | `SELECT pg_is_in_recovery();` | `false`: a réplica virou primário (`evidencias/05`) |
| [10:58:40] | `SELECT max(id) FROM eventos;` | **[118]** (`evidencias/07`) |
| **[11:01:28]** | **T1:** `INSERT ... RETURNING id, clock_timestamp();` | escrita aceita (`evidencias/08`) |
| [11:12:00] | Conexão `primario-5442` × `replica-5443` | 5442 recusa; 5443 aceita escrita (`evidencias/09`, `10`) |
| [11:14:39] | Antigo primário religado e depois isolado | split-brain demonstrado; volume destruído (`evidencias/11`, `12`) |

## 4. Medições de RTO e RPO

### Método

- **T0**: horário da última linha impressa pelo \watch antes da queda (13:53:02 UTC = 10:53:02), registrado em evidencias/03-saida-watch-T0.txt. O primário foi derrubado com docker compose kill primary nesse mesmo segundo
- **T1**: clock_timestamp() retornado pelo primeiro INSERT aceito no novo primário (14:01:28 UTC = 11:01:28), no print evidencias/08.png
- **Último id confirmado**: último `id` que o `\watch` imprimiu antes do erro. Um id só é impresso depois do COMMIT, então ele foi confirmado para o cliente.
- **max(id) no novo primário**: medido **depois da promoção e antes de qualquer nova escrita**. Uma escrita nova já pularia a sequência (veja a observação abaixo).

### Cálculo do RTO

```
RTO = T1 − T0
RTO = 11:01:28 − 10:53:02 = 8 min 26 s = 506 segundos
obs: Esses horarios estão diferentes do print devido as 3h descontadas do UTC
```

### Cálculo do RPO

```
RPO = (último id confirmado) − (max(id) no novo primário)
RPO = 118 − 118 = 0 registro(s)
```
nenhuma transação confirmada foi perdida.


## 5. Causa raiz

- **Causa imediata:** o servidor primário ficou indisponível (disco criptografado pelo ransomware) e não existia failover automático. Ninguém detecta a falha nem promove a réplica sem ação humana.
- **Causa do RPO maior que zero (se houver):** a replicação é **assíncrona**. O primário confirma o COMMIT para o cliente antes de a réplica receber o WAL. As transações confirmadas nessa janela ficam só no primário e se perdem com ele.
- **Fatores contribuintes:** a aplicação tem o endereço do banco fixo (porta 5442), sem um endpoint virtual. Também não há fencing para impedir que o antigo primário volte.
- **Ponto de atenção sobre ransomware:** a réplica física só protege contra a perda **do servidor**. Se o ataque cifrasse ou apagasse dados **via SQL** (UPDATE, DROP), o dano seria replicado na hora. Nesse caso, só backup e PITR, de preferência imutáveis, resolvem.

## 6. Correção aplicada e prova

### O que a aplicação precisou mudar

A string de conexão passou de `host=localhost port=5442` para `host=localhost port=5443`. Usuários, senhas e permissões vieram junto, porque a replicação física copia o cluster inteiro. Só o endereço muda.

### Antes

a conexão primario-5442 recusada

### Depois

 `replica-5443` com `pg_is_in_recovery() = false` e um INSERT funcionando.


## 7. Comparação com serviço gerenciado

| Aspecto | Nosso ambiente | Serviço gerenciado (ex.: AWS RDS Multi-AZ, Google Cloud SQL HA) |
|---|---|---|
| Detecção da falha | Manual (humano percebe) | Automática, por health checks |
| Failover | Manual (`pg_promote()`) | Automático |
| RTO típico | 506s medidos, mas sem contar a detecção humana | Na faixa de 1–2 minutos, segundo a documentação do provedor |
| RPO | 0 registros (assíncrona) | ≈ 0 no Multi-AZ, porque o standby é **síncrono** |
| Endereço da aplicação | Precisou mudar de 5442 para 5443 | Endpoint DNS fixo, que passa a apontar para o novo primário |
| Antigo primário | Isolado e reconstruído à mão | Substituído e reconstruído pelo provedor |
| Ransomware via SQL | Réplica também é afetada | **PITR** restaura para um ponto antes do ataque; snapshots com retenção ou bloqueio contra exclusão |

## 8. Prevenção

- Failover automático (ex.: Patroni + etcd, ou repmgr) e um endpoint único (HAProxy, VIP ou DNS).
- Replicação síncrona (`synchronous_standby_names`, `synchronous_commit = on/remote_apply`) para dados críticos, aceitando a latência maior.
- Backups e arquivamento de WAL fora do servidor, imutáveis e testados (PITR).
- Monitoramento do atraso da réplica (`pg_stat_replication`) com alertas.
- Ter mais de uma réplica: depois do failover, o ambiente ficou **sem nenhuma réplica**.
- Hardening contra ransomware: menor privilégio, segmentação de rede, patches.

## 9. Respostas às perguntas do cenário

**O que o RTO medido incluiu e deixou de fora?**
Incluiu o tempo entre a queda e a primeira escrita bem-sucedida: a reação, a promoção e a gravação. Ficou de fora:
- a **detecção** (eu sabia da queda porque eu mesmo causei);
- a decisão humana numa situação real (diagnóstico e acionamento da equipe);
- a **mudança de DNS ou configuração** e a propagação dela;
- o reinício da aplicação e a **reconexão** do pool de conexões;
- a verificação de integridade;
- a reconstrução de uma nova réplica (a alta disponibilidade só volta depois disso).

O RTO real seria bem maior.

**Por que o RPO deu esse valor e quando seria maior?**
 Deu 0 porque primário e réplica estão na mesma máquina, a latência de rede é quase zero e a carga era baixa (~10 inserts/s). O WAL chegava à réplica em milissegundos. Com replicação assíncrona, o RPO cresce com o **atraso da réplica**. Ele seria maior com:
- carga de escrita alta;
- rede lenta ou réplica em outra região (WAN);
- réplica sobrecarregada (disco ou consultas pesadas);
- queda da rede entre os dois antes da falha.

Com replicação síncrona, o RPO seria 0.

## 10. Estrutura do repositório

```
README.md            relatório (este arquivo)
comandos.md          comandos executados, em ordem, com horário
docker-compose.yml   ambiente usado
evidencias/          saídas dos comandos e capturas de tela
```
