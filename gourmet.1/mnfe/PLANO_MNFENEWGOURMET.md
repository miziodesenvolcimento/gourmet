# Plano da nova versão — `mnfenewgourmet.bpl`

Projeto paralelo para evoluir a emissão fiscal **sem impactar produção**. O projeto atual (`mnfegourmet.bpl`) **não foi alterado** e serve de base. A nova versão nasce como cópia fiel (mesma arquitetura, mesmos módulos de consumo) e recebe as correções da auditoria de forma **gradual**. Ver achados em [`ANALISE_NFE.md`](ANALISE_NFE.md).

## Mapa de arquivos (criados — produção intacta)

| Projeto atual (produção) | Nova versão (`ufnew*`) |
|---|---|
| `mnfegourmet.dpk` / `.dproj` / `.res` | `mnfenewgourmet.dpk` / `.dproj` / `.res` |
| `ufnfe.pas` / `.dfm` | `ufnewnfe.pas` / `.dfm` |
| `ufnfce.pas` / `.dfm` | `ufnewnfce.pas` / `.dfm` |
| `ufnfecorrecao.pas` / `.dfm` | `ufnewnfecorrecao.pas` / `.dfm` |
| `Ping2.pas` | `ufnewping2.pas` |
| `ufaguarde.pas` / `.dfm` | `ufnewaguarde.pas` / `.dfm` |

Saída: `mnfenewgourmet.bpl` em `$(gourmet1)\modulos`. Dependências preservadas: `rtl`, `mpeggourmet` (`uPegaBase`, `uFuncoes`, …), ACBr, UniDAC, `uCloudComunicacao`.

## Estado atual do scaffold

- Cópia 1:1 do módulo de produção, apenas com a cláusula `unit` renomeada (obrigatório — o nome da unit precisa casar com o arquivo) e a referência cruzada `ufnfecorrecao` → `ufnewnfecorrecao` no `ufnewnfe.pas`.
- **Comportamento idêntico ao de produção hoje** — é a linha de base para aplicar as correções.
- Codificação preservada (UTF-8 BOM em `ufnewnfe`/`ufnewnfce`; ISO-8859-1 nos demais).

## ⚠️ Decisão pendente: coexistência dos dois BPLs

As classes e os forms mantêm os mesmos nomes internos (`Tfnfe`/`fnfe`, `Tfnfce`/`fnfce`, …). Isso é seguro para **troca gradual** (o host carrega **um** BPL por vez — você substitui `mnfegourmet.bpl` por `mnfenewgourmet.bpl` quando quiser ativar a nova versão).

Se o ERP precisar carregar **os dois BPLs ao mesmo tempo** (ex.: rodar NF-e pela versão antiga e NFC-e pela nova simultaneamente), será preciso **renomear classes e variáveis de form** (`Tfnfe`→`TfNewNfe`, `fnfe`→`fNewNfe`, etc.) para evitar conflito de registro em runtime. **Confirmar qual cenário antes de prosseguir.**

## Roadmap de correções (sobre a nova versão, em ordem de prioridade)

Cada item será aplicado em `ufnewnfe.pas`/`ufnewnfce.pas`, nunca nos fontes de produção.

| Prioridade | Correção | Origem (análise) | Onde aplicar |
|---|---|---|---|
| 1 — Crítica | ✅ **Aplicado** — Bloquear nova chave quando `meschavenfe <> ''` (não depende mais só do arquivo em disco) | #9 | `ufnewnfe.pas` ~1538, em `ImprimeNFe` |
| 2 — Crítica | ✅ **Aplicado** — Reconcilia `mesmestotal` × soma dos componentes (fórmula vNF NF-e 4.0), tolerância R$0,02, bloqueia antes de assinar/consumir número | #12 | `ufnewnfe.pas` ~3874, após montagem dos totais |
| 3 — Crítica | ✅ **Aplicado** — `mesdatanfe` passa a usar sempre `ide.dEmi` (fonte única) nos 3 pontos de gravação; antes oscilava entre `vpDataAtual` e `DhRecbto` | #6 | `ufnewnfe.pas` 4359, 4595, 4657 |
| 4 — Crítica | ✅ **Aplicado (NCM/CFOP)** — pré-valida formato de NCM (8 díg.) e CFOP (4 díg.) por item sobre `NFe.Det` já montado, lista todas as pendências numa mensagem e bloqueia antes de numerar/assinar. CST/CSOSN: ver nota | #4 | `ufnewnfe.pas` ~3906, após os totais |
| 5 — Alta | ✅ **Aplicado (produto)** — valida descrição não-vazia e valor > 0 por item (junto da pré-validação #4 sobre `NFe.Det`). Destinatário/endereço: ver nota | #3 | `ufnewnfe.pas` ~3940 |
| 6 — Alta | ✅ **Aplicado** — Persiste `mescodigonota` (`ide.cNF`) já no UPDATE da nota pendente, garantindo reuso do código na reemissão | #5 | `ufnewnfe.pas` ~4334 |
| 7 — Alta | ✅ **Aplicado (parcial)** — eliminados os 4 `try/except` vazios (agora registram via `SalvarLogErro`); motivo do erro persistido no log. Status de erro dedicado: ver nota | #7, #11 | `ufnewnfe.pas` 2552, 4338, 4658, 4720 |
| 8 — Alta | ✅ **Aplicado (principal)** — `TimeOut` das WebServices = 30s (anti-travamento) + `try/finally` liberando o `TIniFile` em `LerConfiguracao`. O `@disable_triggers` do fluxo de emissão já é seguro (UPDATE dentro de try/except). Demais blocos `@disable_triggers`/`LoadPackage`: ver nota | #11 | `ufnewnfe.pas` `LerConfiguracao` (TimeOut ~7543, try/finally 7478/7580) |
| 9 — Alta | ✅ **Aplicado** — valida vencimento do certificado (`SSL.CertDataVenc`) no início do `ImprimeNFe`; bloqueia se vencido (`100093`), falha de leitura não bloqueia (só loga). A1/A3 via `NumeroSerie`+`Senha`+WinCrypt já cobertos; PFX-arquivo fica como melhoria | #10 | `ufnewnfe.pas` ~1534, início de `ImprimeNFe` |
| 10 — Alta | ✅ **Aplicado (conservador)** — registra em log (não bloqueia) quando o CST IBS/CBS não está cadastrado para o item. Conferência recalculado × `mnr`: ver nota | #2 | `ufnewnfe.pas` ~3416, bloco IBS/CBS |

### Nota #7 — status de erro dedicado

Avaliado e **deliberadamente adiado** o uso de um `temcodigo` próprio para "erro" (ex.: `3`). A guarda de numeração (#9) só reaproveita o número de notas cujo `temcodigo <> 3`; marcar uma nota que falhou como `3` faria a retentativa **puxar um número novo**, gerando **gap** na sequência (que exigiria inutilização). Como a nota já consome número na fase pendente, manter `temcodigo = 4` (pendente, reaproveitável) é o comportamento correto para retentativa. O motivo do erro passou a ser **persistido no log** (`SalvarLogErro`) em todos os caminhos. Um status de erro visível sem efeito na numeração exigiria uma coluna nova (`mes.mesmsgerro`) + ajuste da tela de pendentes — fica como item de produto, não de código.

### Nota #5 — validação do destinatário

`validaentidade`/`validatransporte` eram stubs (`Result := True`) **e não são chamados** em lugar nenhum; a validação de IE e CNPJ/CPF do destinatário já ocorre inline via `ACBrValidador`. Implementar+chamar uma validação rígida de endereço (rua/CEP/UF) foi **adiado** por risco de falso positivo (NF-e a consumidor não identificado, devolução, ajuste, etc., onde o endereço completo nem sempre se aplica). A validação de produto (descrição/valor), inequívoca, foi aplicada.

### Nota #10 — IBS/CBS (Reforma Tributária)

Por ser área nova e com regras em transição, a intervenção foi **mínima e não-bloqueante**: apenas log quando o CST IBS/CBS está ausente, para dar visibilidade a falhas de cadastro sem risco de barrar emissões válidas. A **conferência do recalculado contra a tabela `mnr`** (quando a nota é relida por já ter protocolo) e a validação de aliquotas/reduções foram **deixadas para revisão de domínio + testes em homologação** — alterar o cálculo IBS/CBS às cegas teria alto risco de quebra.

## Diferenciação visual aplicada (v1.0)

Mudanças de **baixo risco** (só propriedades de componentes já existentes — sem novos componentes, sem relayout, sem alterar código):

- **Título das janelas** marcando `[ NOVA VERSÃO - mnfeNEWgourmet v1.0 ]` (NF-e, NFC-e, correção e splash).
- **NF-e** (`ufnewnfe.dfm`): painel lateral `PlLogos` recolorido para **verde** (faixa accent) com o texto `mnfeNEWgourmet / NOVA VERSÃO / v1.0`.
- **NFC-e** (`ufnewnfce.dfm`): banner de topo `PlTitulo` recolorido para **verde** com o marcador da nova versão.
- **Splash** (`ufnewaguarde.dfm`): fundo verde durante o processamento.
- **Tipografia**: fonte base trocada de Tahoma para **Segoe UI** nos dois forms principais (altura -11 mantida de propósito para não deslocar os controles posicionados em pixels).

Assim, ao rodar o módulo novo, a faixa/splash verde e o título deixam óbvio que é o `mnfenewgourmet.bpl`. A modernização mais profunda (VCL Styles / redesenho) e a modernização de código ficam no roadmap acima.

## Build e ativação

1. Abrir `mnfenewgourmet.dproj` no RAD Studio (mesma IDE/ambiente do atual).
2. Compilar em `Release` / `Win32` → gera `mnfenewgourmet.bpl`.
3. Validar emissão em **homologação** antes de qualquer uso em produção.
4. Ativação gradual: apontar o host para `mnfenewgourmet.bpl` quando a nova versão estiver validada; o `mnfegourmet.bpl` permanece como rollback imediato.
