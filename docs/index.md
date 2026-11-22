# store-pma · documentação individual

???+ info inline end "Edição"

    Platforms, Microservices, DevOps and APIs
    Insper · 2026.2

## Aluno e grupo

**Enzo Perego** ([EnzoPerego](https://github.com/EnzoPerego)), responsável pelo **Order**
(contrato + serviço), pelo contrato de integração com o Exchange
([`EXCHANGE.md`](https://github.com/EP-PC/store-pma/blob/main/EXCHANGE.md)) e pelos meus dois
bottlenecks: **cache com Redis** e **observabilidade com Prometheus e Grafana**.

**Grupo 05** (organização [EP-PC](https://github.com/EP-PC) no GitHub, conta AWS `pmapi-bcc-grupo05`):

| Integrante | GitHub | O que fez no projeto |
|---|---|---|
| Pedro Machado | [pedrohmm1](https://github.com/pedrohmm1) | Gateway, Auth, Account e Product; compose, Minikube, EKS na AWS (`up.ps1`/`down.ps1`), Jenkins e CI/CD, teste de carga com HPA |
| Enzo Perego | [EnzoPerego](https://github.com/EnzoPerego) | Order (contrato + service), contrato do Exchange, Redis do Order, Prometheus e Grafana |
| Pedro Garcia | [garciapp2](https://github.com/garciapp2) | Exchange (Python, PTAX do BCB), cache e varredura de vulnerabilidades, custos/PaaS/SLA e vídeo de apresentação do grupo |

## Entregas do projeto

Tabela de notas da [página do projeto](https://insper.github.io/platform/2026.2/exercises/project/):

| Item | Peso | Status | Onde |
|---|---|---|---|
| Gateway | 5% | :material-check: | [Serviços](projeto/servicos.md#gateway) |
| Auth | 5% | :material-check: | [Serviços](projeto/servicos.md#auth) |
| Account | 5% | :material-check: | [Serviços](projeto/servicos.md#account) |
| Exchange | 5% | :material-check: | [Serviços](projeto/servicos.md#exchange) |
| Bottlenecks (≥ 2 por integrante) | 20% | :material-check: | [Bottlenecks](projeto/bottlenecks.md) |
| AWS | 5% | :material-check: | [Arquitetura](projeto/arquitetura.md#na-aws) |
| EKS | 10% | :material-check: | [Como rodar](projeto/como-rodar.md#aws-eks) |
| Jenkins (CI/CD) | 10% | :material-check: | [CI/CD](projeto/cicd.md) |
| Load Testing | 15% | :material-check: | [Teste de carga](projeto/load-test.md) |
| PaaS, Custos e SLA | 10% | :material-check: | [Custos](projeto/custos.md) |
| MkDocs | 10% | :material-check: | esta documentação |

Além da tabela: **Order** (o meu serviço) e **Product** (serviços extras do projeto), a
**declaração de uso de IA** ([Uso de IA](ia.md)) e o **vídeo de apresentação** do grupo, que
está na documentação do Pedro Garcia ([Apresentação](apresentacao.md)).

## Repositórios

| Repositório | O que tem |
|---|---|
| [EP-PC/store-pma](https://github.com/EP-PC/store-pma) | root: submódulos, compose, scripts da AWS, Jenkins, teste de carga, Prometheus/Grafana |
| [EP-PC/Order](https://github.com/EP-PC/Order) · [EP-PC/Order-service](https://github.com/EP-PC/Order-service) | **contrato e implementação do Order (meu serviço)** |
| [EP-PC/Gateway](https://github.com/EP-PC/Gateway) | gateway (Spring Cloud Gateway) |
| [EP-PC/Auth-service](https://github.com/EP-PC/Auth-service) | autenticação (JWT) |
| [EP-PC/Account](https://github.com/EP-PC/Account) · [EP-PC/Account-service](https://github.com/EP-PC/Account-service) | contrato e implementação do Account |
| [EP-PC/Product](https://github.com/EP-PC/Product) · [EP-PC/Product-service](https://github.com/EP-PC/Product-service) | contrato e implementação do Product |
| [EP-PC/Exchange-service](https://github.com/EP-PC/Exchange-service) | Exchange (Python) |
| [EP-PC/store-pma-docs-enzo](https://github.com/EP-PC/store-pma-docs-enzo) | esta documentação |
| [EP-PC/store-pma-docs](https://github.com/EP-PC/store-pma-docs) | documentação do Pedro Garcia ([site](https://ep-pc.github.io/store-pma-docs/), com o vídeo do grupo) |

!!! note "Repositórios privados"

    Parte dos repositórios da organização é privada (entre eles store-pma, Gateway, Order,
    Order-service e Exchange-service). O código do Order está descrito, com trechos, na página
    [Order](projeto/order.md).
