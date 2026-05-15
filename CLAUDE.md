# CLAUDE.md — Conversor DOCX → CKEditor | IXC Provedor

Este arquivo descreve o projeto para que o Claude entenda o contexto, as decisões técnicas e como contribuir corretamente.

---

## O que é este projeto

Ferramenta web em arquivo HTML único (`conversor_ixc.html`) que converte documentos Word (`.docx`) para HTML compatível com o CKEditor do sistema **IXC Provedor**, usado por provedores de internet.

Funciona **100% offline** — sem servidor, sem backend, sem instalação. Basta abrir o arquivo no navegador.

---

## Por que existe

No IXC Provedor, modelos de contrato são editados via CKEditor e impressos em PDF pelo TCPDF (biblioteca PHP). O fluxo antigo era:

1. Cliente entrega contrato em Word
2. Operador copia e cola no CKEditor
3. Tabelas quebram — o Word usa estilos MSO incompatíveis
4. O TCPDF quebra células verticalmente por causa de `<figure>` dentro de `<td>`

Esta ferramenta resolve o problema convertendo o XML interno do `.docx` diretamente para o formato nativo de cada versão do CKEditor.

---

## Arquivo principal

```
conversor_ixc.html   ← único arquivo, tudo dentro dele
```

Não há dependências locais. A única lib externa é o **JSZip** carregado via CDN:
```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/jszip/3.10.1/jszip.min.js"></script>
```

---

## Como funciona tecnicamente

### Leitura do .docx
Um `.docx` é um arquivo ZIP. O JSZip abre esse ZIP no browser e lê `word/document.xml`, que contém toda a estrutura do documento em XML (formato OOXML da Microsoft).

### Parsing do XML
O `DOMParser` nativo do browser interpreta o XML. O namespace usado é:
```
http://schemas.openxmlformats.org/wordprocessingml/2006/main  (prefixo w:)
```

### Funções auxiliares compartilhadas
```js
getLN(el)           // retorna o localName do elemento (ignora namespace)
getChild(el, tag)   // retorna o primeiro filho com determinado localName
getWAttr(el, name)  // lê atributo w:name, tentando com e sem namespace
runToHTML(rEl)      // converte um <w:r> em HTML (bold, italic, underline, texto)
paraToHTML(pEl)     // converte um <w:p> retornando { text, align }
cellToHTML(tcEl)    // converte uma <w:tc> em HTML
getColspan(tc)      // lê w:gridSpan para colspan
isSkipRow(tc)       // detecta w:vMerge de continuação (rowspan)
```

### Diferença entre CKEditor 4 e CKEditor 5

| Aspecto | CKEditor 4 | CKEditor 5 |
|---|---|---|
| Wrapper de tabela | nenhum | `<figure class="table" style="width:100%;">` |
| Tag table | `<table border="1" cellpadding="1" cellspacing="1" style="width:100%;">` | `<table class="ck-table-resized" style="border-style:solid;">` |
| Bordas das células | sem estilo inline | `<td style="border-color:#000000;">` |
| Colgroup | não | sim, com larguras percentuais |
| Cores da UI | âmbar `#f59e0b` | azul `#3b82f6` |

### Por que o TCPDF quebrava
O CKEditor 5 envolve tabelas com `<figure class="table">`. Quando havia tabelas aninhadas dentro de `<td>`, o TCPDF recebia `<figure>` dentro de célula — que ele não sabe renderizar — e quebrava o texto verticalmente letra por letra. A solução foi ler o XML do Word diretamente, sem camadas intermediárias que gerem aninhamento.

### Espaços entre palavras
O Word separa palavras em `<w:r>` (runs) independentes. Espaços ficam em runs próprios com `xml:space="preserve"`. A função `runToHTML` preserva qualquer `text` não vazio (incluindo espaços puros), e só descarta runs completamente vazios (`text === ''`).

### Alinhamento
Lido de `<w:pPr><w:jc w:val="center|right|both"/>`. Mapeamento:
- `center` → `text-align:center`
- `right` → `text-align:right`
- `both` → `text-align:justify`

---

## Estrutura do JS

```
ESTADO GLOBAL
  currentMode     // 'ck4' ou 'ck5'
  currentHTML     // último HTML gerado
  lastFile        // último File carregado (para reconversão ao trocar modo)
  fileHistory     // array de até 10 conversões recentes

MODOS (objeto MODES)
  ck4 / ck5       // cores, label, info box, cor do código

TROCA DE MODO
  setMode(mode)   // atualiza CSS vars, badges, e reconverte se houver arquivo

HELPERS COMPARTILHADOS
  getLN, getChild, getWAttr, runToHTML, paraToHTML, cellToHTML, getColspan, isSkipRow

TABELAS CK4
  tableToHTMLCK4(tblEl)
  docToHTMLCK4(bodyEl)

TABELAS CK5
  tableToHTMLCK5(tblEl)
  docToHTMLCK5(bodyEl)

PROCESSAMENTO PRINCIPAL
  processDocx(arrayBuffer)   // JSZip → parseXML → body → docToHTMLCK4|CK5

UI E ESTADO
  setStatus, fmt, escHTML, showResult
  addHistory, renderHistory, loadHist
  handleFile, copyHTML, clearAll

EVENTOS
  // todos via addEventListener, zero onclick inline no HTML
  btnCK5, btnCK4, btnCopy, btnClear, btnSelectFile, fi, .tab, dragover, dragleave, drop
```

---

## Regras ao modificar o código

1. **Nunca usar `onclick` inline no HTML** — todos os eventos são registrados via `addEventListener` no final do `<script>`, após todas as funções estarem definidas.

2. **Nunca usar template literals (backticks)** para strings que serão salvas pelo Python — usar concatenação com `+` ou aspas simples/duplas normais para evitar problemas de encoding.

3. **Nunca usar emojis diretamente** no código JS — usar entidades HTML (`&#128203;`) ou unicode escapes (`\u2192`) para evitar problemas de surrogate pairs.

4. **`paraToHTML` retorna objeto `{ text, align }`**, não string — quem chama (`docToHTMLCK4`, `docToHTMLCK5`, `cellToHTML`) precisa acessar `.text` e `.align` separadamente.

5. **Funções de conversão são compartilhadas** entre CK4 e CK5 — apenas `tableToHTMLCK4/CK5` e `docToHTMLCK4/CK5` diferem. Alterações em `runToHTML`, `paraToHTML`, `cellToHTML` afetam ambos os modos.

6. **`copyHTML` tem três camadas de fallback** — Clipboard API (HTTPS) → `execCommand` (HTTP) → instrução manual. Manter os três pois a ferramenta roda em HTTP na VM.

7. **Testar sempre com arquivo `.docx` real** — especialmente após mudanças no parsing de runs, pPr, tcPr ou gridSpan.

---

## Deploy na VM

Servir o arquivo estático com Nginx ou Python:

```bash
# Nginx
sudo cp conversor_ixc.html /var/www/html/
# acesso: http://IP-DA-VM/conversor_ixc.html

# Python (dev)
python3 -m http.server 8080
# acesso: http://IP-DA-VM:8080/conversor_ixc.html
```

> **Atenção:** `navigator.clipboard` só funciona em HTTPS ou localhost. Em HTTP, o fallback `execCommand` é usado automaticamente.

---

## O que NÃO fazer

- Não usar Pyodide ou qualquer lib que exija servidor — a ferramenta deve funcionar como `file://`
- Não usar Mammoth.js — ele perde colspan/rowspan e gera tabelas aninhadas
- Não gerar `<figure class="table">` no modo CK4 — o CKEditor 4 não usa esse wrapper
- Não remover o fallback de `copyHTML` — a VM roda em HTTP puro
