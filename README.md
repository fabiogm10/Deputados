# 📊 Projeto Deputados - Análise de Dados Abertos da Câmara

Este projeto realiza a coleta, processamento e análise de dados abertos da Câmara dos Deputados, focando nas despesas parlamentares, votações e informações cadastrais dos deputados.

## 🏗️ Arquitetura do Projeto

### Camadas de Dados
- **Bronze**: Dados brutos coletados da API e arquivos CSV
- **Silver**: Dados tratados e enriquecidos
- **Gold**: Tabelas fato e dimensões para análise

### Estrutura de Pastas no Databricks
```
/Workspace/Users/fabiogm10@gmail.com/Deputados/
├── Bronze/           # Dados brutos em JSON e CSV
├── Despesas_2025/    # Despesas específicas do ano
├── Votacoes/         # Dados de votações em CSV
├── Tables/           # Tabelas Delta
└── Volume: /Volumes/workspace/deputados/deputados/
    └── gold/         # Camada gold (fdespesas_deputados, fvotacoes)
```

## 📁 Notebooks do Projeto

### 1. 🎯 Coleta de Dados dos Deputados
**Objetivo**: Coletar dados cadastrais de todos os deputados da API da Câmara

**Funcionalidades**:
- Busca dados da API: `https://dadosabertos.camara.leg.br/api/v2/deputados`
- Salva em JSON na camada Bronze
- Tratamento de paginação automática
- Geração de timestamps para versionamento

**Saída**: `deputados_YYYYMMDD_HHMMSS.json`

### 2. 💰 Coleta de Despesas dos Deputados
**Objetivo**: Coletar despesas detalhadas de cada deputado para 2025

**Funcionalidades**:
- Loop através de IDs de deputados coletados
- Busca despesas por deputado: `https://dadosabertos.camara.leg.br/api/v2/deputados/{id}/despesas?ano=2025`
- Tratamento de paginação para deputados com muitas despesas
- Salva despesas individuais e consolidadas
- Delay entre requisições para não sobrecarregar a API

**Saída**: `despesas_consolidadas_YYYYMMDD_HHMMSS.json`

### 3. 📊 Processamento de Dados de Votações
**Objetivo**: Processar arquivos CSV com dados de votações dos deputados

**Funcionalidades**:
- Leitura de arquivos CSV da pasta Votacoes
- Tratamento de encoding e delimitadores
- Padronização de colunas e tipos de dados
- Enriquecimento com dados cadastrais dos deputados
- Criação de tabela Delta com métricas de votação

**Fontes de Dados**:
- Arquivos CSV com histórico de votações
- Estrutura típica: `id_votacao, id_deputado, voto, data_votacao, proposicao`

**Saída**: Tabela `workspace.deputados.silver_votacoes`

### 4. 🗃️ Processamento e Criação de Tabelas Delta
**Objetivo**: Transformar dados JSON e CSV em tabelas Delta otimizadas

**Funcionalidades**:

#### Tabela Silver - Despesas
```python
# Schema principal
- id_deputado, ano, mes, tipo_despesa, valor_liquido
- cnpj_cpf_fornecedor, nome_fornecedor, data_documento
- ano_mes (partição), data_processamento
```

#### Tabela Silver - Cadastro Deputados
```python
# Schema principal  
- id, nome, siglaPartido, siglaUf, uri, email
- idLegislatura, urlFoto, uriPartido
```

#### Tabela Silver - Votações
```python
# Schema principal
- id_votacao, id_deputado, voto, data_votacao
- proposicao, tipo_votacao, resultado
- ano, mes, ano_mes (partição)
```

#### Tabela Gold - Fato Despesas
```python
# Agregações e métricas de negócio
- Dimensões: deputado, tempo, despesa, fornecedor
- Métricas: valor_liquido, valor_glosa, total_registros
- Indicadores: percentual_glosa, valor_medio_despesa
- Partição: ano_mes para otimização
```

#### Tabela Gold - Fato Votações
```python
# Agregações de comportamento de voto
- Dimensões: deputado, tempo, proposicao, tipo_votacao
- Métricas: total_votos, votos_sim, votos_nao, ausencias
- Indicadores: taxa_presenca, alinhamento_partidario
- Partição: ano_mes para otimização
```

## 🚀 Como Executar

### Pré-requisitos
- Databricks Workspace
- Acesso à API da Câmara dos Deputados
- Arquivos CSV de votações na pasta `/Votacoes/`
- Permissões para criar tabelas Delta

### Ordem de Execução
1. **Preparar dados de votações**
   - Colocar arquivos CSV na pasta `/Workspace/Users/fabiogm10@gmail.com/Deputados/Votacoes/`
   - Formato esperado: CSV com delimitador vírgula ou ponto-e-vírgula

2. **Executar notebook de coleta de deputados**
   ```python
   buscar_deputados()  # Gera arquivos na camada Bronze
   ```

3. **Executar notebook de coleta de despesas**
   ```python
   # Processa automaticamente todos os deputados coletados
   buscar_despesas_com_paginacao()
   ```

4. **Executar notebook de processamento de votações**
   ```python
   # Processa arquivos CSV e cria tabela silver_votacoes
   ```

5. **Executar notebook de processamento geral**
   ```python
   # Cria todas as tabelas Silver e Gold automaticamente
   ```

### Configurações Importantes

#### API Endpoints Utilizados
```python
DEPUTADOS_API = "https://dadosabertos.camara.leg.br/api/v2/deputados"
DESPESAS_API = "https://dadosabertos.camara.leg.br/api/v2/deputados/{id}/despesas"
```

#### Parâmetros de Coleta
```python
params = {
    'ano': 2025,
    'ordem': 'ASC',
    'ordenarPor': 'ano',
    'itens': 100,  # Paginação
    'pagina': 1
}
```

#### Configurações CSV Votações
```python
csv_config = {
    'header': True,
    'inferSchema': True,
    'delimiter': ',',  # ou ';' dependendo do arquivo
    'encoding': 'utf-8'
}
```

## 📊 Estrutura das Tabelas

### Tabela Fato: `fdespesas_deputados`
**Localização**: `/Volumes/workspace/deputados/deputados/gold/fdespesas_deputados`

**Principais Campos**:
- **Dimensões**: `id_deputado`, `ano_mes`, `tipo_despesa`, `cnpj_cpf_fornecedor`
- **Métricas**: `valor_liquido`, `valor_glosa`, `total_registros`
- **Indicadores**: `percentual_glosa`, `valor_medio_despesa`
- **Controle**: `data_criacao_registro`

### Tabela Fato: `fvotacoes_deputados`
**Localização**: `/Volumes/workspace/deputados/deputados/gold/fvotacoes_deputados`

**Principais Campos**:
- **Dimensões**: `id_deputado`, `ano_mes`, `proposicao`, `tipo_votacao`
- **Métricas**: `total_votos`, `votos_sim`, `votos_nao`, `ausencias`
- **Indicadores**: `taxa_presenca`, `indice_aprovação`
- **Controle**: `data_processamento`

**Particionamento**: `ano_mes` para otimização de consultas

## 🔍 Exemplos de Consultas

### Análise de Despesas por Tipo
```sql
SELECT 
    tipo_despesa,
    SUM(valor_liquido) as total_gasto,
    COUNT(*) as quantidade
FROM workspace.deputados.fato_despesas_deputados
WHERE ano = 2025
GROUP BY tipo_despesa
ORDER BY total_gasto DESC
```

### Comportamento de Votação por Partido
```sql
SELECT 
    cad.sigla_partido,
    COUNT(*) as total_votacoes,
    SUM(CASE WHEN vot.voto = 'Sim' THEN 1 ELSE 0 END) as votos_sim,
    SUM(CASE WHEN vot.voto = 'Não' THEN 1 ELSE 0 END) as votos_nao,
    ROUND(AVG(CASE WHEN vot.voto = 'Sim' THEN 1 ELSE 0 END) * 100, 2) as perc_aprovacao
FROM workspace.deputados.silver_votacoes vot
JOIN workspace.deputados.silver_cadastro_deputados cad
    ON vot.id_deputado = cad.id
GROUP BY cad.sigla_partido
ORDER BY total_votacoes DESC
```

### Cross Analysis: Despesas vs. Presença em Votações
```sql
SELECT 
    desp.id_deputado,
    cad.nome_deputado,
    cad.sigla_partido,
    SUM(desp.valor_liquido) as total_despesas,
    COUNT(vot.id_votacao) as total_votacoes,
    SUM(CASE WHEN vot.voto IS NOT NULL THEN 1 ELSE 0 END) as votos_presentes,
    ROUND(SUM(CASE WHEN vot.voto IS NOT NULL THEN 1 ELSE 0 END) * 100.0 / COUNT(vot.id_votacao), 2) as taxa_presenca
FROM workspace.deputados.fato_despesas_deputados desp
JOIN workspace.deputados.silver_cadastro_deputados cad
    ON desp.id_deputado = cad.id
LEFT JOIN workspace.deputados.silver_votacoes vot
    ON desp.id_deputado = vot.id_deputado
    AND desp.ano_mes = vot.ano_mes
WHERE desp.ano = 2025
GROUP BY desp.id_deputado, cad.nome_deputado, cad.sigla_partido
ORDER BY total_despesas DESC
```

### Top Deputados por Gasto
```sql
SELECT 
    nome_deputado,
    sigla_uf,
    sigla_partido,
    SUM(valor_liquido) as total_gasto
FROM workspace.deputados.fato_despesas_deputados
WHERE ano_mes = '2025-01'
GROUP BY nome_deputado, sigla_uf, sigla_partido
ORDER BY total_gasto DESC
LIMIT 10
```

## ⚙️ Manutenção e Otimização

### Comandos Úteis
```python
# Otimizar tabelas
spark.sql("OPTIMIZE workspace.deputados.fato_despesas_deputados")
spark.sql("OPTIMIZE workspace.deputados.fato_votacoes_deputados")

# Limpar versões antigas
spark.sql("VACUUM workspace.deputados.fato_despesas_deputados RETAIN 168 HOURS")
spark.sql("VACUUM workspace.deputados.fato_votacoes_deputados RETAIN 168 HOURS")

# Verificar histórico
spark.sql("DESCRIBE HISTORY workspace.deputados.fato_despesas_deputados")
```

### Monitoramento
- Verificar espaço em disco das tabelas
- Monitorar performance de consultas
- Validar consistência dos dados entre camadas
- Acompanhar atualizações dos arquivos CSV de votações

## 🐛 Solução de Problemas

### Erros Comuns
1. **API Rate Limit**: Delay entre requisições já implementado
2. **JSON Corrupt**: Uso de `multiline=true` na leitura
3. **CSV Encoding**: Configuração de encoding UTF-8 ou Latin1
4. **Delimitador CSV**: Verificar se é vírgula ou ponto-e-vírgula
5. **Duplicatas**: Merge operations nas tabelas Delta

### Logs e Debug
- Timestamps em todos os processos
- Contadores de registros processados
- Validações de consistência
- Amostras de dados em cada etapa

## 📈 Próximos Passos

- [ ] Adicionar mais anos históricos
- [ ] Criar dashboards no Databricks SQL
- [ ] Implementar alertas de qualidade de dados
- [ ] Adicionar dados de fornecedores
- [ ] Criar análises de sazonalidade
- [ ] Desenvolver modelo de machine learning para prever comportamento de voto
- [ ] Integrar com dados de redes sociais dos deputados

## 🔗 Links Úteis

- [API Dados Abertos Câmara](https://dadosabertos.camara.leg.br/)
- [Documentação Delta Lake](https://docs.delta.io/)
- [Databricks Documentation](https://docs.databricks.com/)
- [Repositório Dados Abertos](https://dadosabertos.camara.leg.br/)

---

**Desenvolvido por**: Fabio GM  
**Repositório**: https://github.com/fabiogm10/Deputados  
**Última Atualização**: ${data_atual}
