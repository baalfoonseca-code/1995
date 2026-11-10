# CODEX — ESPECIFICAÇÃO CONSOLIDADA E AUDITÁVEL
## Scalper 1.995 | Gatilho ABS no preço do sinal + continuação por pullback

**Tipo:** ordem de implementação MQL5, com testes e critérios de bloqueio.  
**Estado de origem:** candidato experimental 1.995; não compilado nem homologado no material de referência.  
**Prioridade:** precisão causal, preservação financeira, rastreabilidade, ausência de operações retroativas.

### 0. IDENTIDADE DO FONTE — REGRA DE BLOQUEIO ANTES DE EDITAR

Atue como engenheiro MQL5 sênior e auditor de estratégias de microestrutura para WIN/IND e WDO/DOL. Execute mudanças reais em fonte MQL5 e entregue diff, testes e evidências; não apenas opiniões.

**Não presumir que o nome do repositório determina a versão.** Os repositórios `baalfoonseca-code/scalper1995` e `baalfoonseca-code/scalper-1995` foram inspecionados, e os arquivos principais de ambos declaram `#property version "1.122"` e têm o mesmo SHA de blob (`ecf9e6f59e4111de2e9379645d2014a0a53f933e`). Não adaptar esse fonte como se já fosse o 1.995.

O PDF `scalper 1995 (1).pdf` descreve um **candidato** que declara `#property version "1.995"`, mas é uma transcrição, não evidência de compilação. Localize no ambiente um `.mq5` correspondente ao 1.995 — preferencialmente o fonte exato do executável operado — e confronte funções, inputs, SHA/revisão e lógica com o PDF. `Scalper1995_EntradaNoSinal.mq5`, se fornecido, é **outro candidato**, possivelmente já alterado; verificar diffs e sua procedência antes de usá-lo.

**Se não existir fonte `.mq5` 1.995 verificável, interromper alterações de código financeiro e solicitar o artefato exato.** Não reconstruir a versão 1.995 a partir da 1.122, não fazer engenharia reversa fictícia do PDF e não editar outro projeto denominado “Hammer” sem prova documental da identidade. Preservar a fonte original intacta, criar branch/cópia e registrar hash antes/depois. Não fazer deploy real automaticamente.

### 1. OBJETIVO FUNCIONAL E LIMITES DESTA ENTREGA

Implementar **duas alterações independentes**, em commits/diffs separados:

1. **ABS / sinal → gatilho:** preço de referência imutável de um negócio real elegível observado quando a tese é reconhecida, não máxima/mínima de N candles.
2. **CONT / tese → pullback → confirmação:** permitir entrada de continuação apenas após recuo à metade favorável da última vela M1 fechada, com confirmação de preço inteiramente dentro dessa zona.

Preservar nesta entrega a matriz original C1–C5 e seu **relógio de reconhecimento atual baseado na M1 fechada**. Depois da tese, os estados de execução podem reagir a negócios posteriores durante a vela seguinte. **Não prometer identificação de novos sinais no começo/meio da vela M1 nesta entrega.** Uma futura **Fase 2: matriz intrabar causal** será somente auditada e especificada aqui, não implementada nem declarada concluída. Para realizá-la seria necessária revisão explícita de C1–C5, VWAP, Weis Wave, POC, caches, clocks e lookahead.

Respeitar `Inp_ContextTF`: no PDF 1.995 o padrão é `PERIOD_M5`, embora comentários herdados mencionem M3. Não trocar timeframes incidentais.

### 2. INPUTS NOVOS E COMPATIBILIDADE

Criar controles independentes com nomes estáveis e identificáveis:

```cpp
enum ABS_TRIGGER_REFERENCE
{
   ABS_TRIGGER_LEGACY_REGION = 0,
   ABS_TRIGGER_SIGNAL_TRADE = 1
};

input ABS_TRIGGER_REFERENCE Inp_ABSTriggerReference =
   ABS_TRIGGER_SIGNAL_TRADE;

input bool   Inp_CONT_EnablePullback = true;
input double Inp_CONT_PullbackZonePct = 50.0;   // 0 < pct <= 50
input int    Inp_AnchorCaptureTimeoutMs = 2000; // hipótese inicial de engenharia, TESTAR
input double Inp_BreakoutMargin_WIN = 175.0;
input double Inp_BreakoutMargin_WDO = 3.5;
```

`Inp_AnchorCaptureTimeoutMs=2000` é um **limite experimental de espera**, não parâmetro já validado. Não o otimizar com resultados do mesmo conjunto de teste; relatar quantos sinais deixam de receber âncora por esse limite. Se o feed não fornecer relógio confiável suficiente, bloquear, sem extrapolar.

Usar unidades de **pontos de preço do contrato**, como inputs existentes, nunca confundir com `SYMBOL_POINT`, número de ticks, ou dinheiro por ponto. Validar `SYMBOL_TRADE_TICK_SIZE`, precisão e arredondamento; os defaults alterados não são os do PDF antigo (100 WIN / 2,5 WDO). Para regressão **congelar os valores históricos reais do baseline**, e testar 175/3,5 separadamente.

Modos:
- ABS LEGACY: reproduz integralmente o cálculo original pelos extremos, sem aplicar exigências novas de captura de negócio.
- ABS SIGNAL_TRADE: nova âncora de negócio e margem fixa; nunca cair silenciosamente em LEGACY.
- CONT pullback OFF: continua exatamente a CONT original, inclusive gatilho estrutural, **independente** do modo escolhido para ABS.
- CONT pullback ON: segue apenas a lógica congelada da zona; nunca cair silenciosamente na CONT original.
- Criar telemetria explicitando a modalidade por sinal e versão dos parâmetros.

### 3. IDENTIDADE DO NEGÓCIO E DO RELÓGIO

Não usar `SymbolInfoTick().last` só porque é positivo ou possui `time_msc`: em tick **apenas de Bid/Ask**, esse campo pode carregar o **Last de um negócio anterior**.

Para captura e confirmação, exigir evento de negócio real verificável pelas flags relevantes (`TICK_FLAG_LAST`, `TICK_FLAG_VOLUME`, `TICK_FLAG_BUY`, `TICK_FLAG_SELL`) e/ou `COPY_TICKS_TRADE`, com volume válido, preço positivo/finito/em grade, timestamp confiável e evidência de ordem temporal. Flags sozinhas não dispensam verificação de consistência. Não inferir agressão BUY/SELL quando desconhecida; para preço, não exigir classificação de agressor se o negócio é comprovado.

Separar claramente:
- `thesisRecognizedMsc`: instante em que a matriz fechada produziu a tese;
- `signalAnchorTradeMsc`: primeiro negócio válido **observado** após a tese;
- `confirmationTradeMsc`: negócio válido, distinto e posterior que tocou o gatilho;
- `dispatchQuoteMsc`: cotação Bid/Ask de execução;
- `fillDealMsc`, preço/volume: somente confirmação do servidor.

**Estado WAIT_ANCHOR para ABS.** Se a tese aparece em tick de cotação sem negócio novo, não usar Last defasado nem cancelar instantaneamente a tese. Aguardar o **primeiro negócio comprovado posterior**, dentro de `Inp_AnchorCaptureTimeoutMs` e do timeout original do ARMED. Quando chegar, ancorar uma única vez e começar a esperar o deslocamento. Se a espera vencer, cancelar com `SIGNAL_TRADE_UNAVAILABLE`/`ANCHOR_TIMEOUT`, não mover a âncora para um negócio arbitrariamente tardio.

Não selecionar retroativamente um negócio que existia **antes do reconhecimento** da tese nem construir um gatilho atingido no passado. Quando a matriz for confirmada no início de uma nova M1, documentar que a referência pertence ao **primeiro trade após reconhecimento**, não necessariamente ao fechamento da barra anterior. A diferença temporal e de preço deve ser auditada.

Um negócio de confirmação deve ter identificação inequivocamente posterior ao da âncora. Se vários negócios partilham `time_msc`, aceitar mesmo-milissegundo apenas se houver sequência comprovável; caso contrário, esperar timestamp maior. Evitar reprocessar o mesmo registro.

`armed.armedAt`/timeout original começa na **tese**, não reinicia ao capturar âncora, entrar em pullback, confirmar, mudar de barra ou reconectar. Manter identidade única do ciclo e proteção contra ordens duplicadas.

### 4. ABS — GATILHO DEFINIDO PELO PREÇO CONGELADO

No modo novo, após ancorar um negócio real validado:

```text
BUY  : T = RoundUpTick(anchorTradePrice + marginPricePoints)
SELL : T = RoundDownTick(anchorTradePrice - marginPricePoints)
```

Arredondar para garantir **pelo menos** a margem solicitada, nunca abaixo dela, usando `SYMBOL_TRADE_TICK_SIZE`. Congelar `anchorTradePrice`, `anchorTradeMsc`, `direction`, `route`, `margin`, `T` e ID até terminar esse sinal. Não recalcular T sobre cada tick/candle.

Não usar `armed.regionHigh/regionLow` para determinar T em ABS novo. Preservar esses extremos **somente se** forem necessários para outros controles estruturais, FAR ou diagnósticos.

Disparo BUY exige negócio posterior `Last >= T`; SELL exige negócio posterior `Last <= T`. Revalidar o lado executável (Ask para compra, Bid para venda), spread, overshoot, direção viva, Weis Wave, VWAP, posição, exposição, limites, stops e estado financeiro, inclusive imediatamente antes do despacho.

**Caso de teste obrigatório:**
- Sinal ABS BUY WDO em `5000.0`, máxima de vela anterior `5010.0`, margem `3.5`: gatilho é `5003.5`, sem exigir `5010.0`.
- Espelhar em venda e no WIN com `175` pontos.

### 5. CONT — PULLBACK NA METADE FAVORÁVEL DA M1 ANTERIOR

Ao reconhecer tese CONT válida pela matriz original, congelar `H=high`, `L=low`, `barTime` da **última M1 fechada** disponível nesse evento. Não escolher uma nova M1 enquanto o sinal aguarda. Exigir H e L finitos, na grade, `H > L` e barra temporalmente encerrada.

Definir `f = Inp_CONT_PullbackZonePct/100`, exigindo `0 < f <= 0.5`.

```text
CONT BUY:
  Zbuy_low = L
  Zbuy_high = RoundDownTick(L + f*(H-L))
  Trade de pullback P deve satisfazer: L <= P <= Zbuy_high
  T = RoundUpTick(P + margem)
  Exigir: T <= Zbuy_high

CONT SELL:
  Zsell_low = RoundUpTick(H - f*(H-L))
  Zsell_high = H
  Trade de pullback P deve satisfazer: Zsell_low <= P <= H
  T = RoundDownTick(P - margem)
  Exigir: T >= Zsell_low
```

Em 50%, trata-se da metade inferior para BUY e metade superior para SELL. A referência é a **amplitude total high-low**, não apenas o corpo.

**Validação de viabilidade:** se `f*(H-L) < margem` (incluindo o arredondamento da grade), registrar `CONT_ZONE_TOO_NARROW` e não armar pullback. Com f=0,5 e margens novas, H-L **menor que 350 pontos WIN** ou **menor que 7 pontos WDO** é inviável matematicamente; igualdade não garante execução, sobretudo com spread.

**Máquina de estados:**
- `WAIT_PULLBACK`: aguardar o primeiro negócio posterior à tese que esteja dentro da zona e deixe espaço suficiente para a confirmação. Para BUY, caso negócio posterior fique `Last < L`, cancelar `CONT_ADVERSE_EXTREME`; para SELL, `Last > H`, idem. Ausência de toque: esperar até timeout original.
- `WAIT_CONFIRMATION`: no primeiro negócio elegível de pullback, congelar P, timestamp e T. Exigir **negócio distinto e posterior** na direção da entrada e **dentro da zona**, BUY `T <= Last <= Zbuy_high`, SELL `Zsell_low <= Last <= T`. Se negócio passar além do limite favorável, cancelar `CONT_ZONE_BROKEN`, não perseguir. Sem gatilho móvel, recaptura ou reset de prazo.
- `READY_TO_DISPATCH`: somente se todos os filtros e também preço executável couberem na mesma zona. Revalidar atômica e imediatamente antes de enviar. Se não for possível, não despachar; manter/cancelar segundo prazo e eventos, sem inventar fill.

A compra precisa satisfazer `T <= Ask <= Zbuy_high`; a venda, `Zsell_low <= Bid <= T` **na cotação de despacho**, além do requisito sobre Last e das proteções existentes. Não converter um toque de preço no pullback em ordem imediata. Não baixar o trigger, expandir a zona ou trocar a vela para forçar trades.

**Caso de teste:** WDO `L=5000.0, H=5010.0`, BUY P=`5001.5`, margem=`3.5`, T=`5005.0`; negócio Last=`5005.0` com Ask=`5005.5` **bloqueia despacho**. Testar venda simétrica e WIN 175.

### 6. FILTROS CONT, FAR E REGIÃO — SEM MUDANÇA DE TESE

A matriz C1–C5 continua decidindo **direção/rota**. O pullback não inverte direção, VWAP nem C5. No ponto já existente de verificação de entrada, manter Weis Wave viva, glass, VWAP e `Inp_EnableLateDirectionBlock`.

Especificamente, no 1.995 o filtro tardio CONT quando habilitado exige:
- CONT BUY: `livePosRel >= Inp_VWAP_Inner_Pct`;
- CONT SELL: `livePosRel <= 1 - Inp_VWAP_Inner_Pct`.

Esses critérios podem ser incompatíveis com a nova zona de 50%. Não relaxá-los. Registrar `CONT_LATE_DIRECTION_CONFLICT` e, quando possível, determinar se a interseção com a zona era vazia. Medir frequência dos bloqueios.

Preservar `FAR_OFF`, `FAR_OBSERVAR`, `FAR_APLICAR`, seu veto de contradição e a tese original. Os campos `armed.regionHigh/Low` podem seguir necessários ao FAR para invalidar estrutura — **não os deletar por engano**. O sistema ABS SIGNAL_TRADE **não** pode depender de `farArmPrice`, pois FAR pode estar desligado.

Revisar todos os consumidores de `ComputeTrigger`, `armed.regionHigh/Low`, `armed.armedAt`, `FARCaptureArm`, `FAREvaluate`, `FARBeforeDecision`, `SendOrder` e telemetria. Um gatilho ainda indisponível em `WAIT_ANCHOR`/`WAIT_PULLBACK` não pode ser representado por zero e interpretado como rompimento no FAR ou no despacho. Distinguir trigger original e trigger novo na telemetria; preservar comportamento financeiro do FAR conforme modo.

### 7. DISTINÇÃO CRÍTICA: PREÇO NA COTAÇÃO ≠ PREÇO EFETIVAMENTE EXECUTADO

No fluxo atual com `trade.PositionOpen(...)` a mercado, validar Ask/Bid até o último instante **não garante**, por si só, que o negócio final será executado dentro da zona. FOK refere-se principalmente à condição de preenchimento de **volume**, não a garantia universal de preço-limite.

Entregar **dois resultados explicitamente distintos**:
1. **Guard de despacho**: implementação verificável que só envia quando cotação executável e demais filtros estão dentro da zona. Pode ser usada como experimento em tester, mas **não** pode ser anunciada como garantia de fill estrito.
2. **Garantia de fill dentro da zona**: somente alegar se o protocolo de ordens da corretora/símbolo e o tipo de ordem realmente impondo limite de preço forem verificados. Se exigir mudança para ordem limitada ou protocolo de entrada diferente, **não alterar silenciosamente** a máquina financeira. Descrever o trabalho separado, deixar modalidade estrita **não homologada para uso real** e solicitar autorização específica antes de mudar o tipo de ordem.

Registrar preenchimentos reais (deal, preço, volume, timestamp, requisição) e incidentes de fill fora da zona. Proteger SL/TP, sizing, desvio, margem, journal, reinício, cancelamentos, expiração e retcodes. Não simular fill dentro de zona por inferência.

### 8. INVARIANTES FINANCEIROS E DE SEGURANÇA

Manter integralmente:
- sizing por lote fixo/percentual; multiplicador Turbo; cálculo de margem, SL, TP, BE e progressão sem TP;
- limites diários, perdas consecutivas, quotas, cooldown, sessão e EOD;
- `HasOpenPosition`, ordens em trânsito, proteção contra execução duplicada, netting/exchange e preenchimento FOK permitido;
- identidade de ordens, journal durável de preparação/despacho, recuperação/reconciliação e falhas fail-closed;
- controles VWAP/Weis/POC/C1–C5, late direction/glass, FAR e contradição forte;
- código de produção legado intocado e arquivos históricos CSV/journals intocados.

Se um invariante exigir alteração financeira adicional, **não supor autorização**. Relatar bloqueio técnico e manter o caminho novo indisponível até solução comprovada.

### 9. TELEMETRIA NOVA, SEM MUDAR JOURNALS HISTÓRICOS

Criar schema diagnóstico separado, versionado e em arquivo novo, vinculando eventos por ID único composto de instância/reinício/sinal; não anexar colunas ao journal financeiro existente.

Campos mínimos:

```text
BUILD_ID,BASELINE_SHA,SIGNAL_ID,ACCOUNT_INSTANCE,
THESIS_TIME_MSC,THESIS_SNAPSHOT_CLOSE,ANCHOR_TRADE_TIME_MSC,
ANCHOR_TRADE_PRICE,ANCHOR_VALIDATION_SOURCE,ANCHOR_WAIT_MS,
ROUTE,DIRECTION,TRIGGER_MODE,BREAKOUT_MARGIN,
PREVIOUS_BAR_TIME,PREVIOUS_HIGH,PREVIOUS_LOW,ZONE_FRACTION,
ZONE_LOW,ZONE_HIGH,PULLBACK_TIME_MSC,PULLBACK_TRADE_PRICE,
TRIGGER_PRICE,CONFIRMATION_TRADE_TIME_MSC,CONFIRMATION_LAST,
DISPATCH_QUOTE_TIME_MSC,DISPATCH_ASK,DISPATCH_BID,
REQUEST_PRICE,FILL_DEAL_ID,FILL_PRICE,FILL_VOLUME,FILL_TIME_MSC,
DISTANCE_ANCHOR_TO_TRIGGER,DISTANCE_THESIS_TO_FILL,
FAR_STATE,READINESS_STATE,BLOCK_REASON
```

**Não** registrar `FILL_PRICE`/`ENTRY_PRICE` como gatilho, Last ou Ask/Bid quando não houver negócio de preenchimento confirmado pelo servidor. Medir sinais sem âncora, tempo de âncora, recuos perdidos, zona estreita, conflitos VWAP, spread, overshoot, expiração, quote desatualizada e fills fora da zona.

### 10. TESTES DETERMINÍSTICOS E COMPARAÇÃO CAUSAL

Exigir testes com ticks de negócio e cotação reproduzíveis, inclusive:
1. Negócio ausente em tick só de quote; WAIT_ANCHOR captura primeiro trade válido posterior ou expira.
2. Mesmo negócio não pode ser âncora e confirmação; mesmo milissegundo sem sequência comprovável não serve.
3. ABS com pavio anterior enorme, sem impacto em T; compra/venda e diferenças de tick-grid.
4. Referência WDO 5000 + 3,5 = 5003,5 apesar de high anterior 5010; espelho WIN 175.
5. CONT zona válida e inválida, igualdade da amplitude, faixa e midpoint fora da grade.
6. CONT Last dentro da zona, mas Ask/Bid fora; não envia.
7. CONT preço move além da zona; não persegue; ultrapassa extremo adverso; cancela.
8. Late-direction e VWAP incompatíveis com zona; bloqueio registrado, sem desligar filtro.
9. FAR OFF/OBS/APLICAR, contradição dura e alteração da barra durante sinal armado.
10. Dois sinais com mesmo timestamp de quote, reconexão/restart e journals pendentes; não duplica pedido.
11. Salto sobre trigger, overshoot, spread, disponibilidade, stop-level, retcodes e ordem em voo.
12. Verificação separada de **cotação no despacho** e **fill real**.
13. Paridade exacta da modalidade 100% LEGACY com o baseline compilável correto, usando valores originais, seed, ticks, custos e sessão iguais.
14. Nunca usar tick futuro, fechamento de barra viva, nem resultado de backtest para decidir dinamicamente parâmetros.

Comparar em **quatro ablações** no mesmo fluxo tick a tick, sem mudar vários elementos simultaneamente:
- A. baseline integral: ABS LEGACY + CONT original, inputs e dados históricos iguais;
- B. só ABS SIGNAL_TRADE, CONT original;
- C. ABS LEGACY, só CONT com pullback;
- D. ABS SIGNAL_TRADE + CONT pullback.

Executar separadamente uma análise de sensibilidade dos novos defaults WIN 175/WDO 3,5 (não misturá-la com A–D). Relatar número de teses, tempo até âncora, taxa de execução, recuo, dispersão da distância, MAE/MFE, stop-outs, PnL líquido, expectativa R, DD, custos/slippage, razão dos descartes, testes fora da amostra e incerteza. Não atribuir causalidade ao “preço melhor” sem contrafactual e amostra suficiente.

### 11. FASE 2 — AUDITORIA INTRABAR, SEM IMPLEMENTAÇÃO NESTE COMMIT

Documentar tudo o que impede a matriz de reconhecer sinais no meio de M1. Em particular examinar `ComputeMatrixSnapshot` (fechamento `iClose(...,1)`), `SnapshotForSignal`, `EvaluateMatrix`, caches, `OnTick`, VWAP, Weis Wave, POC, `Inp_ContextTF` configurável e condições C1–C5.

Produzir plano isolado para sinais intrabar **causais**, que inclua controles contra repintura e duplicação e especificação de testes. Não modificar essa matriz no mesmo commit das duas mudanças de execução. Não afirmar que “gatilho por tick” equivale a “sinal intrabar”.

### 12. CRITÉRIOS OBJETIVOS DE ACEITAÇÃO E RELATÓRIO

Só marcar a entrega como concluída se:

- Fonte 1.995 correto e hash/revisão estiverem comprovados.
- ABS novo congelar **negócio real posterior à tese**, sem máximas/mínimas no gatilho.
- ABS LEGACY reproduzir algoritmo original com inputs históricos.
- CONT pullback ON usar vela M1 fechada única, zona e confirmação, sem exceções tácitas.
- CONT pullback OFF reproduzir CONT original.
- Timeout não reiniciar; não houver dados futuros nem ordem duplicada.
- Verificações do Last, cotação executável e fill real forem registradas separadamente.
- No mínimo, guard de despacho obedecer zona; nenhum fill estrito for prometido sem protocolo comprovado.
- FAR, contabilidade, ordem, sizing, risco e segurança permanecerem sob testes de regressão.
- Mudanças estejam isoladas em commits/diffs, com logs e arquivos de teste reprodutíveis.
- Haja relatório de compilação **nativa MetaEditor**, se disponível. Sem MetaEditor, declarar “NÃO COMPILADO NATIVAMENTE”, e não dizer “homologado”.
- Resultado da Fase 2 intrabar esteja descrito **como auditoria futura, não como funcionalidade entregue**.

**Saída obrigatória:** arquivos `.mq5` modificados (ou impedimento concreto de fonte), hashes, diff completo, tabela dos novos inputs e defaults, lista de caminhos alterados, bateria de testes com resultado por caso, limitações de execução/retcodes, relatório comparativo A–D e decisão final explícita: `IMPLEMENTADO E TESTADO`, `IMPLEMENTADO NÃO COMPILADO`, `PARCIAL` ou `BLOQUEADO`.

**REGRA FINAL:** execute o código somente se houver fonte 1.995 correto. Nunca substitua a necessidade de evidência por afirmações de resultado, nem declare proteção absoluta dos 50% quando o fill puder escapar da zona.

### Referências técnicas para conferir

- MQL5 `MqlTick` e flags de atualização: https://www.mql5.com/pt/docs/constants/structures/mqltick
- MQL5 `CopyTicks` e classes de eventos: https://www.mql5.com/en/docs/series/copyticks
- MQL5 modalidades `FOK`, `IOC` e limites: https://www.mql5.com/pt/docs/constants/tradingconstants/orderproperties
