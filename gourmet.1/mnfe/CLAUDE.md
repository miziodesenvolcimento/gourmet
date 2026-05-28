# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

`mnfegourmet` is a Delphi VCL **runtime package** (`.bpl`) that implements Brazilian fiscal-document emission (NF-e, NFC-e, MDF-e and related events: cancellation, correction letter, inutilização, contingência) for the **Gourmet ERP** by Mizio Sistemas. It is not a standalone executable — it is loaded at runtime by `Gourmeterp.exe`, the host ERP application that lives one directory up.

Code, comments, identifiers, UI text and database column names are all **Portuguese (pt-BR)**. Do not translate them.

## Build / run

There is no command-line build, lint, or test suite in this directory. The package is built from the Embarcadero Delphi IDE (RAD Studio).

- Main project file: `mnfegourmet.dproj` (package, `<AppType>Package</AppType>`, `<RuntimeOnlyPackage>true</RuntimeOnlyPackage>`)
- Package source: `mnfegourmet.dpk`
- Platform: **Win32 only**; configurations `Debug` and `Release`
- Output: `mnfegourmet.bpl` deployed to `$(gourmet1)\modulos\` (Debug DCUs go to `$(gourmet1)\dcu\`). `$(gourmet1)` is an IDE environment variable pointing at the parent `gourmet.1` tree.
- Required package dependency: `mpeggourmet.dcp` (the base package from sibling projects — provides `uPegaBase`, `uFuncoes`, etc.). It must already be compiled and visible to the IDE before this one will build.
- Debugger host app (set in `.dproj`): `c:\gourmet\gourmeterp.exe` (Release) / `$(gourmet1)\Gourmeterp.exe` (Debug). To debug a form here, run the ERP and exercise the relevant menu action.

If a CLI build is ever needed: `msbuild mnfegourmet.dproj /t:Build /p:Config=Release /p:Platform=Win32` from a RAD Studio command prompt. There is no automated test runner.

## Architecture

The package contains five units, but only two are substantial — `ufnfce.pas` (~232 KB) and `ufnfe.pas` (~267 KB). Everything else is small support code.

### Forms shipped in this package

| Unit | Form | Role |
|---|---|---|
| `ufnfe.pas` | `Tfnfe` | NF-e (modelo 55) emission, consultation, cancellation, CCe, inutilização, e-mail dispatch, PDF/DANFE generation |
| `ufnfce.pas` | `Tfnfce` | NFC-e (modelo 65) emission, contingency flow, DANFE-NFCe printing, TEF receipt printing |
| `ufnfecorrecao.pas` | `Tfnfecorrecao` | Modal for CCe text entry (15–1000 chars) |
| `ufnfcecorrecao.pas` | `Tfnfcecorrecao` | Same modal, NFC-e variant. (Note: only `ufnfecorrecao.pas` is listed in `mnfegourmet.dpk`; `ufnfcecorrecao.pas` exists but is not compiled into the BPL — verify whether the host project includes it before editing.) |
| `ufaguarde.pas` | `Tfaguarde` | "Please wait" splash with timer |
| `ufmdfe.pas` | `Tfmdfe` | MDF-e (Manifesto de Documentos Fiscais Eletrônicos). **Not in the .dpk** — present in the directory but compiled elsewhere (likely a separate package). Do not assume edits here affect this BPL. |
| `ufnfcejustificativa.pas` | `Tfnfcejustificativa` | Justification modal. Also **not in the .dpk**. |
| `Ping2.pas` | — | ICMP ping helper via `iphlpapi.dll` (used to check SEFAZ reachability) |
| `uCloudComunicacao.pas` + `uFormComunicacao.pas` | `TMizioComunicacao` / `TFormComunicacao` | WebDAV upload of issued XMLs to Mizio's cloud, organized as `CNPJ/AAAA/MM/`. Not currently registered in the .dpk either. |
| `uknf.pas` | — | Empty stub. |

When adding a new unit, remember to register it in both `mnfegourmet.dpk` (`contains` clause) **and** `mnfegourmet.dproj` (`<DCCReference>`) — the IDE will only edit one of these if you go through Project Manager.

### How fiscal logic is structured

Both `Tfnfe` and `Tfnfce` follow the same shape, so patterns learned in one carry over:

1. **ACBr** is the engine. `ACBrNFeNotas: TACBrNFe` is the central component — it owns `Configuracoes` (cert, ambiente, schemas), `NotasFiscais` (the in-memory NFe list), and `WebServices` (StatusServico, Consulta, Envio, RetRecibo, Cancelamento, EventoNFe, Inutilizacao). The flow is always: clear → load XML / build from DB → `Enviar` / `Consultar` / event → parse response → persist to DB.
2. **Data access is Devart UniDAC**. Dozens of `TUniQuery` components per form (`etd`, `toe`, `mes`, `itm`, `cfg`, `enf`, …) are connected to a `TUniConnection` injected by the host ERP via the `zcone` property setter (`Tfnfce.Setzcone`). Field components are pre-declared (`mesmeschave: TIntegerField`, etc.) — this is generated, not hand-written; regenerating is done in the IDE Fields Editor.
3. **The host ERP drives this package**. It opens the form, sets the connection, calls a public method (`Inicializar`, `mConsultaSEFAZ`, `entraemcontigencia`, `ImprimirComprovantesTEF`, etc.), and closes the form. The forms are not self-launching.
4. **Paths and certificates** come from the `cfg` table (`cfgnumecertif`, `cfgsenhacertificado`, etc.). `AjustaCaminhoGeralNF(Data)` builds the per-day folder layout for XMLs (`enviados\Autorizados\AAAAMM\`, `Cancelados\`, `Denegados\`, …) — match this convention when adding new artifact types.
5. **External dependencies pulled from the parent tree**: `uPegaBase` (base classes, global state), `uFuncoes` (string/CNPJ/date helpers including `SoNumeros`). Reading these requires opening the sibling package projects — there is no API doc.

### Conventions worth knowing

- Field components are named `<query><table-prefix><column>` (e.g. `mesmeschave` = query `mes`, table `mes`, column `chave`). Three-letter table prefixes are pervasive (`mes`=mestre/header, `itm`=item, `etd`=entidade/empresa, `toe`=tomador/destinatário, `cfg`=configuração, `enf`=evento NFe, `cst`/`icm`/`cfo`=fiscal lookup tables).
- Variables prefixed `vp` are form-instance fields ("variável privada"); `vl` are locals; `v` plain is a parameter or general local. Follow the existing prefix when adding variables — it's a project-wide convention, not just stylistic.
- **File encoding is mixed across the directory.** The two big units (`ufnfe.pas`, `ufnfce.pas`) are UTF-8 with BOM; the smaller modal units (`ufnfecorrecao.pas`, `ufnfcecorrecao.pas`, etc.) are ISO-8859-1 / Windows-1252. Match whatever the file you're editing already uses — converting between them silently corrupts every accented character.
- Many showmessages and exceptions are tagged with numeric codes (e.g. `100058 - Documento fiscal não localizado`) used by support to triage. Keep the numbers when editing those strings.
- `MidasLib` is linked so client datasets don't need `midas.dll`. Don't remove it from `uses`.

## Working in this repo

- **Never edit files in `__history\` or `__recovery\`** — those are the IDE's autosave backups; changes there will be overwritten and won't compile.
- **`.dfm` files are the form layouts** (binary-equivalent text). Avoid editing them by hand for layout changes — let the IDE do it. Editing event handler name strings or constants in `.dfm` is fine.
- **There is no `.gitignore`** anywhere in this tree, so `git status` is permanently noisy with `.dcu`, `.bpl`, `.identcache`, `.dproj.local`, etc. from every sibling package. Stage source files **by name** — never `git add -A` / `git add .`, you will pull in megabytes of compiler output. To see only `.pas`/`.dfm`/`.dpr` changes: `git status -s | grep -E '\.(pas|dfm|dpr|dpk|dproj)$'`.
- **Commit message style** (from `git log`): lowercase Portuguese, verb-first describing the fix area. Examples: `ajustado integração com stone e envio de mensagens via whats`, `correções para integração onde o recebimento é online pelo aiqfome`. No conventional-commits prefixes, no PR/issue refs.
- When the user mentions a fix "in the NFC-e flow" they almost certainly mean `ufnfce.pas`; "NF-e" means `ufnfe.pas`. They're distinct files even though they share patterns — if a bug exists in one, check the other for the parallel fix.
