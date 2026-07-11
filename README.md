# SPTrans Operational Efficiency Analytics 

Análise macroscópica e de  performance da grade horária e frequências do sistema de transporte público de São Paulo (SPTrans), processada diretamente a partir de dados brutos de GTFS.

## Arquitetura e Stack Técnica

O maior desafio deste projeto foi processar matrizes de frequência em nível de segundos sem sobrecarregar a memória local. A solução foi desenhada utilizando um pipeline de engenharia de dados:

*   **Engine de Processamento:** DuckDB (WebAssembly / Local) para consultas analíticas de alta velocidade via SQL e agregação colunar direta na memória.
*   **Formato de Armazenamento:** Parquet (formato binário, colunar e auto-descritivo) eliminando problemas de encoding e delimitadores regionais (`.csv`), reduzindo o tamanho do arquivo final e garantindo a tipagem estrita dos dados para o consumo.
*   **Camada de Business Intelligence:** Power BI para modelagem dimensional, cálculo de contexto via DAX e design de interface.

## Modelagem de Dados & Métricas DAX

Para medir a eficiência e a elasticidade do sistema de transporte sem mascarar os dados por médias globais, foram desenvolvidas métricas baseadas em Pesquisa Operacional:

1.  **Frequência Horária Equivalente:** Calculada na camada de engenharia para converter intervalos secundários (*headways*) em veículos por hora ativa.
2.  **Métrica de Capacidade Residual (% Ociosidade Fora do Pico):** 
    Isola dinamicamente a hora de saturação da linha (Frota de Pico) e calcula o recuo da oferta nos horários de vale, expondo o custo ocioso da operação:

```dax
% Ociosidade Fora do Pico = 
VAR _TabelaHoras = VALUES(sptrans_processado[faixa_horaria])
VAR _VolumeMaximoPico = MAXX(_TabelaHoras, [Total Viagens])
VAR _HoraDoPico = TOPN(1, _TabelaHoras, [Total Viagens], DESC)
VAR _MediaViagensVale = AVERAGEX(FILTER(_TabelaHoras, NOT(sptrans_processado[faixa_horaria] IN _HoraDoPico)), [Total Viagens])
RETURN
IF(_VolumeMaximoPico > 0, DIVIDE(_VolumeMaximoPico - _MediaViagensVale, _VolumeMaximoPico), BLANK())
