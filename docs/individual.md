# Atividades individuais

A página de apresentação pede, além do projeto, a documentação e o código das atividades feitas.
As atividades da seção *Exercises → Individual* do site (por exemplo,
[Order API](https://insper.github.io/platform/2026.2/exercises/order/)) e onde cada uma está:

| Atividade | Situação | Onde |
|---|---|---|
| Order API | **feita por mim**, dentro do projeto | [Order](projeto/order.md) · [EP-PC/Order](https://github.com/EP-PC/Order) · [EP-PC/Order-service](https://github.com/EP-PC/Order-service) |
| Bottlenecks | **2 feitos por mim**, no Order | [Bottlenecks](projeto/bottlenecks.md#enzo-perego-order) |
| Jenkins | **pipeline do Order** (dependências, testes, imagem multi-plataforma, deploy no EKS) | [CI/CD](projeto/cicd.md#pipeline-do-order) · `Jenkinsfile` do Order-service |
| MiniKube | **manifesto `k8s/k8s.yaml` do Order** (order + order-redis), como o de cada serviço | [Como rodar](projeto/como-rodar.md#minikube) |
| Exchange API | feita no projeto por outro integrante (Pedro Garcia), sobre o **contrato que eu defini** | [Serviços](projeto/servicos.md#exchange) · [Order](projeto/order.md#contrato-com-o-exchange-definido-por-mim) |
| Product API | feita no projeto por outro integrante (Pedro Machado) | [Serviços](projeto/servicos.md#product) |

## O que eu fiz

- **Order**: contrato `store:order:1.0.0` (interface `OrderController` + DTOs) e implementação
  (Spring Boot 4, JPA, Flyway, OpenFeign); validação no Product com preço congelado; conversão
  de moeda pelo Exchange; **42 testes automatizados**; Dockerfile; manifesto Kubernetes com
  probe no health-check; Jenkinsfile que roda os testes; imagem `enzoperego/order` para amd64 e
  arm64.
- **Contrato do Exchange** (`EXCHANGE.md`), definido e testado com um Exchange simulado antes de o
  serviço existir.
- **Integração** no store-pma: submódulos, compose, scripts de build, `up.ps1` e README.
- **Bottlenecks**: cache com Redis (medido: `GET ?currency=BRL` p50 335 ms → 10 ms; chamadas ao
  Product 60 → 2) e observabilidade com Prometheus e Grafana (painel provisionado, métricas fora
  do alcance do gateway).
- **Esta documentação.**
