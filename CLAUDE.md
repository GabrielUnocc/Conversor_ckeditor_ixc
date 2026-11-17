# CLAUDE.md — Conversor DOCX → CKEditor | IXC Provedor

Este arquivo descreve o projeto para que o Claude entenda o contexto, as decisões técnicas e como contribuir corretamente.

---

## O que é este projeto

Ferramenta web em arquivo HTML único (`index.html`) que converte documentos Word (`.docx`) para HTML compatível com o CKEditor (versões 4 e 5) do sistema **IXC Provedor**, usado por provedores de internet. Publicada em https://conversor.ixcsoft.net/.

**Sem servidor e sem backend**: todo o processamento acontece no navegador e nenhum dado do contrato sai da máquina. Também abre direto do disco (`file://`), mas **precisa de internet** para carregar o JSZip (CDN) e as fontes (Google Fonts) — não é 100% offline.

---

## Por que existe

No IXC Provedor, modelos de contrato são editados via CKEditor e impressos em PDF pelo **TCPDF** (biblioteca PHP). O fluxo antigo era:

1. Cliente entrega contrato em Word
2. Operador copia e cola no CKEditor
3. Tabelas quebram — o Word usa estilos MSO incompatíveis
4. O TCPDF quebra células verticalmente por causa de `<figure>` dentro de `<td>`

Esta ferramenta resolve o problema lendo o XML interno do `.docx` e gerando direto o formato nativo de cada versão do CKEditor, já ajustado para as manias do TCPDF.

---

## Arquivo principal

```
index.html   ← único arquivo, tudo dentro dele (servido como página inicial do site)
```

Não há build nem dependências locais. A única lib externa é o **JSZip** via CDN:
```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/jszip/3.10.1/jszip.min.js"></script>
```

---

## Como funciona tecnicamente

### Arquivos lidos do .docx
Um `.docx` é um ZIP. O JSZip abre o ZIP no navegador e o `DOMParser` nativo interpreta os XMLs (namespace `http://schemas.openxmlformats.org/wordprocessingml/2006/main`, prefixo `w:`):

| Arquivo | Para quê | Obrigatório |
|---|---|---|
| `word/document.xml` | Conteúdo: parágrafos, runs, tabelas | sim |
| `word/styles.xml` | Negrito/itálico/sublinhado/alinhamento/numeração **herdados de estilos** (ex: "Título 1") | não |
| `word/numbering.xml` | Formato das listas numeradas e marcadores | não |

`processDocx(arrayBuffer, mode)` monta o objeto global **`CTX`** (`styles`, `numbering`, `counters`) a cada conversão e então chama `docToHTML(body, mode)`. Todas as funções de parágrafo dependem de `CTX` estar preenchido.

### Formatação efetiva (estilos)
A ordem de aplicação é: `docDefaults` → estilo do parágrafo (com a cadeia `basedOn` resolvida por `resolveStyle`) → estilo de caractere (`w:rStyle`) → `w:rPr` direto do run. Detalhes:
- `<w:b w:val="0"/>` / `"false"` **desliga** o negrito (não basta a tag existir) — ver `onOff`.
- `w:bCs` / `w:iCs` (negrito/itálico de scripts complexos) **não** contam como negrito/itálico.
- Texto com `w:vanish` (oculto) é descartado.
- Parágrafo sem `w:pStyle` usa o estilo padrão do documento (`w:default="1"`).
- Alinhamento: `w:jc` direto no parágrafo ou herdado do estilo.

### Numeração de listas
`nextListLabel(numId, ilvl)` mantém contadores por `numId` em `CTX.counters`, respeita `w:start`, `w:startOverride`, `w:lvlOverride`, reinicia níveis mais profundos e monta o rótulo a partir do `w:lvlText` (`%1.`, `%1.%2)`, `CLÁUSULA %1ª:`...). Formatos: `decimal`, `decimalZero`, `lowerLetter`, `upperLetter`, `lowerRoman`, `upperRoman`, `bullet`, `none`. O `numPr` pode vir do parágrafo ou do estilo.

A lista **não** vira `<ol>/<ul>`: o rótulo é inserido como texto no início do parágrafo (`"1. "`, `"• "`). Marcadores das fontes Symbol/Wingdings (área privada U+F0xx) viram `•`.

### Tabulações e linhas (campos para preencher e assinaturas)
- `w:tab` **sublinhado** → sequência de `_` (campo para preencher: `Assinatura: ______`, `__/__/____`).
- `w:tab` comum → alguns ` ` (espaçamento).
- Linha horizontal desenhada (`w:drawing` com altura ~0, `w:pict` VML com `v:line` ou altura ≤ 1,5pt, ou `mc:AlternateContent` com qualquer um dos dois) → sequência de `_` proporcional à largura no Word.
- Imagens (`pic:pic`, `v:imagedata`) continuam ignoradas.

O comprimento é **estimado**: o parágrafo guarda a posição horizontal aproximada `ps.x` (≈110 twips por caractere, `TW_PER_CHAR`) para saber até qual parada de tabulação (`w:tabs`, ou a padrão de 720 twips) o sublinhado vai. Cada `_` ≈ 120 twips (`TW_PER_UNDER`). Usa-se `_` (e não CSS/borda) porque é o que imprime de forma confiável no TCPDF.

### Envelopes do Word
Conteúdo pode vir embrulhado em elementos que não são conteúdo. Eles são atravessados para o texto não sumir:
- Nível de bloco (`flatChildren`): `w:sdt`/`w:sdtContent` (controles de conteúdo), `w:customXml`, `w:ins`, `w:moveTo`. Vale para `body`, linhas (`tr`) e células (`tc`).
- Nível de run (`collectRuns`): `w:hyperlink`, `w:sdt`, `w:ins`, `w:moveTo`, `w:smartTag`, `w:customXml`, `w:fldSimple`, `w:dir`, `w:bdo`.
- `w:del` / `w:moveFrom` (texto excluído com controle de alterações) ficam de fora de propósito.

### Tabelas
`readTable(tbl)` lê a tabela uma vez só (compartilhado por CK4 e CK5):
- `numCols` = número de `w:gridCol` (só eles — `w:tblGridChange` não conta). Sem `tblGrid`, usa a linha mais larga.
- Larguras reais das colunas via `gridColWidths` → `<colgroup>` com porcentagens (nos dois modos).
- `colspan` vem de `w:gridSpan`; células de continuação de `w:vMerge` são puladas (rowspan não é suportado).
- Toda linha é completada até `numCols` aumentando o `colspan` da última célula (mantém o alinhamento no TCPDF).
- **Tabela dentro de célula** não vira tabela aninhada (o TCPDF quebra): `nestedTableLines` transforma cada linha dela em uma linha de texto, células separadas por `&nbsp;|&nbsp;`.

### Diferença entre CKEditor 4 e CKEditor 5

| Aspecto | CKEditor 4 | CKEditor 5 |
|---|---|---|
| Wrapper de tabela | nenhum | `<figure class="table" style="width:100%;">` |
| Tag table | `<table border="1" cellpadding="1" cellspacing="1" style="width:100%;">` | `<table class="ck-table-resized" style="border-style:solid;">` |
| Bordas das células | sem estilo inline | `<td style="border-color:#000000;">` |
| Colgroup | sim, larguras reais em % | sim, larguras reais em % |
| Cores da UI | âmbar `#f59e0b` | azul `#3b82f6` |

Só `tableToHTMLCK4` e `tableToHTMLCK5` diferem; o resto é compartilhado.

---

## Regras específicas do TCPDF (não quebrar!)

Cada uma destas regras existe por causa de um bug real que só aparecia no PDF impresso:

| Bug | Sintoma no PDF | Regra |
|---|---|---|
| `<figure>` dentro de `<td>` | texto vertical, letra por letra | nunca gerar tabela aninhada; nunca usar Mammoth |
| Bug 5 | palavra cortada no meio ("caracterí/sticas") | espaço **único** entre palavras continua espaço normal (quebrável); só runs de 2+ espaços viram ` ` |
| Bug 6 | palavras coladas em negrito ("TERMODEADESÃO") | juntar runs vizinhos de mesma formatação **antes** de envolver em `<strong>/<em>/<u>`; espaço nunca pode ficar solto entre duas tags (se sobrar, vira ` `/`&nbsp;`) |
| Bug 7 | rótulos espremidos ("Bair/ro:", "CE/P:") | usar a largura real de cada coluna (`w:tblGrid`), não larguras iguais |
| Issue #7 | títulos sem negrito/centralização, sem numeração, linhas de assinatura e campos sumindo | ler `styles.xml` e `numbering.xml`; converter tabulação sublinhada e linhas desenhadas em `_` |

---

## Estrutura do JS

```
ESTADO GLOBAL
  currentMode     // 'ck4' ou 'ck5'
  currentHTML     // último HTML gerado
  lastFile        // File atual (reconvertido ao trocar de modo)
  fileHistory     // até 10 conversões { name, size, file, html, mode, date }
  convSeq         // id da conversão mais recente; resultados velhos são descartados

MODOS (objeto MODES)  // cores, label, info box, cor do código
setMode(mode, noReconvert)

HELPERS          getLN, getChild, getWAttr, repeatStr, clamp, escText,
                 flatChildren, collectRuns
ESTILOS          onOff, readRunFmt, applyFmt, readNumPr, parseStyles, resolveStyle
NUMERAÇÃO        parseNumbering, getLevelDef, nextListLabel, fmtNumber, toRoman, toLetter, bulletChar
TAB / LINHAS     readTabStops, nextTabStop, lineFromDrawing, lineFromPict, lineFromAny
RUNS/PARÁGRAFOS  parseRun(rEl, ps, baseFmt) -> { text, b, i, u } | null
                 mapJc, paraToHTML(pEl) -> { text, align }
CÉLULAS/TABELAS  cellToHTML, nestedTableLines, getColspan, isSkipRow, countGridCols,
                 gridColWidths, readTable, colgroupHTML, spanAttr
                 tableToHTMLCK4, tableToHTMLCK5
DOCUMENTO        docToHTML(bodyEl, mode)
PRINCIPAL        readZipText, processDocx(arrayBuffer, mode)

UI               setStatus, fmt, escHTML, showResult,
                 addHistory(file, html, mode), renderHistory, loadHist,
                 handleFile, copyHTML, clearAll

EVENTOS          // todos via addEventListener, zero onclick inline
  btnCK5, btnCK4, btnCopy, btnClear, btnSelectFile, fi, histList, .tab,
  dragover, dragleave, drop
```

---

## Regras ao modificar o código

1. **Nunca usar `onclick` inline no HTML** — todos os eventos são registrados via `addEventListener` no final do `<script>`, após todas as funções estarem definidas.

2. **Nunca usar template literals (backticks)** — usar concatenação com `+` e aspas simples/duplas, para evitar problemas de encoding quando o arquivo é salvo por scripts.

3. **Nunca usar emojis diretamente** no código — usar entidades HTML (`&#128203;`) ou unicode escapes (`→`) para evitar problemas de surrogate pairs.

4. **`paraToHTML` retorna objeto `{ text, align }`**, não string — quem chama (`docToHTML`, `cellToHTML`) acessa `.text` e `.align` separadamente.

5. **`processDocx` recebe o modo como parâmetro** — não ler `currentMode` dentro da conversão (o usuário pode trocar de modo no meio de uma conversão assíncrona).

6. **Toda conversão passa por `processDocx`**, que monta o `CTX`. Não chamar `paraToHTML`/`parseRun` sem `CTX`.

7. **`copyHTML` tem três camadas de fallback** — Clipboard API (HTTPS) → `execCommand` (HTTP) → instrução manual. Manter as três.

8. **Testar sempre com arquivo `.docx` real** — especialmente após mudanças em runs, estilos, numeração, `pPr`, `tcPr` ou `gridSpan`. Dá para testar sem abrir o navegador manualmente: servir a pasta com `python3 -m http.server`, carregar o `index.html` num `<iframe>` de uma página de teste e chamar `iframe.contentWindow.processDocx(buf, 'ck5')` com `google-chrome --headless=new --dump-dom` (usar o `fetch` do próprio iframe para obter o `ArrayBuffer`, senão o JSZip rejeita por ser de outro realm).

---

## Limitações conhecidas

- Imagens não são convertidas.
- Tamanho/família de fonte, cor do texto e recuo de parágrafo não são preservados (títulos saem em negrito/alinhados, mas no tamanho padrão).
- Listas viram parágrafos com o rótulo em texto (não `<ol>/<ul>`).
- Rowspan (`w:vMerge`) é detectado e ignorado.
- Caixas de texto (`wps:txbx`), cabeçalho/rodapé e notas de rodapé não são lidos.
- Comprimento de tabulações/linhas é estimado.
- `.doc` (formato legado) não é suportado.

---

## Deploy

Site estático, um arquivo só:

```bash
# Nginx
sudo cp index.html /var/www/html/

# Python (dev)
python3 -m http.server 8080
```

> `navigator.clipboard` só funciona em HTTPS ou localhost. Em HTTP, o fallback `execCommand` é usado automaticamente.

---

## O que NÃO fazer

- Não usar Pyodide ou qualquer lib que exija servidor
- Não usar Mammoth.js — ele perde colspan/rowspan e gera tabelas aninhadas
- Não gerar `<figure class="table">` no modo CK4 — o CKEditor 4 não usa esse wrapper
- Não gerar tabela dentro de célula (o TCPDF quebra o texto letra por letra)
- Não remover o fallback de `copyHTML`
