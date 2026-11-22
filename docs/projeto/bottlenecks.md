# Bottlenecks

A [página de apresentação](https://insper.github.io/platform/2026.2/exercises/project/presentation/)
pede **pelo menos 2 bottlenecks por integrante**. As opções estão na página
[Bottlenecks](https://insper.github.io/platform/2026.2/exercises/bottlenecks/): caching,
observability, messaging, load balancing, vulnerability scanning, payment processing e RBAC/OAuth2.

| Integrante | Bottleneck 1 | Bottleneck 2 |
|---|---|---|
| **Enzo Perego (Order)** | **Caching**: Redis para preços e cotações | **Observability**: Prometheus + Grafana |
| Pedro Garcia (Exchange) | **Caching**: cache das cotações com fallback | **Vulnerability scanning**: pip-audit + Trivy no pipeline |
| Pedro Machado (Gateway, Auth, Account, Product) | **Load balancing / escala**: HPA do gateway | **Disponibilidade no deploy**: readinessProbe |

## Enzo Perego (Order)

### 1. Cache com Redis

**Problema.** Toda requisição do Order depende de outro serviço pela rede. O `POST /orders` faz
uma chamada ao Product **por item** (preço e existência do produto), e o
`GET /orders/{id}?currency=` faz uma chamada ao Exchange, que por sua vez consulta uma API de
câmbio externa (centenas de milissegundos). Pedidos seguidos repetem as mesmas consultas.

**Solução.** Cache do Spring (`@Cacheable`) com um **Redis próprio do Order** (`order-redis`):

| Cache | Chave | Valor | TTL |
|---|---|---|---|
| `products` | `order:products::<idProduct>` | produto (JSON) | 60 s |
| `rates` | `order:rates::USD-<moeda>` | taxa `sell` | 60 s |

- A cotação é a mesma para todos os usuários: a chave **não** inclui o `id-account`, então a
  consulta de um usuário aproveita a de outro.
- Só entra no cache resposta válida: produto inexistente (404), corpo vazio, taxa inválida e
  erros não são guardados (um produto criado depois aparece na consulta seguinte).
- Até 60 s de defasagem no preço/cotação é o custo aceito (TTL configurável em
  `store.order.cache.ttl`).
- **Redis fora do ar não derruba o Order:** um `CacheErrorHandler` registra um aviso e a
  requisição segue sem cache, chamando o Product/Exchange direto. Timeouts de 0,5 s no Redis
  para não travar.
- Chaves com prefixo `order:`, para não colidir com outro serviço que use o mesmo Redis.
- O `@Cacheable` fica em classes próprias (`ProductLookup`, `ExchangeLookup`), e não no
  `OrderService`, porque funciona via proxy do Spring: uma chamada de dentro da própria classe
  não passaria pelo cache.

```java title="ProductLookup.java e ExchangeLookup.java (trechos)"
// só entra no cache produto encontrado: 404 (exceção) e corpo vazio (null) ficam de fora
@Cacheable(cacheNames = CacheConfig.PRODUCTS, unless = "#result == null")
public ProductOut find(String idProduct) {
    return productController.findById(idProduct).getBody();
}

// chave "USD-BRL", sem o id-account: a cotação de um usuário serve para todos
@Cacheable(cacheNames = CacheConfig.RATES, key = "#from + '-' + #to", unless = "#result == null")
public BigDecimal sell(String from, String to, String idAccount) {
    ExchangeOut exchange = exchangeClient.rate(from, to, idAccount).getBody();
    if (exchange == null || exchange.sell() == null || exchange.sell().signum() <= 0) {
        return null;                        // resposta inválida: não cacheia; o Order responde 503
    }
    return exchange.sell();
}
```

**Como testei.** No compose local, 30 requisições seguidas de cada tipo, com o Redis de verdade
e um Exchange simulado com **300 ms de latência** (como uma API externa). O mesmo código rodou
duas vezes: com o cache desligado (`SPRING_CACHE_TYPE=none`) e ligado.

| Requisição | Sem cache (p50 / média) | Com cache (p50 / média) | Chamadas externas |
|---|---|---|---|
| `GET /orders/{id}?currency=BRL` | 335 ms / 357 ms | **10 ms / 67 ms** | Exchange: 30 → **1** |
| `POST /orders` (2 itens) | 19 ms / 33 ms | **11 ms / 18 ms** | Product: 60 → **2** |

- No Redis: **87 hits e 3 misses** (2 produtos + 1 cotação). A média com cache inclui a primeira
  requisição (cache vazio + aquecimento da JVM).
- As chaves ficam legíveis no `redis-cli`, com o TTL contando:

  ```text
  order:products::28494943-...  (TTL 59 s) = {"id":"28494943-...","name":"Caneca","price":10.12,"unit":"un"}
  order:rates::USD-BRL          (TTL 58 s) = 6.0
  ```

- **Redis derrubado no meio do teste:** as mesmas requisições continuaram respondendo 200/201,
  sem nenhum erro, só com avisos no log e ~0,5 s a mais (o timeout do Redis).
- O `LookupCacheTest` (9 testes) prova a regra em contexto Spring: 5 consultas ao mesmo produto
  = **1** chamada HTTP; cotação de um usuário reaproveitada por outro; 404, corpo vazio, taxa
  inválida e erros **não** entram no cache.

**Relação com o cache do Exchange.** O Exchange (Pedro Garcia) também guarda as cotações, por
5 min. As duas camadas se completam: a do Order evita a chamada de rede ao Exchange; a do Exchange
protege o PTAX quando há várias réplicas do Order, quando alguém chama `/exchange` direto pelo
gateway e quando o PTAX cai.

### 2. Observabilidade (Prometheus + Grafana)

**Problema.** Sem métricas, não dava para saber onde o Order gasta tempo, quantas chamadas faz ao
Product e ao Exchange, se o cache está acertando ou se o pool do banco está no limite. As medições
do bottleneck 1 precisaram de scripts feitos à mão.

**Solução.**

- **No Order:** `spring-boot-starter-actuator` + `micrometer-registry-prometheus` +
  `feign-micrometer`. As métricas ficam em `GET :8081/actuator/prometheus`, numa **porta de
  management separada**: o gateway só encaminha `/orders/**` para a 8080, então as métricas nunca
  ficam acessíveis de fora (testado: pela 8080, `/actuator/prometheus` responde **404**).
- Histogramas de latência ligados para as rotas (`http.server.requests`) e para as chamadas Feign
  (`http.client.requests`), o que permite calcular p95/p99. Todas as métricas levam o rótulo
  `application="order"`.
- **No store-pma:** Prometheus (coleta a cada 5 s) e Grafana no profile `observability` do
  `api/compose.yaml` (só sobem quando pedidos, para não pesar no dia a dia), com a fonte de dados
  e o painel **Store / Order** provisionados por arquivo (`api/observability/`), sem configurar
  nada na tela.

```yaml title="application.yaml do Order (trecho)"
management:
  server:
    port: ${MANAGEMENT_PORT:8081}       # fora do alcance do gateway
  endpoints:
    web:
      exposure:
        include: prometheus
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
        http.client.requests: true
    tags:
      application: order
```

| Painel do Grafana | Métrica |
|---|---|
| Requisições/s, erros (4xx+5xx), latência p95, por rota | `http_server_requests_seconds_*` |
| Chamadas ao Product e ao Exchange e latência p95 | `http_client_requests_seconds_*` por `clientName` (Feign) |
| Acerto do cache (bottleneck 1), por cache | `cache_gets_total` (hit/miss) |
| Conexões ativas no pool do banco | `hikaricp_connections_active` |
| Heap da JVM | `jvm_memory_used_bytes{area="heap"}` |

```bash
cd store-pma/api
docker compose --profile observability up --build -d
# Grafana: http://localhost:3000 (painel Store / Order, sem login, só leitura)
# tráfego de demonstração: api/observability/traffic.py (instruções no topo do arquivo)
```

**Como testei.** 90 s de tráfego misto contra o Order (4 clientes simultâneos, cerca de 5.500
requisições: criar pedido, listar, detalhar, converter, produto inexistente, moeda inválida), com
o Exchange simulado com 300 ms de latência. Números lidos no Prometheus:

| Medida | Valor |
|---|---|
| `GET /orders` | p50 3,8 ms / p95 10,1 ms |
| `POST /orders` | p50 3,9 ms / p95 11,9 ms |
| `GET /orders/{id}` (com e sem `currency`) | p50 2,4 ms / **p95 284 ms** |
| Acerto do cache | `products` **96,1%**, `rates` 83,8% |
| Chamadas ao Product | 173 para cerca de 2.400 pedidos (o resto saiu do cache) |
| Conexões ativas no pool do banco | no máximo 2 (de 10) |
| Heap da JVM | no máximo 104 MB |

**O que o painel mostrou.** O p95 do `GET /orders/{id}` (284 ms) ficou muito acima do p50
(2,4 ms). O painel das chamadas Feign explicou: quase todas as chamadas ao Exchange vinham de
pedidos com moeda **inválida** (`?currency=XYZ`), porque o cache do Order, de propósito, não
guarda erros; cada uma pagava a latência do Exchange simulado. As conversões válidas (`BRL`)
saíam do cache. No Exchange real, moeda inválida responde 404 **sem chamar o PTAX** (a lista de
moedas fica em cache por 24 h), o que reduz esse custo do outro lado. O pool do banco nunca passou
de 2 conexões ativas: o banco não é o gargalo nesse cenário, o que também confirma o efeito de
desligar o `open-in-view`.

!!! note "Seções dos colegas"

    As duas seções abaixo são dos outros integrantes e foram reproduzidas da documentação do
    [Pedro Garcia](https://ep-pc.github.io/store-pma-docs/projeto/bottlenecks/) para o grupo
    ficar completo. Os bottlenecks que eu implementei são os da seção acima.

## Pedro Garcia (Exchange)

### 1. Cache das cotações com fallback

**Problema.** Cada conversão chamava o PTAX do Banco Central (~230 ms). Além da latência, o
Exchange ficava preso à disponibilidade de um serviço público: se o PTAX caísse, toda conversão
de pedido do Order virava 503. E o site pede para **não fazer teste de carga contra o PTAX**.

**Solução** (`ptax.py`):

- cache em memória **por moeda**, TTL de 5 min (`CACHE_SECONDS`). Como a cotação é a mesma para
  todos os usuários, a de um aproveita a do outro;
- **um lock por moeda** (*single flight*): com o cache vazio e várias requisições ao mesmo tempo,
  só uma vai ao PTAX; as outras esperam e usam o valor que ela trouxe;
- **fallback**: se o PTAX falhar e houver uma cotação de até 24 h (`MAX_STALE_SECONDS`), o serviço
  responde com ela (`X-Cache: stale`) em vez de 503, e só tenta o PTAX de novo a cada 30 s
  (`RETRY_SECONDS`), para não martelar um serviço que está fora;
- lista de moedas em cache por 24 h: moeda inválida (`XYZ`) dá 404 sem chamar o PTAX.

```python title="ptax.py (trecho)"
with self._lock(key):                      # uma busca por moeda por vez
    entry = self._cache.get(key)
    now = self.clock()
    if entry and now < entry.expires_at:   # outra requisição já atualizou
        return entry.value, "hit"
    try:
        value = fetch()                     # PTAX
    except ProviderError:
        if entry and now - entry.fetched_at <= self.max_stale_seconds:
            entry.expires_at = now + self.retry_seconds
            entry.stale = True
            return entry.value, "stale"     # cotação antiga em vez de 503
        raise
    self._cache[key] = _Entry(value, fetched_at=now, expires_at=now + ttl)
    return value, "miss"
```

**Como testar.** `bench/bench.py` dispara 100 × `GET /exchange/USD/BRL` (5 em paralelo) e conta o
header `X-Cache`. Rodei uma vez com `CACHE_SECONDS=0` (cache desligado) e outra com o padrão,
sempre com o serviço recém-iniciado e contra o PTAX real (18/11/2026):

| | Sem cache | Com cache | Diferença |
|---|---|---|---|
| Latência p50 | 233,9 ms | 3,7 ms | **−98%** |
| Latência p95 | 279,7 ms | 5,5 ms | **−98%** |
| Vazão | 20 req/s | 347 req/s | **17×** |
| Chamadas ao PTAX | 101 | 2 | **−98%** |

```text
== SEM cache ==
requisicoes: 100 (5 em paralelo) em 4.92 s -> 20 req/s
X-Cache: {'miss': 100}
latencia ms: p50=233.9 p95=279.7 max=387.4
chamadas PTAX: 101
== COM cache (servico frio) ==
requisicoes: 100 (5 em paralelo) em 0.29 s -> 347 req/s
X-Cache: {'miss': 1, 'hit': 99}
latencia ms: p50=3.7 p95=5.5 max=216.3
chamadas PTAX: 2
```

As 5 primeiras requisições chegaram juntas com o cache vazio e só **uma** foi ao PTAX: o lock
funcionou. A outra chamada é a lista de moedas. O fallback está coberto pelos testes
`test_ptax_fora_serve_cotacao_antiga` e `test_cotacao_antiga_demais_vira_503`, que usam um PTAX
simulado e um relógio falso.

**Relação com o Order.** O Order também guarda a cotação no Redis por 60 s. As duas camadas se
completam: a do Order evita a chamada de rede ao Exchange; a do Exchange protege o PTAX quando
há várias réplicas do Order, quando alguém chama `/exchange` direto pelo gateway e quando o PTAX
cai. O Order tinha apontado como próximo passo cachear "moeda inválida"; no Exchange, moeda
inválida já responde 404 sem chamar o PTAX, porque a lista de moedas fica em cache por 24 h.

**Limite.** O cache vive na memória do processo: com N réplicas, cada pod faz no máximo uma
chamada por moeda a cada 5 min. Para compartilhar entre pods, o próximo passo seria Redis.

### 2. Varredura de vulnerabilidades no pipeline

**Problema.** Uma dependência com vulnerabilidade conhecida, ou uma imagem base desatualizada,
chegava ao Docker Hub e ao EKS sem ninguém perceber: o pipeline só buildava e publicava.

**Solução** (`Dockerfile` + `Jenkinsfile`):

1. **`Test & Audit`**: `docker build --target test` roda `pytest` e
   [`pip-audit`](https://github.com/pypa/pip-audit) (base de vulnerabilidades do PyPI/OSV) nas
   versões fixadas em `requirements.txt`. Teste falhando ou vulnerabilidade conhecida param o
   build. Roda dentro do Docker porque o Jenkins não tem Python.
2. **`Image Scan`**: o [Trivy](https://trivy.dev) varre a imagem final (pacotes Debian da imagem
   base + pacotes Python) e **barra a publicação** se houver vulnerabilidade **HIGH ou CRITICAL
   com correção disponível** (`--ignore-unfixed --exit-code 1`).
3. Endurecimento: versões fixas, `docker build --pull` (imagem base sempre atualizada), imagem
   final sem root (`USER 10001`) e `runAsNonRoot: true` no Kubernetes.

```groovy title="Jenkinsfile (trecho)"
stage('Image Scan') {
    steps {
        sh "docker build -t ${env.SERVICE}:scan ."
        sh """
            docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                -v trivy-cache:/root/.cache/ ${env.TRIVY} image --scanners vuln \
                --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 ${env.SERVICE}:scan
        """
    }
}
```

**Como testar.** Rodei o `pip-audit` (18/11/2026) em dois conjuntos de dependências:

| Dependências | Resultado do `pip-audit` | Pipeline |
|---|---|---|
| antigas: `fastapi==0.109.0`, `uvicorn==0.27.0`, `httpx==0.26.0` | **15 vulnerabilidades em 2 pacotes** (14 no `starlette`, 1 no `fastapi`) | para no `Test & Audit` |
| atuais: `fastapi==0.165.0`, `uvicorn==0.54.0`, `httpx==0.28.1` | **nenhuma vulnerabilidade conhecida** | segue |

**O Trivy pegou um problema de verdade.** A primeira versão da imagem do Exchange foi **barrada**
(exit code 1), mesmo com as dependências limpas, por 2 vulnerabilidades HIGH que já tinham
correção. Nenhuma das duas vinha do código do grupo:

| Pacote | CVE | De onde vinha | Correção no `Dockerfile` |
|---|---|---|---|
| `zlib1g` 1.3.1-1+b1 | CVE-2026-85091 | Debian da imagem `python:3.13-slim`, ainda sem o patch `deb13u1` | `apt-get upgrade` no estágio base |
| `setuptools` 70.3.0 | CVE-2025-47273 | embutido no próprio `pip` (`pip/_vendor`) | imagem final sem `pip`, que não é usado em produção |

Depois da correção, o mesmo `trivy image ... --exit-code 1` passou com **exit code 0** (gravação
no vídeo de apresentação). O segundo caso mostra por que varrer a **imagem**, e não só o
`requirements.txt`: o `pip-audit` não enxerga o que vem dentro da imagem base.

**Por que barrar só HIGH/CRITICAL com correção.** Barrar tudo pararia o pipeline por falhas sem
correção disponível na imagem base, sobre as quais o grupo não tem o que fazer. O que tem
correção vira ação concreta: subir a versão ou rebuildar com `--pull`.

## Pedro Machado (Gateway, Auth, Account, Product)

### 1. Gateway saturando CPU → HPA

Com 800 usuários, o gateway bateu no limite de CPU (500m) e as requisições fizeram fila. Com o
**HPA** (1 → 4 réplicas em ~40 s), o p95 caiu de 770 para **320 ms (−58%)** e o p99 de 1.400 para
910 ms (−35%), sem erros. Detalhes em [Teste de carga](load-test.md).

### 2. Deploy derrubando a API → readinessProbe

Sem probe, o Kubernetes mandava tráfego para o pod novo antes do Spring terminar de subir: ~40 s
de HTTP 503 a cada deploy. Com `readinessProbe`, o rollout passou a perder no máximo 1 requisição.
Detalhes em [CI/CD](cicd.md#deploy-sem-derrubar-a-api-readinessprobe).
