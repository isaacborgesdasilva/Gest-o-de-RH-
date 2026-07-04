# -*- coding: utf-8 -*-
"""
Projeto: Análise de Dados de Gestão de RH
-------------------------------------------
Lê a base dados/colaboradores.csv e gera:
  - Indicadores (KPIs) de headcount, turnover, absenteísmo, folha e desempenho
  - Gráficos (PNG) na pasta graficos/
  - Relatório resumo em texto na pasta relatorios/

Como executar:
    python3 analise.py
"""

import pandas as pd
import numpy as np
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt
import matplotlib.ticker as mticker
import seaborn as sns

sns.set_theme(style="whitegrid")
plt.rcParams["figure.dpi"] = 110

CAMINHO_DADOS = "dados/colaboradores.csv"
PASTA_GRAFICOS = "graficos"
PASTA_RELATORIOS = "relatorios"
DATA_REFERENCIA = pd.Timestamp("2026-06-30")


def carregar_dados(caminho: str) -> pd.DataFrame:
    df = pd.read_csv(caminho, parse_dates=["data_admissao", "data_desligamento"])
    df["ativo"] = df["ativo"].astype(bool)
    df["mes_admissao"] = df["data_admissao"].dt.to_period("M").astype(str)
    df["mes_desligamento"] = df["data_desligamento"].dt.to_period("M").astype(str)
    df.loc[df["data_desligamento"].isna(), "mes_desligamento"] = np.nan
    return df


def calcular_kpis(df: pd.DataFrame) -> dict:
    ativos = df[df["ativo"]]
    desligados = df[~df["ativo"]]

    headcount_inicial = ((df["data_admissao"] <= DATA_REFERENCIA - pd.DateOffset(years=1)) &
                          ((df["data_desligamento"].isna()) | (df["data_desligamento"] > DATA_REFERENCIA - pd.DateOffset(years=1)))).sum()
    headcount_atual = len(ativos)
    desligamentos_12m = (df["data_desligamento"] >= DATA_REFERENCIA - pd.DateOffset(years=1)).sum()
    turnover_12m = desligamentos_12m / max(headcount_inicial, 1) * 100

    kpis = {
        "headcount_atual": headcount_atual,
        "total_historico": len(df),
        "desligamentos_12m": int(desligamentos_12m),
        "turnover_12m_pct": turnover_12m,
        "tempo_medio_casa_anos": (ativos["tempo_casa_dias"] / 365).mean(),
        "idade_media": ativos["idade"].mean(),
        "pct_pos_graduacao": (ativos["escolaridade"] == "Pós-graduação").mean() * 100,
        "pct_feminino": (ativos["genero"] == "Feminino").mean() * 100,
        "pct_masculino": (ativos["genero"] == "Masculino").mean() * 100,
        "folha_salarial_total": ativos["salario"].sum(),
        "salario_medio": ativos["salario"].mean(),
        "faltas_media_12m": ativos["faltas_ultimos_12m"].mean(),
        "horas_extras_media_mes": ativos["horas_extras_mes_media"].mean(),
        "nota_desempenho_media": ativos["nota_desempenho"].mean(),
        "enps_medio": ativos["enps_individual"].mean(),
    }
    return kpis, desligados


def top_rankings(df: pd.DataFrame, ativos: pd.DataFrame):
    por_departamento = (
        ativos.groupby("departamento")
        .agg(headcount=("id_colaborador", "count"), salario_medio=("salario", "mean"),
             folha=("salario", "sum"), nota_media=("nota_desempenho", "mean"),
             faltas_media=("faltas_ultimos_12m", "mean"))
        .sort_values("headcount", ascending=False)
    )

    motivos = (
        df[~df["ativo"]]["motivo_desligamento"].value_counts()
        .rename_axis("motivo").reset_index(name="quantidade")
    )

    por_cargo_salario = (
        ativos.groupby("cargo")
        .agg(headcount=("id_colaborador", "count"), salario_medio=("salario", "mean"))
        .sort_values("salario_medio", ascending=False)
    )

    return por_departamento, motivos, por_cargo_salario


def formatar_eixo_moeda(ax, eixo="x"):
    formatter = mticker.FuncFormatter(lambda x, _: f"R$ {x:,.0f}")
    if eixo == "y":
        ax.yaxis.set_major_formatter(formatter)
    else:
        ax.xaxis.set_major_formatter(formatter)


def gerar_graficos(df: pd.DataFrame, ativos: pd.DataFrame, por_departamento, motivos):
    # 1. Evolução de headcount (admissões x desligamentos por mês)
    admissoes_mes = df.groupby("mes_admissao").size().rename("admissoes")
    desligamentos_mes = df.dropna(subset=["mes_desligamento"]).groupby("mes_desligamento").size().rename("desligamentos")
    evolucao = pd.concat([admissoes_mes, desligamentos_mes], axis=1).fillna(0).sort_index()
    evolucao["saldo"] = evolucao["admissoes"] - evolucao["desligamentos"]
    evolucao["headcount_acumulado"] = evolucao["saldo"].cumsum()

    fig, ax1 = plt.subplots(figsize=(11, 5))
    ax1.bar(evolucao.index, evolucao["admissoes"], color="seagreen", label="Admissões", alpha=0.8)
    ax1.bar(evolucao.index, -evolucao["desligamentos"], color="indianred", label="Desligamentos", alpha=0.8)
    ax1.set_ylabel("Admissões (+) / Desligamentos (-)")
    ax1.set_title("Movimentação de Pessoal por Mês (Admissões x Desligamentos)")
    plt.xticks(rotation=90, fontsize=7)
    ax1.axhline(0, color="black", linewidth=0.8)
    ax1.legend(loc="upper left")
    plt.tight_layout()
    plt.savefig(f"{PASTA_GRAFICOS}/01_movimentacao_pessoal.png")
    plt.close()

    # 2. Headcount por departamento
    fig, ax = plt.subplots(figsize=(8, 5))
    dados_dep = por_departamento.reset_index().sort_values("headcount", ascending=False)
    sns.barplot(data=dados_dep, x="headcount", y="departamento", hue="departamento",
                palette="crest", legend=False, ax=ax)
    ax.set_title("Headcount Atual por Departamento")
    ax.set_xlabel("Nº de colaboradores")
    ax.set_ylabel("Departamento")
    plt.tight_layout()
    plt.savefig(f"{PASTA_GRAFICOS}/02_headcount_departamento.png")
    plt.close()

    # 3. Motivos de desligamento
    fig, ax = plt.subplots(figsize=(7, 5))
    cores = sns.color_palette("Set2", len(motivos))
    ax.pie(motivos["quantidade"], labels=motivos["motivo"], autopct="%1.1f%%",
           colors=cores, startangle=90)
    ax.set_title("Motivos de Desligamento (histórico)")
    plt.tight_layout()
    plt.savefig(f"{PASTA_GRAFICOS}/03_motivos_desligamento.png")
    plt.close()

    # 4. Distribuição salarial por departamento (boxplot)
    fig, ax = plt.subplots(figsize=(9, 5))
    ordem = por_departamento.sort_values("salario_medio", ascending=False).index
    sns.boxplot(data=ativos, x="salario", y="departamento", order=ordem,
                hue="departamento", palette="flare", legend=False, ax=ax)
    ax.set_title("Distribuição Salarial por Departamento")
    ax.set_ylabel("Departamento")
    ax.set_xlabel("Salário (R$)")
    formatar_eixo_moeda(ax, "x")
    plt.tight_layout()
    plt.savefig(f"{PASTA_GRAFICOS}/04_distribuicao_salarial.png")
    plt.close()

    # 5. Desempenho x Faltas (dispersão) — identifica riscos de retenção
    fig, ax = plt.subplots(figsize=(8, 5))
    sns.scatterplot(data=ativos, x="faltas_ultimos_12m", y="nota_desempenho",
                     hue="departamento", palette="tab10", alpha=0.7, ax=ax)
    ax.set_title("Desempenho x Faltas nos Últimos 12 Meses")
    ax.set_xlabel("Faltas (últimos 12 meses)")
    ax.set_ylabel("Nota de Desempenho (0-10)")
    plt.tight_layout()
    plt.savefig(f"{PASTA_GRAFICOS}/05_desempenho_vs_faltas.png")
    plt.close()

    # 6. Pirâmide etária / distribuição por gênero
    fig, ax = plt.subplots(figsize=(8, 5))
    sns.histplot(data=ativos, x="idade", hue="genero", multiple="stack",
                 bins=15, palette="Set2", ax=ax)
    ax.set_title("Distribuição Etária dos Colaboradores Ativos por Gênero")
    ax.set_xlabel("Idade")
    ax.set_ylabel("Nº de colaboradores")
    plt.tight_layout()
    plt.savefig(f"{PASTA_GRAFICOS}/06_distribuicao_etaria_genero.png")
    plt.close()

    print(f"Gráficos salvos em '{PASTA_GRAFICOS}/'")


def gerar_relatorio_texto(kpis: dict, por_departamento, motivos, por_cargo_salario):
    linhas = []
    linhas.append("=" * 60)
    linhas.append("RELATÓRIO DE GESTÃO DE RH")
    linhas.append("=" * 60)
    linhas.append("")
    linhas.append("INDICADORES DE HEADCOUNT E TURNOVER")
    linhas.append("-" * 60)
    linhas.append(f"Headcount atual (ativos): {kpis['headcount_atual']}")
    linhas.append(f"Total histórico (ativos + desligados): {kpis['total_historico']}")
    linhas.append(f"Desligamentos nos últimos 12 meses: {kpis['desligamentos_12m']}")
    linhas.append(f"Turnover (últimos 12 meses): {kpis['turnover_12m_pct']:.1f}%")
    linhas.append(f"Tempo médio de casa (ativos): {kpis['tempo_medio_casa_anos']:.1f} anos")
    linhas.append("")
    linhas.append("PERFIL DOS COLABORADORES ATIVOS")
    linhas.append("-" * 60)
    linhas.append(f"Idade média: {kpis['idade_media']:.1f} anos")
    linhas.append(f"% com pós-graduação: {kpis['pct_pos_graduacao']:.1f}%")
    linhas.append(f"% feminino: {kpis['pct_feminino']:.1f}% | % masculino: {kpis['pct_masculino']:.1f}%")
    linhas.append("")
    linhas.append("FOLHA E DESEMPENHO")
    linhas.append("-" * 60)
    linhas.append(f"Folha salarial total (mensal, ativos): R$ {kpis['folha_salarial_total']:,.2f}")
    linhas.append(f"Salário médio: R$ {kpis['salario_medio']:,.2f}")
    linhas.append(f"Faltas médias (últimos 12 meses): {kpis['faltas_media_12m']:.1f}")
    linhas.append(f"Horas extras médias/mês: {kpis['horas_extras_media_mes']:.1f} h")
    linhas.append(f"Nota média de desempenho: {kpis['nota_desempenho_media']:.1f} / 10")
    linhas.append(f"eNPS médio (proxy individual): {kpis['enps_medio']:.1f} / 10")
    linhas.append("")
    linhas.append("HEADCOUNT E SALÁRIO POR DEPARTAMENTO")
    linhas.append("-" * 60)
    linhas.append(por_departamento.to_string())
    linhas.append("")
    linhas.append("MOTIVOS DE DESLIGAMENTO (HISTÓRICO)")
    linhas.append("-" * 60)
    linhas.append(motivos.to_string(index=False))
    linhas.append("")
    linhas.append("CARGOS COM MAIOR SALÁRIO MÉDIO (TOP 5)")
    linhas.append("-" * 60)
    linhas.append(por_cargo_salario.head(5).to_string())
    linhas.append("")

    texto = "\n".join(linhas)
    with open(f"{PASTA_RELATORIOS}/relatorio_resumo.txt", "w", encoding="utf-8") as f:
        f.write(texto)

    print(texto)
    print(f"\nRelatório salvo em '{PASTA_RELATORIOS}/relatorio_resumo.txt'")


def main():
    df = carregar_dados(CAMINHO_DADOS)
    ativos = df[df["ativo"]]
    kpis, desligados = calcular_kpis(df)
    por_departamento, motivos, por_cargo_salario = top_rankings(df, ativos)
    gerar_graficos(df, ativos, por_departamento, motivos)
    gerar_relatorio_texto(kpis, por_departamento, motivos, por_cargo_salario)


if __name__ == "__main__":
    main()
