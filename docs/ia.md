# Declaração de uso de IA

A [página do projeto](https://insper.github.io/platform/2026.2/exercises/project/) permite o uso de
IA na implementação, limitado a tarefas específicas, e exige que ele seja declarado: o que foi
feito com IA e o que foi feito pelo integrante. Esta declaração cobre a minha parte (Enzo
Perego). Os outros integrantes declaram o uso deles nas próprias documentações.

**Ferramenta:** Claude Code (Anthropic), como assistente no terminal e no VS Code.

## Como trabalhei com a IA

Escrevi um plano em etapas (reconhecimento do modelo do grupo, contrato, serviço, testes,
containerização, integração com o Exchange, integração no store-pma, documentação) e pedi para a
IA seguir **um passo de cada vez**: antes de cada etapa ela mostrava o desenho e só seguia com a
minha aprovação; a cada arquivo, explicava o que ele fazia e por quê. Os commits foram feitos com
a minha autorização, um por unidade lógica, e os pushes foram feitos por mim.

## O que foi feito com ajuda de IA

| Parte | Uso da IA |
|---|---|
| Leitura do modelo do grupo | resumo do padrão contrato + implementação do Product-service e do enunciado do Order |
| Contrato `Order` | rascunho da interface `OrderController` e dos DTOs, a partir do enunciado |
| `Order-service` | rascunho do código (entidades, repositório, service, resource, tratamento de erros, Flyway) adaptado do Product-service |
| Testes | rascunho dos 42 testes e verificação de que eles pegam erros (código estragado de propósito) |
| `Dockerfile`, `compose.yaml`, `k8s/k8s.yaml`, `Jenkinsfile` | adaptação dos modelos do Product-service |
| Contrato do Exchange (`EXCHANGE.md`) | rascunho das regras (path, resposta, erros, timeouts) e um Exchange simulado para os testes |
| Bottlenecks | implementação do cache com Redis e da observabilidade (actuator, Prometheus, Grafana e o painel) e execução das medições |
| Integração no store-pma | edição do compose, dos scripts de build, do `up.ps1` e dos READMEs |
| Esta documentação | rascunho do texto a partir dos READMEs e da documentação do Pedro Garcia, usada como modelo |

## O que foi feito por mim

- o plano de trabalho em etapas e as regras (seguir o modelo do grupo, nunca commitar segredos,
  não alterar os serviços dos colegas, declarar o uso de IA);
- as decisões ao longo do caminho, entre elas:
    - seguir o padrão do projeto (`OrderController`/`OrderResource`, IDs UUID) em vez dos nomes
      do meu plano inicial;
    - desligar o `open-in-view` do JPA;
    - rodar os testes no Jenkins (diferente do modelo, que pulava os testes);
    - as regras do Exchange (path no singular, taxa `sell`, 4xx → 422), que eu defini e passei
      para o colega;
    - os dois bottlenecks (cache com Redis e observabilidade) e quando publicar a imagem;
- a criação da conta e do repositório de imagens no Docker Hub, os pushes e a coordenação com o
  grupo (gateway, EKS, Exchange);
- a revisão do código, dos testes, dos números e do texto antes de cada commit.

!!! warning "Revisar antes de entregar"

    Esta lista foi rascunhada com a IA a partir do histórico do trabalho. Ajuste o que for preciso
    para refletir exatamente o que você fez e revisou: a disciplina exige que cada integrante saiba
    explicar e defender tudo o que entregou, inclusive a parte feita com IA.
