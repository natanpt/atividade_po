# Problema de Roteirização com Coleta e Entrega (Pickup & Delivery — PDVRP)

**Grupo:** Aguisson Alves, Natan Pedrosa, José Luciano e João Pedro

**Resumo:**  
Este projeto modela e resolve um problema de roteirização de veículos com pares de coleta e entrega (pickup & delivery). O objetivo é atribuir e ordenar visitas a nós (coletas e entregas) por uma frota de veículos com capacidades limitadas, minimizando a distância total percorrida e respeitando precedência (coleta antes da entrega) e restrições de capacidade.

**Tipo:**  
Problema de otimização combinatória — Programação inteira mista / Problema de roteirização com restrições (PDVRP). A solução é construída com abordagem de Routing (Constraint Programming / OR-Tools), que trata variáveis discretas de roteamento e dimensões (cumulativas) para capacidades.

---

## Contexto / enunciado
Dada uma rede de nós (um depósito e vários clientes que são, alternativamente, pontos de coleta ou de entrega) e uma frota de veículos com capacidades limitadas, planejar rotas que:

- comecem e terminem no depósito;
- atendam todos os nós;
- respeitem que cada par (coleta, entrega) seja visitado pelo mesmo veículo e que a coleta ocorra antes da entrega;
- nunca excedam a capacidade do veículo ao longo da rota;
- minimizem o custo total (por exemplo, distância ou tempo).

Aplicações típicas: logística reversa (coleta de devoluções), transporte de mercadorias que exigem picking e posterior entrega, serviços simultâneos de coleta e entrega por um mesmo veículo.

---

## Conjuntos e parâmetros

- \(V = \{0,1,\dots,n\}\): conjunto de nós, onde \(0\) é o depósito.  
- \(P \subseteq V	imes V\): conjunto de pares \((p,d)\) onde \(p\) é nó de pickup (coleta) e \(d\) é nó de delivery (entrega).  
- \(K=\{1,\dots,m\}\): conjunto de veículos.

Parâmetros:

- \(c_{uv}\): custo (distância/tempo) de ir do nó \(u\) para o nó \(v\). (Matriz de distâncias \(\mathbf{C}\).)
- \(q_i\): demanda no nó \(i\). Convenção: \(q_p>0\) em nós de coleta, \(q_d=-q_p\) no nó de entrega correspondente; \(q_0=0\).
- \(Q_k\): capacidade do veículo \(k\).
- \(d_{max}\) (opcional): limite máximo de distância/tempo por veículo.
- \(t_i\) (opcional): tempo de serviço no nó \(i\).

---

## Variáveis de decisão

Usando a formulação inspirada no modelo de roteamento (OR-Tools):

- \(x_{uv}^k \in \{0,1\}\): 1 se o veículo \(k\) percorre o arco \(u	o v\). (No framework OR-Tools, estas variáveis são implícitas no RoutingModel.)
- \(pos_i^k\) (implícito via ordenação/índice) — ordem de visita do nó \(i\) pela rota do veículo \(k\). (No OR-Tools a precedência é tratada via índices e restrições de pickup/delivery.)
- \(l_i^k\) — carga cumulativa (cumulVar da dimensão 'Capacity') do veículo \(k\) ao sair do nó \(i\). No OR-Tools é representada pela dimensão `Capacity`.

---

## Modelagem matemática

**Função objetivo**  
Minimizar a distância total percorrida por todos os veículos:
\[
\min \sum_{k\in K}\sum_{u\in V}\sum_{v\in V} c_{uv}\, x_{uv}^k
\]

**Restrições principais**

1. **Cada nó (não-depósito) é visitado exatamente uma vez:**
\[
\sum_{k\in K}\sum_{u\in V} x_{ui}^k = 1,\qquad orall i\in V\setminus\{0\}
\]

2. **Fluxo (entrada = saída) para cada nó e veículo:**
\[
\sum_{v\in V} x_{iv}^k - \sum_{u\in V} x_{ui}^k = 0,\qquad orall i\in V,\;orall k\in K
\]

3. **Capacidade dos veículos (carga cumulativa):**
Para cada arco \(u	o v\) usado pelo veículo \(k\):
\[
l_v^k \ge l_u^k + q_v - M\,(1 - x_{uv}^k)
\]
e
\[
0 \le l_i^k \le Q_k,\qquad orall i\in V,\;orall k\in K
\]
(com \(M\) grande o suficiente — no OR-Tools isso é tratado naturalmente via `AddDimensionWithVehicleCapacity`).

4. **Precedência e pareamento pickup-delivery:**  
Para cada par \((p,d)\in P\):
- ambos \(p\) e \(d\) devem ser servidos pelo mesmo veículo;
- a coleta precede a entrega:
\[
	ext{vehicle}(p)=	ext{vehicle}(d)
\quad	ext{e}\quad pos_p < pos_d
\]
No OR-Tools: `RoutingModel.AddPickupAndDelivery(p_idx, d_idx)` e vinculamos `VehicleVar` e `CumulVar` quando necessário.

5. **Início e término no depósito:**
Cada veículo começa e termina no depósito:
\[
	ext{Start}_k = 0,\quad 	ext{End}_k = 0,\qquad orall k\in K
\]

6. **(Opcional) Limite de distância por veículo:**
Se aplicável, adicionar dimensão `Distance` com limite máximo por veículo:
\[
	ext{DistanceCumul}_i \le d_{max}^k
\]

---

## Comentários sobre a modelagem

- A formulação clássica (fluxo + pareamento) é NP-hard; para instâncias de médio/grande porte recorremos a algoritmos especializados (heurísticas, metaheurísticas) ou ao solver de roteamento do OR-Tools que combina CP e heurísticas.
- OR-Tools usa uma estrutura de índices (RoutingIndexManager + RoutingModel). Em vez de manipular explicitamente \(x_{uv}^k\), trabalha-se com índices de roteamento e `Dimensions` (cumulativas) para restrições como capacidade e tempo.
- Relaxações possíveis: relaxar integridade das decisões de rota não é trivial no contexto do RoutingModel; alternativas incluem formular como PL inteira (fluxo) e relaxar a integralidade, mas isso pode perder escalabilidade prática para VRP.
- Linearizações: para variar (time windows, penalidades), pode-se usar variáveis auxiliares; OR-Tools já oferece suporte direto a time windows e dimensões cumulativas.

---

## Dados de entrada

Formato do arquivo: `data/data.json` (JSON). Chaves esperadas:

- `"distance_matrix"`: matriz NxN (lista de listas) com custos \(c_{uv}\).
- `"pickups_deliveries"`: lista de pares `[pickup_node, delivery_node]` (nós referenciados por índices 0..N-1).
- `"demands"`: lista de inteiros (tamanho N), demandas positivas em pickups e negativas nas entregas correspondentes.
- `"num_vehicles"`: número de veículos (inteiro).
- `"depot"`: índice do depósito (normalmente 0).
- `"vehicle_capacities"`: lista com a capacidade de cada veículo.

**Exemplo (trecho do `data/data.json` de teste):**
```json
{
  "distance_matrix": [
    [0, 9, 9, 9, 9, 14, 14, 14, 14],
    [9, 0, 2, 4, 6, 10, 10, 10, 10],
    [9, 2, 0, 8, 2, 9, 9, 9, 9],
    [9, 4, 8, 0, 6, 8, 8, 8, 8],
    [9, 6, 2, 6, 0, 7, 7, 7, 7],
    [14,10,9,8,7,0,2,4,6],
    [14,10,9,8,7,2,0,8,4],
    [14,10,9,8,7,4,8,0,2],
    [14,10,9,8,7,6,4,2,0]
  ],
  "pickups_deliveries": [
    [1, 5],
    [2, 6],
    [3, 7],
    [4, 8]
  ],
  "demands": [0,1,1,1,1,-1,-1,-1,-1],
  "num_vehicles": 2,
  "depot": 0,
  "vehicle_capacities": [2,2]
}
```

---

## Solver escolhido e justificativa

**OR-Tools (Routing + CP-SAT components)** — justifica-se por:

- Implementações específicas para VRP/PDVRP (RoutingModel, AddPickupAndDelivery, Dimensions).
- Suporte a restrições comuns (capacidades, time windows, precedência) e heurísticas eficientes (PATH_CHEAPEST_ARC, GUIDED_LOCAL_SEARCH, etc.).
- Boa escalabilidade prática para instâncias reais e facilidade de integração com Python.

> Alternativas: modelar como PL/inteira com PuLP/Pyomo e resolver com CBC/Gurobi para pequenas/medianas instâncias; porém para VRP com precedências o OR-Tools costuma ser mais direto e eficiente.

---

## Estratégia de implementação

Arquivos principais:

- `solver.py` — script que:
  - carrega `data/data.json`;
  - cria `RoutingIndexManager` e `RoutingModel`;
  - registra callbacks (distância, demanda);
  - adiciona dimensões (Capacity) via `AddDimensionWithVehicleCapacity`;
  - adiciona pares `AddPickupAndDelivery` e restrições auxiliares;
  - configura `search_parameters` (first_solution_strategy, time_limit);
  - resolve e imprime rotas e cargas cumulativas.
- `notebooks/solucao.ipynb` — notebook mínimo para reproduzir execução e visualizar dados/saída.
- `data/` — arquivos JSON de instância.

Verificações / testes:
- Testar integridade do JSON (dimensões corretas da matriz).
- Garantir `len(demands) == N` e que `pickups_deliveries` referenciam índices válidos.
- Testes em instâncias pequenas (sanity checks) antes de escalar.

---

## Como executar (comandos)

1. Criar/ativar virtualenv (recomendado):
```bash
python -m venv .venv
source .venv/bin/activate   # Linux/Mac
.venv\Scriptsctivate      # Windows
```

2. Instalar dependências:
```bash
pip install -r requirements.txt
# requirements.txt contém: ortools>=9.0
```

3. Executar solver com a instância padrão:
```bash
python solver.py data/data.json
```

4. (Opcional) Executar notebook:
```bash
jupyter notebook notebooks/solucao.ipynb
```

---

## Resultados esperados e discussão

- Saída textual com as rotas para cada veículo, carga cumulativa nas visitas e distância total mínima encontrada.
- Para instâncias pequenas, o solver deve encontrar solução ótima ou uma heurística de qualidade em poucos segundos. Para instâncias maiores, ajustar `search_parameters` (estratégias de busca e time_limit) para equilibrar tempo/qualidade.
- Sensibilidades importantes:
  - reduzir capacidade dos veículos pode aumentar a necessidade de mais veículos e aumentar custo total;
  - custos assimétricos na matriz podem alterar alocação de pares a veículos diferentes;
  - adicionar time windows frequentemente torna o problema mais restritivo e pode aumentar tempo de solução.

Métricas a reportar:
- Distância total;
- Número de veículos utilizados (algumas vezes menos que `num_vehicles` disponível);
- Carga máxima observada por veículo;
- Tempo de execução.

---

## Comentários finais e extensões possíveis

- Adicionar **time windows** (`time` dimension) para cada nó.
- Tornar `distance_matrix` derivada de coordenadas Euclidianas (gerador de instâncias).
- Implementar visualização (mapa/plot) das rotas no notebook usando `matplotlib`.
- Produzir CSV/JSON estruturado com as rotas para integração com outros sistemas.

---

## Referências / bibliografia

- OR-Tools — *https://developers.google.com/optimization/routing/pickup_delivery?hl=pt-br*
- Código referência - *https://github.com/google/or-tools/blob/stable/ortools/constraint_solver/samples/vrp_pickup_delivery.py*
- Link do Binder - **