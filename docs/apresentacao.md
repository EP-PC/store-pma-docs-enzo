# Apresentação

A [página de apresentação](https://insper.github.io/platform/2026.2/exercises/project/presentation/)
pede um vídeo de **1 a 2 minutos** com introdução, arquitetura, demonstração, desafios
(bottlenecks) e conclusão.

!!! info "Vídeo do grupo"

    Combinado com o professor: o grupo entrega **um vídeo só**, que está na documentação do
    Pedro Garcia. Eu não gravei um vídeo individual.

    **[▶ Assistir ao vídeo do grupo (1 min 58 s)](https://ep-pc.github.io/store-pma-docs/apresentacao/)**

## Onde a minha parte aparece no vídeo

| Tempo | Parte | O que é meu |
|---|---|---|
| 0:35 | Demonstração pela loja | o **Order**: 401 sem login, pedido criado e convertido em reais (que passa pelo Exchange), 422 para moeda inválida |
| 1:05 | Bottlenecks, 2 por integrante | os meus: **cache com Redis** e **observabilidade** |
| 1:21 | Observabilidade | o **painel Store / Order do Grafana** com tráfego |

## Roteiro da apresentação (minha parte)

1. **O Order:** pedidos do usuário logado, com preço vindo do Product e conversão de moeda pelo
   Exchange; padrão contrato + implementação ([Order](projeto/order.md)).
2. **Integração:** contrato do Exchange definido antes do serviço existir (`EXCHANGE.md`); o
   Exchange real entrou sem mudar o Order nem o gateway.
3. **Bottleneck 1, cache com Redis:** `GET ?currency=BRL` p50 de 335 ms para 10 ms; chamadas ao
   Product de 60 para 2 em 30 pedidos; Redis fora do ar não derruba o Order
   ([Bottlenecks](projeto/bottlenecks.md#1-cache-com-redis)).
4. **Bottleneck 2, observabilidade:** métricas numa porta que o gateway não alcança, painel no
   Grafana e o que ele revelou: o p95 alto vinha de moedas inválidas, que não entram no cache
   ([Bottlenecks](projeto/bottlenecks.md#2-observabilidade-prometheus-grafana)).
5. **Conclusão:** próximos passos (cache de "moeda não suportada", réplicas do Order no EKS).
