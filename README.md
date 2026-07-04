# 📊 Análise de Dados — Gestão de RH

Projeto em Python para análise de indicadores de Recursos Humanos: headcount,
turnover, motivos de desligamento, folha salarial, absenteísmo, horas extras,
desempenho e perfil demográfico dos colaboradores.

## 🚀 Funcionalidades

- Geração de base de dados simulada (450 colaboradores, histórico de 3+ anos)
- Cálculo automático de KPIs de RH (headcount, turnover, folha, absenteísmo, desempenho)
- Geração de 6 gráficos analíticos em PNG
- Relatório executivo em texto, pronto para leitura ou anexo em e-mail

## 🗂️ Estrutura do projeto

## ⚙️ Como usar

1. Clone o repositório:
```bash
   git clone https://github.com/SEU_USUARIO/gestao-rh.git
   cd gestao-rh
```

2. Instale as dependências:
```bash
   pip install -r requirements.txt
```

3. (Opcional) Gere uma nova base de dados simulada:
```bash
   python3 gerar_dados.py
```
   Ou substitua `dados/colaboradores.csv` pelos seus dados reais, mantendo as mesmas colunas.

4. Rode a análise:
```bash
   python3 analise.py
```

Isso gera 6 gráficos em `graficos/` e um relatório completo em
`relatorios/relatorio_resumo.txt`.

## 📑 Colunas da base de dados

| Coluna                  | Descrição                                                |
|-------------------------|-----------------------------------------------------------|
| id_colaborador          | Identificador único do colaborador                        |
| departamento            | Departamento/área                                          |
| cargo                   | Cargo ocupado                                              |
| genero                  | Feminino / Masculino                                       |
| idade                   | Idade do colaborador                                       |
| escolaridade            | Ensino Médio / Graduação / Pós-graduação                   |
| salario                 | Salário mensal (R$)                                        |
| data_admissao           | Data de admissão                                            |
| ativo                   | True se ainda está na empresa, False se desligado           |
| data_desligamento       | Data de desligamento (vazio se ativo)                       |
| motivo_desligamento     | Motivo do desligamento (vazio se ativo)                     |
| tempo_casa_dias         | Tempo de casa em dias                                       |
| faltas_ultimos_12m      | Nº de faltas nos últimos 12 meses                            |
| horas_extras_mes_media  | Média de horas extras por mês                                |
| nota_desempenho         | Nota de avaliação de desempenho (0 a 10)                     |
| enps_individual         | Proxy de satisfação individual (0 a 10)                      |

## 📈 Indicadores (KPIs) calculados

- Headcount atual e histórico total
- Turnover dos últimos 12 meses
- Tempo médio de casa dos colaboradores ativos
- Perfil demográfico (idade, escolaridade, gênero)
- Folha salarial total e salário médio
- Absenteísmo médio e horas extras médias
- Nota média de desempenho e eNPS médio
- Headcount, salário médio e desempenho por departamento
- Ranking de motivos de desligamento
- Cargos com maior salário médio

## 📊 Gráficos gerados

1. Movimentação de pessoal por mês (admissões x desligamentos)
2. Headcount atual por departamento
3. Motivos de desligamento (histórico)
4. Distribuição salarial por departamento (boxplot)
5. Desempenho x faltas nos últimos 12 meses (dispersão)
6. Distribuição etária por gênero

## 🔄 Adaptando para dados reais

Basta substituir `dados/colaboradores.csv` por um arquivo com as mesmas colunas
(pode ter menos colunas — ajuste `analise.py` removendo os cálculos que
dependam de campos ausentes).

## 🛠️ Tecnologias

- Python 3
- pandas / numpy
- matplotlib / seaborn

## 📄 Licença
Este projeto pode ser usado livremente para fins de estudo e adaptação interna.


Este projeto pode ser usado livremente para fins de estudo e adaptação interna.
