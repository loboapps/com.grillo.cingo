# Calculadora de horas de ponto — design

## Objetivo

Página estática única onde o usuário cola um relatório de marcações de ponto
(formato tabulado: Data / Dia Semana / Marcações / Ocorrências) e vê, por dia,
se trabalhou mais ou menos que 8 horas, mais o saldo total do período. Sem
banco de dados, sem persistência — tudo roda em memória no navegador.

## Formato de entrada esperado

Texto colado com colunas separadas por tabulação, uma linha por dia,
primeira linha como cabeçalho (ignorada no parse):

```
Data	Dia Semana	Marcações	Ocorrências
15/06/2026	Segunda-feira	09:03 (O)     12:00 (PA)    13:00 (PA)    18:00 (O)     	
20/06/2026	Sábado		
```

A coluna de marcações contém zero ou mais tokens no formato `HH:MM (LABEL)`,
separados por espaços, onde `LABEL` é `O` (marcação real de entrada/saída) ou
`PA` (pausa/intervalo).

## Regra de cálculo por dia

1. Extrair todos os tokens `HH:MM (O)` e `HH:MM (PA)` da coluna de marcações
   via regex, na ordem em que aparecem.
2. **0 marcações (O):** dia sem dado — não entra no total. Exibido como `—`.
3. **1 marcação (O):** dia incompleto — não entra no total. Exibido como
   `incompleto`.
4. **2+ marcações (O):** horas trabalhadas = última (O) − primeira (O).
   Marcações (O) intermediárias são ignoradas.
   - Se houver um par de (PA) (duas marcações), subtrai a duração do
     intervalo (segunda PA − primeira PA) do total.
5. Diferença do dia = horas trabalhadas − 8h, exibida com sinal (+/−).
6. **Tolerância de 20 minutos:** se `|diferença| <= 20min`, o dia não
   impacta o saldo do período (contribuição = 0). Se `|diferença| > 20min`,
   o valor **inteiro** da diferença conta no saldo — não só o excedente
   acima de 20min (lógica de tolerância de ponto tipo CLT, não dedução).
   A diferença bruta continua exibida na tabela; dias dentro da tolerância
   aparecem com destaque neutro (nem verde nem vermelho) indicando que não
   contam pro saldo.

## Saída

Tabela com colunas: Data | Dia da semana | Horas trabalhadas | Diferença.
Linhas sem dado ou incompletas aparecem visualmente distintas (esmaecidas) e
não contam pro total. Dias dentro da tolerância de 20min aparecem com a
diferença em destaque neutro. Rodapé mostra o saldo total do período: soma
das diferenças **ajustadas pela tolerância** de todos os dias computados.

## Interação

- `<textarea>` para colar o texto + botão "Calcular".
- Botão reprocessa o conteúdo atual da textarea a qualquer momento —
  usuário pode colar um novo período e recalcular sem dar refresh.
- Nenhum dado é salvo entre sessões (sem localStorage, sem backend).

## Implementação

- Um único arquivo `index.html` autocontido: HTML + CSS + JS inline, zero
  dependências, zero build.
- Publicado como site estático no Vercel, puxando do repositório GitHub
  `loboapps/com.grillo.cingo` (branch `main`).

## Fora de escopo

- Persistência de dados (banco, localStorage).
- Autenticação.
- Suporte a formatos de entrada diferentes do relatório tabulado descrito
  acima.
