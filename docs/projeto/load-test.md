# Teste de carga e HPA

Página da disciplina: [Load Testing](https://insper.github.io/platform/2026.2/exercises/project/load-testing/).
Arquivos em `store-pma/load-test/`: `locustfile.py`, `hpa-gateway.yaml` e o README com os
resultados. O teste de carga e o HPA são do Pedro Machado; esta página resume o trabalho dele
para o grupo.

## Cenário

- **Locust** (`locust==2.46.7`) com dois perfis: visitantes (`GET /products`) e clientes logados
  (`/products` + `/auth/me`, que passa pelo `/auth/solve` do gateway).
- **HPA do gateway**: CPU 50% do request, de 1 a 6 réplicas (`kubectl apply -f load-test/hpa-gateway.yaml`).
- metrics-server instalado como add-on do EKS pelo `up.ps1`.
- Ambiente: EKS us-east-2, 2 nós t3.medium, RDS db.t4g.micro. Gerador no Brasil: a rede sozinha
  custa ~160–230 ms por ida e volta, então o **p50 (~170 ms) é quase todo rede**. O que a carga
  afeta é a cauda (p95/p99).

## Resultados (12/11/2026)

| Cenário | Usuários | req/s | Falhas | p50 | p95 | p99 | CPU máx. do gateway |
|---|---|---|---|---|---|---|---|
| Referência, sem HPA | 100 | 77,5 | 0 | 170 ms | 260 ms | 440 ms | 470m |
| Estresse, sem HPA | 300 | 235 | 0 | 170 ms | 220 ms | 330 ms | 296m |
| **Estresse, sem HPA** | **800** | **566** | **0** | **180 ms** | **770 ms** | **1.400 ms** | **493m / 500m (teto)** |
| **Estresse, com HPA** | **800** | **597** | **0** | **170 ms** | **320 ms** | **910 ms** | dividida em 4 pods |

Com 800 usuários, o HPA escalou o gateway de **1 → 2 → 4 réplicas em ~40 s** (eventos
`SuccessfulRescale`) e o p95 caiu **58%** (770 → 320 ms).

## Vídeo do teste com HPA

O vídeo está na documentação do Pedro Garcia:
**[▶ vídeo do teste com HPA](https://ep-pc.github.io/store-pma-docs/projeto/load-test/#video-do-teste-com-hpa)**.

Gravação real, em tempo real (2 min), com três painéis:

- **em cima:** 4 geradores de carga chamando `GET /products` em loop, de dentro do cluster;
- **embaixo à esquerda:** `kubectl get hpa gateway -w`. A CPU vai de 0% a **186% do alvo** e o
  HPA sobe de **1 para 4 réplicas**; com as réplicas prontas, cai para 79% e, sem carga, para 49%;
- **embaixo à direita:** os pods do gateway nascendo e a CPU de cada um (`kubectl top`).

!!! note "Onde foi gravado"

    O vídeo usa os **mesmos `k8s.yaml` e o mesmo `hpa-gateway.yaml`** do EKS, num cluster kind
    (1 control plane + 2 nós, como os 2 nós do EKS), no GitHub Actions. Os números da tabela acima
    são os do teste com Locust no EKS em 12/11/2026.
