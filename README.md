# Otimização de Alocação de Recursos de T&D

Este documento apresenta a formulação e a solução computacional para o problema de alocação de orçamento de horas de treinamento, modelado como um **Problema da Mochila (Knapsack Problem)**.

## 1. Formulação do Problema

**Problema:** Otimização de Alocação de Recursos de T&D (Talentos e Desenvolvimento).

**Contexto:** Um gestor de T&D ("Ricardo") possui um orçamento fixo de horas de formação para distribuir entre vários colaboradores. Cada colaborador necessita de um certo número de horas para ser requalificado e, se requalificado, gera um "valor estratégico" (baseado na sua experiência, conhecimento tácito, etc.) para a empresa.

### Definições
* **Entrada:**
    * Uma lista de $N$ colaboradores (os "itens").
    * Para cada colaborador $i$:
        * `horas_necessarias[i]` (o "peso" do item).
        * `valor_estrategico[i]` (o "valor" do item).
    * Uma `capacidade_total_horas` (a "capacidade da mochila").
* **Saída:**
    * O valor estratégico máximo que pode ser obtido.
    * A lista de colaboradores selecionados para a requalificação que atinge esse valor máximo, sem exceder o orçamento.
* **Objetivo:**
    * Maximizar a soma do `valor_estrategico` dos colaboradores selecionados, sujeito à restrição de que a soma das `horas_necessarias` seja $\le$ `capacidade_total_horas`.

---

## 2. Solução em Python (Código Completo)

O script abaixo (`main.py`) inclui a geração de dados, manipulação via Pandas, ordenação via Merge Sort recursivo e a solução de otimização via Programação Dinâmica.

```python
import pandas as pd
import random
import sys

# Aumentar o limite de recursão para o Merge Sort em dataframes grandes
# e para a própria DP em casos com muitos itens.
sys.setrecursionlimit(3000)

def gerar_dados_colaboradores(num_colaboradores=25):
    """
    Cria uma lista de dicionários com dados fictícios de colaboradores.
    """
    nomes = [f"Colaborador_{chr(65 + i)}" for i in range(num_colaboradores)]
    dados = []
    for nome in nomes:
        dados.append({
            "Nome": nome,
            "Horas_Necessarias": random.randint(10, 100),  # "Peso" do item
            "Valor_Estrategico": random.randint(100, 1000) # "Valor" do item
        })
    return dados

def criar_dataframe(dados_lista):
    """
    Converte a lista de dados para um DataFrame do Pandas.
    """
    df = pd.DataFrame(dados_lista)
    return df

# --- Estrutura de Ordenação (Merge Sort Recursivo) ---

def merge_sort(df, coluna_ordenacao):
    """
    Função principal do Merge Sort, adaptada para ordenar um DataFrame.
    Esta é uma implementação 'acadêmica' para cumprir o requisito de recursão.
    Na prática, usaríamos df.sort_values().
    """
    if len(df) <= 1:
        return df

    meio = len(df) // 2
    
    # Divisão recursiva
    metade_esquerda = merge_sort(df.iloc[:meio].copy(), coluna_ordenacao)
    metade_direita = merge_sort(df.iloc[meio:].copy(), coluna_ordenacao)
    
    # Conquista (mesclagem)
    return merge(metade_esquerda, metade_direita, coluna_ordenacao)

def merge(esquerda, direita, coluna):
    """
    Função auxiliar do Merge Sort para mesclar dois DataFrames ordenados.
    """
    resultado_lista = []
    idx_esq, idx_dir = 0, 0
    
    # Converte DataFrames para listas de dicionários para iteração eficiente
    lista_esq = esquerda.to_dict('records')
    lista_dir = direita.to_dict('records')

    while idx_esq < len(lista_esq) and idx_dir < len(lista_dir):
        if lista_esq[idx_esq][coluna] <= lista_dir[idx_dir][coluna]:
            resultado_lista.append(lista_esq[idx_esq])
            idx_esq += 1
        else:
            resultado_lista.append(lista_dir[idx_dir])
            idx_dir += 1
            
    # Adiciona os elementos restantes
    while idx_esq < len(lista_esq):
        resultado_lista.append(lista_esq[idx_esq])
        idx_esq += 1
        
    while idx_dir < len(lista_dir):
        resultado_lista.append(lista_dir[idx_dir])
        idx_dir += 1

    return pd.DataFrame(resultado_lista)

# --- Solução DP (Problema da Mochila com Recursão + Memorização) ---

def otimizar_alocacao_formacao(colaboradores, capacidade_total_horas):
    """
    Função 'wrapper' (principal) que inicializa a memorização
    e chama a função recursiva de DP.
    
    Retorna (valor_maximo, lista_de_colaboradores_selecionados)
    """
    # 'memo' armazena os resultados já calculados (Programação Dinâmica)
    # A chave será (n_itens_considerados, capacidade_restante)
    memo = {}
    
    # Converte o DataFrame para uma lista de dicionários para acesso mais rápido
    # na recursão.
    lista_colaboradores = colaboradores.to_dict('records')
    
    n = len(lista_colaboradores)
    
    return knapsack_memo(lista_colaboradores, capacidade_total_horas, n, memo)

def knapsack_memo(colaboradores, capacidade, n, memo):
    """
    Função recursiva com memorização (Top-Down DP) para o Problema da Mochila.
    
    Args:
        colaboradores (list): Lista de dicionários dos itens.
        capacidade (int): Horas restantes na "mochila".
        n (int): Número de itens (colaboradores) que ainda podemos considerar.
        memo (dict): Dicionário de memorização.
        
    Returns:
        tuple: (valor_total_maximo, lista_de_nomes_selecionados)
    """
    # --- Casos Base ---
    # 1. Se não há mais capacidade ou não há mais itens, o valor é 0.
    if n == 0 or capacidade == 0:
        return (0, [])

    # --- Memorização ---
    # 2. Se já calculamos este subproblema (n, capacidade), retorna o resultado.
    estado_atual = (n, capacidade)
    if estado_atual in memo:
        return memo[estado_atual]

    # --- Decisão Recursiva ---
    # Pega o item atual (n-1 pois os índices são base 0)
    colaborador_atual = colaboradores[n - 1]
    horas = colaborador_atual["Horas_Necessarias"]
    valor = colaborador_atual["Valor_Estrategico"]
    nome = colaborador_atual["Nome"]

    # 3. Se o item atual NÃO CABE na mochila (horas > capacidade):
    # Não temos escolha, devemos EXCLUIR o item.
    # Passamos para o próximo item (n-1) com a MESMA capacidade.
    if horas > capacidade:
        resultado = knapsack_memo(colaboradores, capacidade, n - 1, memo)
    
    else:
        # 4. Se o item CABE, temos uma ESCOLHA:
        
        # Opção A: INCLUIR o colaborador
        # Ganhamos 'valor' e 'gastamos' 'horas' da capacidade.
        # Chamamos recursivamente para (n-1) e (capacidade - horas)
        valor_incluindo, nomes_incluindo = knapsack_memo(
            colaboradores, capacidade - horas, n - 1, memo
        )
        valor_incluindo += valor
        nomes_incluindo = nomes_incluindo + [nome] # Cria nova lista
        
        # Opção B: EXCLUIR o colaborador
        # Não ganhamos valor, não gastamos horas.
        # Chamamos recursivamente para (n-1) e (capacidade)
        valor_excluindo, nomes_excluindo = knapsack_memo(
            colaboradores, capacidade, n - 1, memo
        )

        # Comparamos as duas opções e escolhemos a MELHOR (max)
        if valor_incluindo > valor_excluindo:
            resultado = (valor_incluindo, nomes_incluindo)
        else:
            resultado = (valor_excluindo, nomes_excluindo)

    # 5. Antes de retornar, ARMAZENAMOS o resultado no 'memo'.
    memo[estado_atual] = resultado
    return resultado

# --- Estrutura de Saída (Relatório) ---

def apresentar_relatorio(df_original, capacidade, resultado_dp):
    """
    Imprime um relatório formatado dos resultados da otimização.
    """
    valor_max, nomes_selecionados = resultado_dp
    
    # Filtra o DataFrame original para mostrar os detalhes dos selecionados
    df_selecionados = df_original[df_original["Nome"].isin(nomes_selecionados)]
    
    horas_gastas = df_selecionados["Horas_Necessarias"].sum()
    
    print("=" * 70)
    print("    RELATÓRIO DE OTIMIZAÇÃO DE REQUALIFICAÇÃO (T&D) - A.R.C.")
    print("=" * 70)
    print(f"Orçamento Total de Horas de Formação: {capacidade} horas")
    print("-" * 70)
    print("Resultados da Otimização (Programação Dinâmica):")
    print(f"  > Valor Estratégico Máximo Alcançado: {valor_max}")
    print(f"  > Total de Horas Alocadas: {horas_gastas} / {capacidade}")
    print(f"  > Total de Colaboradores Requalificados: {len(nomes_selecionados)}")
    print("-" * 70)
    print("Colaboradores Selecionados para Requalificação:")
    
    # Define a ordem das colunas para o relatório
    colunas_relatorio = ["Nome", "Horas_Necessarias", "Valor_Estrategico"]
    print(df_selecionados[colunas_relatorio].to_string(index=False))
    print("=" * 70)

# --- Execução Principal (main) ---

if __name__ == "__main__":
    
    # 1. DEFINIR PARÂMETROS
    NUM_COLABORADORES = 25
    CAPACIDADE_HORAS_TOTAIS = 500 # "Capacidade da Mochila"
    
    # 2. ENTRADA: Gerar dados e criar DataFrame
    dados = gerar_dados_colaboradores(NUM_COLABORADORES)
    df_colaboradores = criar_dataframe(dados)
    
    print("\n--- (Entrada) Lista Completa de Colaboradores (Primeiros 10) ---")
    print(df_colaboradores.head(10).to_string(index=False))
    print("...")
    
    # 3. ORDENAÇÃO: Aplicar Merge Sort (como solicitado)
    # Vamos ordenar por "Valor_Estrategico" para fins de visualização
    df_ordenado = merge_sort(df_colaboradores, "Valor_Estrategico")
    
    print("\n--- (Ordenação) Lista Ordenada por Valor Estratégico (Maiores) ---")
    print(df_ordenado.tail(5).to_string(index=False)) # Mostra os 5 maiores valores
    
    # 4. PROCESSAMENTO: Rodar a solução de DP
    # Nota: A DP não precisa que os dados estejam pré-ordenados.
    # Usamos o df_colaboradores original.
    resultado_final_dp = otimizar_alocacao_formacao(
        df_colaboradores, CAPACIDADE_HORAS_TOTAIS
    )
    
    # 5. SAÍDA: Apresentar o relatório final
    apresentar_relatorio(df_colaboradores, CAPACIDADE_HORAS_TOTAIS, resultado_final_dp)
```

## 3. Explicação das Funções e Estruturas

- ```gerar_dados_colaboradores(num_colaboradores):``` Cria a massa de dados ($25$ itens, como solicitado) para o problema. Cada colaborador é representado como um dicionário contendo ```Nome```, ```Horas_Necessarias``` (peso) e ```Valor_Estrategico``` (valor).
- ```criar_dataframe(dados_lista):``` Utiliza a biblioteca Pandas para converter a lista de dados brutos num DataFrame. Esta estrutura é excelente para manipulação e visualização de dados tabulares.
- ```merge_sort(df, coluna_ordenacao):``` Implementa a etapa de "divisão" do algoritmo Merge Sort. Ela cumpre o requisito de recursividade dividindo o DataFrame ao meio repetidamente até que restem apenas DataFrames de 1 linha (caso base).
- ```merge(esquerda, direita, coluna):``` Implementa a etapa de "conquista". Ela recebe dois DataFrames já ordenados e os mescla comparando a coluna especificada. Esta é a lógica central de ordenação.
- ```otimizar_alocacao_formacao(...):``` Atua como uma função wrapper (invólucro). Seu objetivo é preparar os dados para a DP, inicializando o dicionário ```memo``` e convertendo o DataFrame em uma lista de dicionários para otimizar a velocidade de acesso durante a recursão profunda.
- ```knapsack_memo(colaboradores, capacidade, n, memo):``` É o núcleo da solução, utilizando Programação Dinâmica com abordagem Top-Down (recursão + memorização).
  - Estado: A chave do subproblema é a tupla $(n, capacidade)$.
  - Memorização: Verifica se o estado já existe em ```memo``` antes de calcular, evitando reprocessamento.
  - Decisão: Para cada colaborador, decide entre **INCLUIR** (ganha valor, gasta capacidade) ou **EXCLUIR** (mantém capacidade). Escolhe-se o ```max()``` entre as duas opções.
  - Backtracking: Retorna não apenas o valor máximo, mas constrói a lista de nomes selecionados durante o retorno da recursão.
- ```apresentar_relatorio(...):``` Responsável pela saída final. Recebe os resultados da DP, filtra o DataFrame original para recuperar os detalhes dos colaboradores escolhidos e exibe um relatório gerencial formatado com o total de horas utilizadas e o valor estratégico alcançado.
