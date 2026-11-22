# Order (meu serviço)

Microsserviço de pedidos em **Java 25 + Spring Boot 4.1.1** (Spring Cloud 2025.1.3, OpenFeign),
no mesmo padrão "contrato + implementação" dos outros serviços Java do grupo. Enunciado:
[Order API](https://insper.github.io/platform/2026.2/exercises/order/). Código:

- [EP-PC/Order](https://github.com/EP-PC/Order): **contrato** `store:order:1.0.0` (interface +
  DTOs), submódulo `api/order`;
- [EP-PC/Order-service](https://github.com/EP-PC/Order-service): **implementação**, submódulo
  `api/order-service`.

## Endpoints

| Endpoint | Sucesso | Erros |
|---|---|---|
| `POST /orders` | 201 + `Location` + pedido | 400 (itens inválidos ou produto inexistente), 401, 503 (Product fora) |
| `GET /orders` | 200, pedidos do usuário `{id, date, total}` | 401 |
| `GET /orders/{id}` | 200, valores em USD | 401, 404 (não existe ou é de outro usuário) |
| `GET /orders/{id}?currency=XXX` | 200, valores convertidos | 401, 404, 422 (moeda não suportada), 503 (Exchange fora) |
| `GET /orders/health-check` | 200, sem login | — |

=== "POST /orders"

    ```json title="requisição"
    { "items": [
        { "idProduct": "0195abfb-7074-73a9-9d26-b4b9fbaab0a8", "quantity": 2 },
        { "idProduct": "0195abfe-e416-7052-be3b-27cdaf12a984", "quantity": 10 } ] }
    ```

    ```json title="resposta 201"
    {
      "id": "0195ac33-73e5-7cb3-90ca-7b5e7e549569",
      "date": "2025-09-01T12:30:00",
      "currency": "USD",
      "items": [
        { "id": "01961b9a-...", "product": { "id": "0195abfb-..." }, "quantity": 2, "total": 20.24 },
        { "id": "01961b9b-...", "product": { "id": "0195abfe-..." }, "quantity": 10, "total": 6.20 }
      ],
      "total": 26.44
    }
    ```

=== "GET /orders/{id}?currency=BRL"

    ```json title="resposta 200 (sell = 6.00)"
    {
      "id": "0195ac33-73e5-7cb3-90ca-7b5e7e549569",
      "date": "2025-09-01T12:30:00",
      "currency": "BRL",
      "items": [
        { "id": "01961b9a-...", "product": { "id": "0195abfb-..." }, "quantity": 2, "total": 121.44 },
        { "id": "01961b9b-...", "product": { "id": "0195abfe-..." }, "quantity": 10, "total": 37.20 }
      ],
      "total": 158.64
    }
    ```

Os números são os do exemplo do enunciado, e o Order devolve exatamente esse JSON (testado com
MockMvc e de ponta a ponta pelo gateway). Erros saem no mesmo formato dos outros serviços:
`{timestamp, status, error, message, path}`.

## Regras de negócio

**Identidade.** O gateway valida o JWT e injeta `id-account`; o Order só lê o header. Sem ele,
**401**. O contrato declara o header com `required = false` de propósito: com `true`, o Spring
responderia 400 sozinho, e o certo é 401.

**Pedido de outro usuário.** O repositório busca por `(id, id_account)`: para outro usuário o
pedido simplesmente "não existe" (**404**), sem revelar que o id é válido. O 404 vem antes da
moeda: pedido alheio com `?currency=` continua 404, sem chamar o Exchange.

**Criação.**

- Todos os itens são validados **antes** da primeira chamada ao Product: `items` não vazio,
  `idProduct` preenchido e `quantity > 0` (senão **400**).
- Cada produto é buscado no Product (`GET /products/{id}`) pelo cliente Feign do contrato
  `store:product`. Produto inexistente (404 no Product) vira **400**, como pede o enunciado;
  Product fora do ar, 5xx ou timeout vira **503**.
- O preço do produto é **congelado** no item (`unit_price`): se o produto mudar de preço depois,
  o pedido antigo não muda. Total do item = `unit_price × quantity`; total do pedido = soma.
- A data sai sem fração de segundo (`2025-09-01T12:30:00`), como no enunciado.

**Conversão de moeda.** O banco guarda tudo em **USD**.

- Sem `currency` (ou `USD`), não chama o Exchange.
- A moeda é normalizada (`" eur "` → `EUR`); se não tiver 3 letras (`REAL`), **422** sem chamar
  o Exchange.
- Chama `GET http://exchange:8080/exchange/USD/{moeda}` repassando o `id-account` e usa a taxa
  **`sell`**: para pagar em reais um pedido em dólar, o cliente compra dólar, e a casa de câmbio
  vende.
- Converte o total de **cada item** (2 casas, `HALF_UP`) e soma os itens convertidos. Assim os
  itens exibidos sempre batem com o total: com `sell = 5.4321`, a soma dá 143,63, e converter o
  total direto daria 143,62.
- Exchange responde 400/404/422 (moeda desconhecida) → **422**; Exchange fora, 5xx, timeout ou
  `sell` inválido → **503**.
- Os clientes Feign têm timeout de **2 s** para conectar e **5 s** para ler (o padrão do Feign
  esperaria até 60 s, segurando a requisição do usuário).

```java title="OrderService.java (trecho da conversão)"
private Order convert(Order order, String currency, String idAccount) {
    String target = currency == null ? "" : currency.trim().toUpperCase(Locale.ROOT);
    if (target.isEmpty() || USD.equals(target)) {
        return order.currency(USD);                        // sem chamar o Exchange
    }
    if (!CURRENCY_CODE.matcher(target).matches()) {        // [A-Z]{3}
        throw new ResponseStatusException(HttpStatus.UNPROCESSABLE_CONTENT, "Currency not supported: " + currency);
    }
    BigDecimal rate = rate(target, idAccount);             // sell do Exchange (com cache)
    List<OrderItem> items = order.items().stream()
        .map(i -> OrderItem.builder()
            .id(i.id()).idProduct(i.idProduct()).quantity(i.quantity()).unitPrice(i.unitPrice())
            .total(i.total().multiply(rate).setScale(2, RoundingMode.HALF_UP))
            .build())
        .toList();
    BigDecimal total = items.stream().map(OrderItem::total).reduce(BigDecimal.ZERO, BigDecimal::add);
    return Order.builder().id(order.id()).idAccount(order.idAccount()).date(order.date())
        .currency(target).items(items).total(total).build();
}
```

## Contrato com o Exchange (definido por mim)

Quando o Order ficou pronto, o Exchange ainda não existia. Defini o contrato no
[`EXCHANGE.md`](https://github.com/EP-PC/store-pma/blob/main/EXCHANGE.md) do store-pma e testei o
Order contra um **Exchange falso** que seguia esse contrato (inclusive com latência simulada,
para testar o timeout). O Exchange real, feito depois pelo Pedro Garcia, segue o mesmo contrato,
então entrou no projeto **sem mudar o Order nem o gateway**.

| Decisão | Por quê |
|---|---|
| `GET /exchange/{from}/{to}` no **singular** | o gateway já roteava `/exchange/**`; o próprio exemplo do enunciado usa o singular |
| header `id-account` obrigatório (401 sem ele) | o Exchange só atende usuário logado; o Order repassa o header do usuário |
| resposta `{sell, buy, date, id-account}`, números > 0 | igual ao enunciado; o Order usa só o `sell` |
| moeda desconhecida = 400/404/422 → 422 no Order | o enunciado pede 422; o Exchange fica livre para escolher qual 4xx |
| API externa fora → 503 | mesmo critério dos outros serviços |
| `GET /exchange/health-check` aberto, sem `id-account` | o gateway abriu os health-checks; a probe não manda header |

## Estrutura do código

**Contrato (`EP-PC/Order`, pacote `store.order`).** O jar é usado pelo service para implementar
as rotas e pode ser usado por qualquer outro serviço como cliente Feign.

| Tipo | Papel |
|---|---|
| `OrderController` | interface com `@FeignClient` + rotas (`@PostMapping("/orders")`...) e o header `id-account` nas assinaturas |
| `OrderIn`, `OrderItemIn` | body do `POST` |
| `OrderOut`, `OrderItemOut`, `ProductRefOut` | resposta do `POST` e do detalhe (`"product": {"id": ...}`) |
| `OrderSummaryOut` | cada linha do `GET /orders` (`{id, date, total}`) |

**Implementação (`EP-PC/Order-service`).**

| Classe | Papel |
|---|---|
| `OrderResource` | `@RestController implements OrderController`: só a camada HTTP (201 + `Location`) |
| `OrderService` | regras: validação, preço, total, conversão, tradução de erros (400/401/404/422/503) |
| `ProductLookup`, `ExchangeLookup` | consultas ao Product e ao Exchange com `@Cacheable` ([bottleneck 1](bottlenecks.md)) |
| `CacheConfig` | caches no Redis (JSON, TTL 60 s) e tratamento de Redis fora do ar |
| `ExchangeClient`, `ExchangeOut` | cliente Feign do Exchange (Python, sem jar de contrato) |
| `Order`, `OrderItem` | domínio |
| `OrderModel`, `OrderItemModel` | entidades JPA; `OrderModel` é a raiz do agregado |
| `OrderRepository` | `findByIdAccountOrderByDateDesc` e `findByIdAndIdAccount` (com `@EntityGraph`) |
| `OrderParser`, `OrderExceptionHandler` | conversão In/Out ↔ domínio e formato único de erro |
| `OrderServiceApplication` | `@EnableFeignClients(basePackages = {"store.product", "store.exchange"})` |

O `@EnableFeignClients` **não** inclui `store.order`: o `OrderController` também é `@FeignClient`
e viraria um cliente da própria interface, conflitando com o `OrderResource`.

## Banco (schema `orders`, Flyway)

| Tabela | Colunas | Índices e restrições |
|---|---|---|
| `orders` | `id`, `id_account`, `date`, `total` | índice em `id_account` (listagem por dono) |
| `order_items` | `id`, `id_order`, `id_product`, `quantity`, `unit_price` | FK para `orders`, índice em `id_order`, `CHECK (quantity > 0)` |

- **Agregado:** o pedido salva os itens em cascata (`@OneToMany(cascade = ALL)`); não existe
  `OrderItemRepository`.
- O total do pedido é gravado (a listagem não precisa carregar itens); o total do item é
  calculado (`unit_price × quantity`).
- O detalhe carrega pedido + itens numa consulta só (`@EntityGraph`); a listagem não carrega
  itens (evita uma consulta de itens por pedido).
- **`spring.jpa.open-in-view: false`:** a conexão do banco volta ao pool logo depois de cada
  consulta, em vez de ficar presa durante a chamada ao Exchange.
- O `create()` não é `@Transactional`: as chamadas HTTP ao Product acontecem antes, sem segurar
  conexão; o `save()` abre a própria transação.

## Testes (42)

| Classe | Tipo | O que cobre |
|---|---|---|
| `OrderServiceTest` (20) | unitário (Mockito, sem Spring) | criação, validações, produto inexistente, Product/Exchange fora, 401, listagem por dono, 404 de outro usuário, conversão (taxa `sell`, arredondamento, normalização, 4xx → 422) |
| `OrderResourceTest` (13) | integração HTTP (`@WebMvcTest` + MockMvc, `OrderService` real) | rotas, header, status, `Location`, JSON igual ao do enunciado, JSON malformado, BRL convertido |
| `LookupCacheTest` (9) | contexto Spring com cache ligado | mesma consulta = 1 chamada HTTP, cotação compartilhada entre usuários, erros e respostas inválidas fora do cache, serialização JSON do Redis |

Banco, Redis, Product e Exchange são mocks: os testes rodam sem Postgres nem rede, inclusive no
Jenkins. Para conferir que os testes pegam erro de verdade, estraguei o código de propósito
(produto inexistente devolvendo 404 e checagem do `id-account` desligada): **8 testes falharam**,
e voltaram a passar ao desfazer.

## Infraestrutura

| Arquivo | O que faz |
|---|---|
| `Dockerfile` | `eclipse-temurin:25-jre` + o jar (igual aos outros serviços Java) |
| `compose.yaml` | teste local do order sozinho: Postgres + Redis + order |
| `k8s/k8s.yaml` | Deployment + Service `order` (`ClusterIP` 8080, porta 8081 de métricas fora do Service), Deployment + Service `order-redis`; `readinessProbe` `httpGet` em `/orders/health-check`; 256–512 Mi e `MaxRAMPercentage=50` |
| `Jenkinsfile` | `Dependencies` (jobs `order` e `product`) → `Build` **com os testes** + relatório JUnit → `Build & Push Image` (amd64 + arm64) → `Deploy to EKS` |

A imagem é [`enzoperego/order`](https://hub.docker.com/r/enzoperego/order), publicada para
**linux/amd64 e linux/arm64**: um `docker build` simples num Mac M1/M2 gera só arm64, que não roda
nos nós amd64 do EKS.

## Integração no store-pma

1. Submódulos `api/order` e `api/order-service`.
2. `api/compose.yaml`: serviços `order` (`DATABASE_*` iguais às do product) e `order-redis`, mais
   Prometheus e Grafana no profile `observability`.
3. `api/build.sh` / `api/build.ps1`: contrato `order` no `install` e `order-service` no `package`.
4. `aws/up.ps1`: `order-service` na lista de manifestos e `order` no `rollout status`.
5. `README.md` do root e `EXCHANGE.md`.
6. Gateway: **nada mudou** (a rota `/orders/**` já existia; o Pedro Machado abriu o
   `GET /orders/health-check`).

Teste de ponta a ponta, só pelo gateway, no compose:

| Cenário | Esperado | Obtido |
|---|---|---|
| usuário A cria pedido (2 × 10,12 + 10 × 0,62) | 201, total 26,44, `Location` com o host público | ok |
| A lista e vê o detalhe | 200 `{id, date, total}` / 200 em USD | ok |
| A pede em `BRL` (Exchange com `sell` 6,00) | 121,44 + 37,20 = 158,64 | ok |
| A pede em `XYZ` | 422 | ok |
| B tenta ver o pedido de A | 404 | ok |
| B forja o header `id-account` de A | 404 (o gateway apaga e injeta o de B) | ok |
| sem cookie, mesmo forjando `id-account` | 401 no gateway | ok |
| produto inexistente / quantidade 0 | 400 / 400 | ok |
| `GET /orders/health-check` sem login | 200 | ok |

Os bottlenecks do Order estão em [Bottlenecks](bottlenecks.md#enzo-perego-order).
