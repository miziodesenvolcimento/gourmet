# Análise do Processo de Emissão de NF-e — Módulo `mnfegourmet.bpl`

> Documento de auditoria gerado a partir da leitura de `ufnfe.pas` (8.300 linhas) e `ufnfce.pas` (7.300 linhas).
> **Nenhum fonte de produção foi alterado para gerar esta análise.** Referências de linha conferidas em 2026-05-28; reconferir na IDE antes de corrigir, pois o arquivo evolui.
> Método central de emissão: `Tfnfe.ImprimeNFe` (~1244–4655).

## Sumário de severidade

| # | Item | Status | Severidade |
|---|------|--------|------------|
| 1 | Número salvo no banco | OK com ressalva | Baixa |
| 2 | XML com produtos + IBS/CBS | OK parcial | Alta |
| 3 | Validação de cliente/produto | FALHA | Alta |
| 4 | NCM/CFOP/CST/CSOSN/tributação | RISCO | Crítica |
| 5 | Campos principais no MySQL | OK com falha pontual | Alta |
| 6 | `mesdatanfe` = dia ou autorização | RISCO | Crítica |
| 7 | Falha de cadastro → status pendente | FALHA | Alta |
| 8 | Reemissão usa a mesma chave | OK | Baixa |
| 9 | Bloqueio de nova chave quando já existe | RISCO | Crítica |
| 10 | Certificado/assinatura/layout | RISCO | Alta |
| 11 | Robustez / blindagem contra travamentos | RISCO | Alta |
| 12 | Validação de totais (itens × total) | FALHA | Crítica |

---

## 1. Número da NF-e salvo no banco — OK (ressalva)

A numeração usa a procedure `NumeroNFe(:flacodigo)` e/ou reaproveita `mes.mesnumero`. Já foi adicionada guarda por modelo (reuso só quando `mestdfcodigo = tdfNotaFiscalEletronica`/'55'), o que corrige o vazamento entre sequências NF-e × NFC-e. **Ressalva:** o número é potencialmente atribuído em dois pontos (`VerificaPreExistencia` ~7609 e `ImprimeNFe` ~4220/4537); consolidar a obtenção do número num único ponto reduz risco de gap/duplicidade.

## 2. XML com produtos + IBS/CBS — OK parcial / RISCO

Os grupos IBS/CBS (Reforma Tributária) **são montados por item e nos totais** (`Total.IBSCBSTot.*`), com condicional por regime (CRT ≥ 3) e releitura da tabela `mnr` quando a nota já tem protocolo. **Riscos:** (a) a base IBS/CBS desconta ICMS/PIS/COFINS — qualquer erro nesses contamina a base; (b) não há validação do CST IBS/CBS antes de usar; (c) não há conferência entre valor recalculado e o gravado em `mnr`. **Correção:** validar CST IBS/CBS, registrar em log quando há recálculo e conferir contra `mnr`.

## 3. Validação de cliente e produto — FALHA

`validaentidade` (`ufnfe.pas:950-953`) e `validatransporte` (`955-958`) **retornam sempre `True`** — não validam nada. `validatemprodutos` (`932-948`) apenas confere se existe ≥1 item não-serviço. IE e CNPJ/CPF são validados via `ACBrValidador`, mas **endereço, razão social, descrição/unidade/valor do produto não são validados** antes da montagem. **Risco:** rejeição SEFAZ por dado obrigatório ausente, descoberta só após assinar/enviar. **Correção:** implementar `validaentidade`/`validatemprodutos` de verdade (endereço completo, descrição não-vazia, valor > 0) e chamá-las no início de `ImprimeNFe`.

## 4. NCM / CFOP / CST / CSOSN / tributação — RISCO (crítico)

`Prod.ncm := SomenteNumeros(...)` e `Prod.CFOP := SoNumeros(...)` **não validam** formato nem existência. A decisão CST × CSOSN usa `Emit.CRT` (Simples × Normal). Erros de NCM/CFOP/CST só são detectados **reativamente** na chamada `Validar`/`Enviar` (após assinar) — gerando reprocessamento e consumo de tempo. **Correção:** pré-validar NCM (8 dígitos + tabela), CFOP (existência + coerência UF), e CST/CSOSN contra o regime, **antes** de montar o item, listando todos os itens inválidos numa só mensagem.

## 5. Campos principais no MySQL — OK (falha pontual)

Os UPDATEs em `mes` gravam `mesnumero`, `messerie`, `mesregistro`, `tdfcodigo`, `meshoranfe`, `mesprotocolo`, `meschavenfe`, `mesdatanfe` e `temcodigo` (ver `ImprimeNFe` ~4234, ~4534; `mConsultaNFe` ~6506). **Falha:** `mescodigonota` (o `ide.cNF`) **só é gravado no caminho de rejeição 539** (~4447); no fluxo normal de autorização **não é persistido** (`4534-4543`). Como `cNF` compõe a chave, perder o XML torna-o irrecuperável. **Correção:** calcular `GerarCodigoDFe`/`ide.cNF` antes do envio e gravar `mescodigonota` no UPDATE de autorização.

## 6. `mesdatanfe` = data do dia ou de autorização — RISCO (crítico)

Há **inconsistência entre caminhos**: um grava `mesdatanfe` com `DhRecbto` (autorização SEFAZ) e o caminho de sucesso normal grava com `vpDataAtual` (data do dia) — `ufnfe.pas:4536`. Como a data fiscal da nota é `ide.dEmi`, gravar valores diferentes em caminhos distintos pode divergir do XML. **Correção:** usar **sempre `ide.dEmi`** como fonte única de verdade para `mesdatanfe`, tornando-o imutável após a primeira emissão.

## 7. Falha de cadastro → status pendente — FALHA

Mapa de `temcodigo`: 2=cancelado/inutilizado, 4=pendente (com número), 5=autorizado; 3 aparece na guarda de reuso mas sem semântica documentada. **Problemas:** (a) o `try/except` em torno de `VerificaPreExistencia('55')` está **vazio** (`ufnfe.pas:1538-1543`) — engole a exceção; (b) ao falhar a validação a nota fica `temcodigo=4` **sem motivo persistido**. O operador não sabe por que a nota não saiu. **Correção:** criar status de erro persistido (ex.: `temcodigo=3` + coluna `mes.mesmsgerro`) e registrar `E.Message`; nunca engolir exceção sem logar.

## 8. Reemissão usa a mesma chave — OK

`VerificaPreExistencia` e `mReimprimirNFe` (`6071`) carregam o XML existente (`CarregaNFe`/`GeraNomeNFe`) e reimprimem **sem regenerar chave nem número**. Comportamento correto.

## 9. Bloqueio de nova chave quando já existe — RISCO (crítico)

`VerificaPreExistencia` decide pela **existência do arquivo .xml em disco**, não por `meschavenfe` no banco. Se o XML for movido/apagado mas `meschavenfe` continuar preenchido, o sistema pode **gerar nova chave com o mesmo número** → duplicidade fiscal. **Correção:** antes de numerar/gerar chave, bloquear quando `mes.meschavenfe <> ''` — se tem protocolo, só reimprime; se não tem, exige consulta SEFAZ para recuperar status. Não depender apenas do arquivo em disco.

## 10. Certificado / assinatura / layout — RISCO

`VersaoDF` é configurável via `cfgversao` (`ve400`/`ve310`) em `ufnfe.pas` (~7385); em `ufnfce.pas` está fixo em `ve400`. **Não há validação de vencimento do certificado** antes do envio — certificado expirado gera rejeição "silenciosa". `ufnfe.pas` carrega só por `NumeroSerie` (A3); `ufnfce.pas` suporta A1 (PFX+senha) e A3. A senha do certificado é lida do banco (avaliar criptografia — ação de DBA). **Correção:** validar `DataVencimento` do certificado antes de `Enviar`; padronizar suporte A1/A3 entre os dois forms; validar `NumeroSerie` não-vazio.

## 11. Robustez / blindagem contra travamentos — RISCO

- **`set @disable_triggers=1` sem `try/finally`/transação** (`ufnfe.pas:4530` e ~26 ocorrências): se houver exceção antes do `=null`, os triggers ficam desativados → corrupção silenciosa. **Correção:** `try ... finally ExecSQL('set @disable_triggers=null') end`, idealmente dentro de transação explícita (commit/rollback).
- **`try/except` vazios** (ex.: `1538-1543`): engolem erros. **Correção:** logar via `SalvarLogErro`.
- **Envio/consulta SEFAZ síncronos na thread da UI sem timeout explícito** (`Enviar`, `StatusServico.Executar`): SEFAZ lenta congela o app. **Correção:** configurar `WebServices.TimeOut` e/ou mover para thread/`TTask`; evitar `sleep` fixo na UI.
- **Vazamento de recursos:** `Ini.Free` e `UnLoadPackage` comentados; queries abertas sem `finally`. **Correção:** `try/finally` para `TIniFile`, `LoadPackage` e queries.
- **Parsing frágil da resposta SEFAZ** (`Copy/pos` sem checar `pos=0`). **Correção:** validar presença da tag antes do `Copy`.

## 12. Validação de totais — FALHA (crítico)

`Total.ICMSTot.vNF := mesmestotal.AsCurrency` (`ufnfe.pas:3832`) — o **valor total da NF vem do banco**, não da soma item a item (`vProd = RoundTo(vxtot,-2)` em `3680`). **Não há conferência** entre a soma dos itens (+frete +outras −desconto) e `mesmestotal`, nem entre tributos acumulados (`vltotpis`, `vltotcofins`, `vtICMS`) e os totais enviados. Se `mesmestotal` foi calculado errado em outro módulo, a NF é enviada divergente. **Correção:** antes de assinar, reconciliar `vxtot + frete + outras − desc` contra `mesmestotal` (tolerância R$0,01) e abortar com mensagem clara em caso de divergência; idem para PIS/COFINS/ICMS.

---

## Ações prioritárias (ordem sugerida)

1. **Crítica** — Bloquear nova chave quando `meschavenfe <> ''` (#9).
2. **Crítica** — Reconciliar totais (itens × `mesmestotal`) antes de assinar (#12).
3. **Crítica** — Fonte única `ide.dEmi` para `mesdatanfe` (#6).
4. **Crítica** — Pré-validar NCM/CFOP/CST/CSOSN antes de montar item (#4).
5. **Alta** — Implementar `validaentidade`/`validatemprodutos` reais (#3).
6. **Alta** — Persistir `mescodigonota` (`ide.cNF`) no fluxo normal (#5).
7. **Alta** — Status de erro persistido + parar de engolir exceções (#7, #11).
8. **Alta** — `try/finally` para `@disable_triggers`, recursos e timeouts SEFAZ (#11).
9. **Alta** — Validar vencimento do certificado antes do envio (#10).
10. **Alta** — Validar/registrar CST IBS/CBS e recálculos (#2).
