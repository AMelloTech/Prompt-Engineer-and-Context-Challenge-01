# Prompt Engineering and Context — Challenge 01

Desafio de pós-graduação: 8 questões aplicando frameworks de prompt engineering
em cenários fictícios da **Hill Valley Tech**. Ver o contexto completo da empresa
e do time em [`context/hill-valley-tech.md`](context/hill-valley-tech.md).

## Entrega

Cada questão vive em um arquivo Markdown próprio em [`questions/`](questions/),
com 4 campos obrigatórios: **Prompt**, **Modelo** (+ motivo da escolha),
**Output** e **Justificativa** (mapeando os componentes do framework ao prompt).
A Questão 08 compara o framework escolhido com 2 alternativas.

Pelo menos 2 providers distintos (OpenAI, Anthropic, Google, Meta ou local via
Ollama) são usados ao longo do desafio. Outputs ruins também são registrados,
com nota sobre o que seria feito diferente.

## Estrutura do repositório

```
.
├── README.md                      este arquivo — visão geral e índice
├── context/
│   └── hill-valley-tech.md        cenário da empresa e do time, referenciado por todas as questões
└── questions/
    ├── _template.md                modelo usado para cada questão
    ├── q01-*.md
    ├── ...
    └── q08-*.md
```

**Por que essa estrutura:** um arquivo por questão deixa o histórico do git
granular — cada commit mostra a evolução de um prompt específico — e o
front matter (`framework`, `provider`, `model`, `effort`) funciona como
metadado filtrável sem repetir o contexto da empresa em cada arquivo. Essa
escolha será revisitada no Capítulo 4 (Criação e Versionamento de Prompts).

## Índice das questões

| # | Framework | Provider | Modelo | Status |
|---|---|---|---|---|
| [01](questions/q01-placeholder.md) | — | — | — | pendente |
| [02](questions/q02-placeholder.md) | — | — | — | pendente |
| [03](questions/q03-placeholder.md) | — | — | — | pendente |
| [04](questions/q04-placeholder.md) | — | — | — | pendente |
| [05](questions/q05-placeholder.md) | — | — | — | pendente |
| [06](questions/q06-placeholder.md) | — | — | — | pendente |
| [07](questions/q07-placeholder.md) | — | — | — | pendente |
| [08](questions/q08-placeholder.md) | — (comparar c/ 2 alternativas) | — | — | pendente |
