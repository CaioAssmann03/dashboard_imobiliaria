# 🏠 Painel de Locações Imobiliárias — SQL + Power BI

Projeto ponta a ponta de análise de dados: modelagem de um banco de dados relacional,
20 consultas SQL para responder perguntas de negócio, uma auditoria de qualidade de
dados, e um dashboard interativo no Power BI. Inspirado na minha vivência profissional
como Assistente Administrativo em uma imobiliária. Dados fictícios, cenário realista.

---

## 📐 Etapa 1 — Modelagem do banco de dados

Banco relacional com 4 tabelas, construído em SQLite:

```sql
CREATE TABLE imoveis (
    id INTEGER PRIMARY KEY,
    endereco TEXT,
    bairro TEXT,
    tipo TEXT,           -- apartamento, casa, comercial, cobertura, loja
    valor_aluguel REAL,
    status TEXT          -- disponivel, alugado, manutencao
);

CREATE TABLE proprietarios (
    id INTEGER PRIMARY KEY,
    nome TEXT,
    telefone TEXT,
    imovel_id INTEGER REFERENCES imoveis(id)
);

CREATE TABLE locatarios (
    id INTEGER PRIMARY KEY,
    nome TEXT,
    telefone TEXT
);

CREATE TABLE contratos (
    id INTEGER PRIMARY KEY,
    imovel_id INTEGER REFERENCES imoveis(id),
    locatario_id INTEGER REFERENCES locatarios(id),
    data_inicio DATE,
    data_fim DATE,
    valor_mensal REAL,
    status_pagamento TEXT  -- em_dia, atrasado, quitado
);
```

**Relacionamentos:** `contratos.imovel_id → imoveis.id` · `contratos.locatario_id → locatarios.id` · `proprietarios.imovel_id → imoveis.id`

**Volume de dados:** 150 imóveis · 150 proprietários · 120 locatários · 80 contratos → **500 registros no total**

**Ferramentas:** SQLite + [DB Browser for SQLite](https://sqlitebrowser.org/) para modelagem e inserção dos dados.

---

## 📊 Etapa 2 — Análise exploratória (20 consultas SQL)

### Ocupação e Receita

**1. Taxa de ocupação**
```sql
SELECT
    COUNT(*) AS total_imoveis,
    SUM(CASE WHEN status = 'alugado' THEN 1 ELSE 0 END) AS alugados,
    ROUND(SUM(CASE WHEN status = 'alugado' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS taxa_ocupacao
FROM imoveis;
```
**Resultado:** 97 de 150 imóveis alugados → **64,67% de ocupação**

**2 e 3. Receita mensal (total e por status de pagamento)**
```sql
SELECT status_pagamento, SUM(valor_mensal) AS receita
FROM contratos
WHERE status_pagamento IN ('em_dia','atrasado')
GROUP BY status_pagamento;
```
**Resultado:** **R$ 363.500** em receita mensal ativa — R$ 286.700 em dia + R$ 76.800 em atraso

**4. Percentual de inadimplência**
```sql
SELECT
    COUNT(*) AS contratos_ativos,
    SUM(CASE WHEN status_pagamento = 'atrasado' THEN 1 ELSE 0 END) AS atrasados,
    ROUND(SUM(CASE WHEN status_pagamento = 'atrasado' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS percentual_inadimplencia
FROM contratos
WHERE status_pagamento IN ('em_dia','atrasado');
```
**Resultado:** 16 de 80 contratos ativos em atraso → **20% de inadimplência**

**5. Ticket médio dos contratos ativos**
```sql
SELECT ROUND(AVG(valor_mensal), 2) AS ticket_medio
FROM contratos
WHERE status_pagamento IN ('em_dia','atrasado');
```
**Resultado:** **R$ 4.543,75** por contrato

**6. Evolução dos contratos ao longo dos anos**
```sql
SELECT strftime('%Y', data_inicio) AS ano, COUNT(*) AS contratos
FROM contratos
GROUP BY ano
ORDER BY ano;
```
**Resultado:** 2023 → 2 contratos · 2024 → 38 contratos · 2025 → 40 contratos (crescimento concentrado nos últimos dois anos)

**7. Tempo médio de permanência dos locatários**
```sql
SELECT ROUND(AVG((julianday(data_fim) - julianday(data_inicio)) / 30), 1) AS meses
FROM contratos;
```
**Resultado:** **19 meses** de permanência média por contrato

### Bairros e Tipos de Imóvel

**8. Aluguel médio por bairro** (33 bairros no total) — Top 5 mais caros:
| Bairro | Qtd. imóveis | Aluguel médio |
|---|---|---|
| Bela Vista | 4 | R$ 8.650 |
| Moinhos de Vento | 8 | R$ 8.312,50 |
| Mont'Serrat | 6 | R$ 6.700 |
| Chácara das Pedras | 3 | R$ 6.033,33 |
| Boa Vista | 5 | R$ 5.920 |

**9. Aluguel médio por tipo de imóvel**
```sql
SELECT tipo, ROUND(AVG(valor_aluguel), 2) AS aluguel_medio
FROM imoveis
GROUP BY tipo
ORDER BY aluguel_medio DESC;
```
**Resultado:** cobertura (R$ 10.550) > loja (R$ 9.250) > comercial (R$ 6.447,37) > casa (R$ 5.060) > apartamento (R$ 3.413,89)

**11 e 12. Distribuição geográfica** — Petrópolis e Moinhos de Vento lideram em quantidade de imóveis (8 cada); Petrópolis e Cristal têm mais unidades disponíveis (3 cada)

**13. Quantidade de imóveis por tipo:** apartamento (90) · casa (35) · comercial (19) · cobertura (4) · loja (2)

**14. Quantidade de imóveis por status:** alugado (97) · disponível (49) · manutenção (4)

### Proprietários e Locatários

**15. Receita por proprietário** (Top 5 de 150):
```sql
SELECT p.nome, SUM(c.valor_mensal) AS receita
FROM proprietarios p
JOIN contratos c ON p.imovel_id = c.imovel_id
WHERE c.status_pagamento IN ('em_dia','atrasado')
GROUP BY p.nome
ORDER BY receita DESC;
```
José Roberto Silva (R$ 11.000) · Cristiane Alves (R$ 10.200) · Aline Pereira (R$ 9.900) · Silvia Martins e Paula Martins (R$ 9.800 cada)

**16. Top 10 locatários com maior gasto total**
```sql
SELECT
    l.id, l.nome,
    COUNT(c.id) AS quantidade_contratos,
    SUM(c.valor_mensal) AS valor_total_pago
FROM locatarios l
JOIN contratos c ON l.id = c.locatario_id
GROUP BY l.id, l.nome
ORDER BY valor_total_pago DESC
LIMIT 10;
```
**Resultado:** Gustavo Becker lidera com R$ 11.000/mês. Nenhum locatário tem mais de 1 contrato ativo na base atual.

**17. Top 10 proprietários com maior receita**
```sql
SELECT
    p.id, p.nome,
    COUNT(c.id) AS quantidade_imoveis_alugados,
    SUM(c.valor_mensal) AS receita_total
FROM proprietarios p
JOIN contratos c ON p.imovel_id = c.imovel_id
WHERE c.status_pagamento IN ('em_dia','atrasado')
GROUP BY p.id, p.nome
ORDER BY receita_total DESC
LIMIT 10;
```
**Resultado:** José Roberto Silva lidera com R$ 11.000/mês, seguido por Cristiane Alves (R$ 10.200) e Aline Pereira (R$ 9.900).

### Rankings

**18. Top 3 imóveis mais caros:** cobertura na Rua Pedro Chaves Barcelos, Bela Vista (R$ 11.500) · cobertura na Rua Quintino Bocaiúva, Moinhos de Vento (R$ 11.000) · loja na Rua Padre Chagas, Moinhos de Vento (R$ 10.200)

**19. Maior aluguel entre contratos ativos:** Rua Quintino Bocaiúva, Moinhos de Vento — **R$ 11.000/mês**

**20. Valor total da carteira imobiliária**
```sql
SELECT SUM(valor_aluguel) AS carteira
FROM imoveis;
```
**Resultado:** **R$ 667.550** somando os 150 imóveis (alugados + disponíveis + em manutenção)

---

## 🔎 Etapa 3 — Auditoria de qualidade de dados (achado bônus)

Cruzando as tabelas `imoveis` e `contratos`, encontrei uma inconsistência real: **97 imóveis
estão marcados com status "alugado"**, mas existem **apenas 80 contratos no total** — e, ao
cruzar os IDs, **17 desses imóveis "alugados" não têm nenhum contrato correspondente** no banco.

```sql
SELECT id, endereco, bairro, status
FROM imoveis
WHERE status = 'alugado'
AND id NOT IN (SELECT imovel_id FROM contratos);
```

Isso simula um problema comum em bases reais: um campo de status desatualizado manualmente
que não reflete a fonte de verdade (a tabela de contratos). Esse achado virou o próprio KPI
de alerta no dashboard abaixo.

---

## 📈 Etapa 4 — Dashboard Power BI

<img width="1276" height="718" alt="image" src="https://github.com/user-attachments/assets/44a16b4d-1765-4e9a-b8d1-174dfecdf9d4" />

Depois da análise em SQL, construí um dashboard interativo no Power BI conectado às
mesmas 4 tabelas, para visualização e exploração dos indicadores.

**KPIs em destaque (topo):**
- 🟢 64,67% — Taxa de Ocupação (verde: indicador positivo)
- 🔵 R$ 363,50 Mil — Receita Mensal Ativa
- 🔴 20% — Taxa de Inadimplência (vermelho: alerta)
- 🔴 17 — Imóveis sem contrato (vermelho: alerta — a inconsistência encontrada na auditoria)

**Visuais do painel:**
- Aluguel médio por bairro (barras)
- Distribuição de imóveis alugados por tipo
- Evolução de contratos por ano (2023 a 2025)
- Receita mensal ativa por proprietário
- Filtros interativos: bairro, status do imóvel, status do pagamento, tipo, ano

**Principais medidas DAX:**
```dax
Taxa de Ocupação =
DIVIDE(
    CALCULATE(COUNTROWS(imoveis), imoveis[status] = "alugado"),
    COUNTROWS(imoveis)
)

Receita Mensal Ativa =
CALCULATE(
    SUM(contratos[valor_mensal]),
    contratos[status_pagamento] IN {"em_dia", "atrasado"}
)

Taxa de Inadimplência =
DIVIDE(
    CALCULATE(COUNTROWS(contratos), contratos[status_pagamento] = "atrasado"),
    CALCULATE(COUNTROWS(contratos), contratos[status_pagamento] IN {"em_dia", "atrasado"})
)

Imóveis sem Contrato (Alerta) =
CALCULATE(
    COUNTROWS(imoveis),
    imoveis[status] = "alugado",
    NOT(imoveis[id] IN VALUES(contratos[imovel_id]))
)
```

---

## 🛠️ Tecnologias
SQLite · SQL (JOIN, GROUP BY, CASE WHEN, subqueries, funções de data e agregação) · Power BI (DAX, modelagem de relacionamentos, formatação condicional) · DB Browser for SQLite

## ▶️ Como rodar
1. Baixe `Imobiliaria.db` e `consultas.sql` deste repositório
2. Abra o `.db` no [DB Browser for SQLite](https://sqlitebrowser.org/) ou em [sqliteonline.com](https://sqliteonline.com) para explorar as queries
3. Para o dashboard: importe as tabelas no Power BI Desktop e recrie as medidas DAX listadas acima

## 💡 Aprendizados
- Modelar um banco relacional do zero exige pensar em quais perguntas de negócio ele
  vai precisar responder depois — o schema influencia diretamente quais queries são fáceis
  ou difíceis de escrever
- Cruzar tabelas pra validar consistência é tão importante quanto escrever a query "certa" —
  o achado da auditoria (imóveis sem contrato) só apareceu porque fui além da pergunta óbvia
- Levar o mesmo modelo de dados do SQL para o Power BI reforça o valor de um bom design de
  banco: as mesmas tabelas e relacionamentos serviram para consulta e para visualização, sem
  retrabalho

---

## 👤 Sobre mim
Caio Assmann — estudante de Análise e Desenvolvimento de Sistemas, focado em Dados e Business Intelligence.

📊 [Portfólio](#) · 💼 [LinkedIn](https://www.linkedin.com/in/caio-assmann/) · 📩 caioassmann7@gmail.com
