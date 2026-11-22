# Esta documentação

Feita com [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/), a partir do
[template da disciplina](https://hsandmann.github.io/documentation.template/) e da estrutura da
documentação do [Pedro Garcia](https://ep-pc.github.io/store-pma-docs/) (colega de grupo).
As páginas do grupo (arquitetura, serviços, como rodar, CI/CD, teste de carga, custos) foram
reaproveitadas e ajustadas; as páginas do Order, dos meus bottlenecks, das atividades e do uso de
IA são minhas. Publicada no GitHub Pages pelo workflow `.github/workflows/main.yaml`
(`mkdocs gh-deploy` a cada push na `main`).

## Rodar localmente

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt       # Windows: .venv\Scripts\pip
.venv/bin/mkdocs serve                          # http://127.0.0.1:8000
```

## Publicar

1. Push na `main`: o GitHub Actions builda o site e publica na branch `gh-pages`.
2. Uma vez só: em **Settings → Pages** do repositório, escolher *Deploy from a branch* →
   `gh-pages` / `root`.
3. Em **Settings → Actions → General → Workflow permissions**: *Read and write permissions*.
