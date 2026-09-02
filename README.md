<div align="center">

# 📄 Conversor de Contratos IXC

**Converte modelos de contrato Word (.docx) para HTML compatível com o CKEditor do IXC Provedor — com bordas corretas, proporções de tabela fiéis e sem palavras coladas ou cortadas na impressão.**

[![Acesse agora](https://img.shields.io/badge/Acesse%20agora-conversor.ixcsoft.net-3B82F6?style=for-the-badge&logo=googlechrome&logoColor=white)](https://conversor.ixcsoft.net/)
![Sem backend](https://img.shields.io/badge/Sem%20backend-100%25%20no%20navegador-f59e0b?style=for-the-badge)
![Dados](https://img.shields.io/badge/Dados-nada%20sai%20do%20navegador-22c55e?style=for-the-badge)

**🔗 [conversor.ixcsoft.net](https://conversor.ixcsoft.net/)**

</div>

---

## O problema

Provedores de internet que usam o **IXC Provedor** precisam cadastrar modelos de contrato no editor de texto do sistema. Ao colar o conteúdo do Word direto no CKEditor, a formatação quebrava — e o problema só aparecia depois, na **impressão em PDF** (gerada pelo TCPDF do IXC):

| Situação | Antes | Depois |
|---|---|---|
| Tempo de cadastro | 60–120 min | 2–5 min |
| Texto vertical nas células (letra por letra) | Frequente | Eliminado |
| Palavras coladas ("TERMODEADESÃO") | Frequente | Eliminado |
| Palavras coladas em **negrito** | Frequente | Eliminado |
| Palavras **cortadas no meio** ("caracterí/sticas") | Frequente | Eliminado |
| Rótulos espremidos em tabelas ("Bair/ro:", "CE/P:") | Frequente | Eliminado |
| Alinhamento e proporções de coluna perdidos | Frequente | Preservados |
| Retrabalho do operador | Alto | Zero |

**Causa raiz:** ao colar do Word, estilos incompatíveis corrompem a formatação; e o CKEditor/TCPDF trata espaços, tags e larguras de coluna de forma diferente do navegador, gerando artefatos que só aparecem no PDF impresso.

---

## Solução

A ferramenta lê o **XML interno do arquivo `.docx`** diretamente no navegador (sem backend), extrai parágrafos, tabelas e formatações, e gera o HTML no formato nativo de cada versão do CKEditor — já tratado para imprimir corretamente no TCPDF.

```
Contrato .docx  →  Ferramenta  →  HTML nativo CKEditor  →  IXC Provedor  →  PDF correto
```

---

## Como usar

Acesse direto no navegador, sem instalar nada:

**➡️ [conversor.ixcsoft.net](https://conversor.ixcsoft.net/)** (Chrome ou Edge)

### Passo a passo

1. **Escolha a versão** do seu editor — **CKEditor 5** ou **CKEditor 4** (primeiro passo, no topo do painel esquerdo). Em caso de dúvida, use o link "Não sabe qual é a versão do seu editor?"
2. **Arraste o `.docx`** para a área de upload ou clique em **Selecionar arquivo**
3. **Confira** o resultado na aba **Preview**
4. Clique em **Copiar HTML** (no topo do painel de resultado)
5. No IXC Provedor: `Modelo de contrato` → botão `</>` (código-fonte) → seleciona tudo → cola → **OK** → **Salvar**

> A ferramenta reconverte automaticamente ao trocar de versão, então você pode comparar CK4 e CK5 sem recarregar o arquivo.

---

## O que a ferramenta trata

- **CKEditor 4 e 5** — alterna entre os formatos com um clique e reconverte o arquivo automaticamente.
- **Tabelas com bordas corretas** — formato nativo de cada editor (`border-color:#000000` no CK5, `border="1"` no CK4).
- **Proporções de coluna fiéis ao Word** — as larguras reais de cada coluna são lidas do `w:tblGrid` e aplicadas via `<colgroup>`, em vez de distribuídas igualmente. Isso evita rótulos espremidos e cortados na impressão de formulários densos.
- **Células mescladas** — `colspan` extraído do XML real do Word (`w:gridSpan`), com correção automática para que toda linha some o total de colunas da grade (mantém o alinhamento no TCPDF).
- **Sem palavras coladas** — runs vizinhos de mesma formatação são unidos numa única tag, evitando o padrão `</strong> <strong>` que o TCPDF descarta (que colava palavras em negrito como "TERMODECONTRATAÇÃO").
- **Sem palavras cortadas no meio** — os espaços entre palavras são mantidos quebráveis, para o TCPDF poder quebrar a linha no lugar certo (e não no meio de "características").
- **Alinhamento** — centralizado, à direita e justificado, lidos do `w:jc` e aplicados via `text-align`.
- **Formatação inline** — negrito (`<strong>`), itálico (`<em>`) e sublinhado (`<u>`) preservados.
- **Histórico de sessão** — as últimas conversões ficam disponíveis enquanto a aba estiver aberta; reconverter o mesmo arquivo (ou trocar de versão) atualiza o registro em vez de duplicar.
- **Cópia inteligente** — detecta HTTPS ou HTTP e usa o método de cópia compatível.

---

## Segurança e privacidade

> **Nenhum dado do contrato sai do navegador.**

- O `.docx` é lido e processado **inteiramente no navegador do usuário** — não há upload para servidor.
- Nenhum contrato, dado de cliente ou informação pessoal é armazenado ou transmitido.
- Ao fechar a aba, tudo é descartado da memória.
- Servido via **HTTPS/TLS** — comunicação criptografada.
- Sem backend, sem banco de dados, sem login.

---

## Formatos de saída

### CKEditor 5

As larguras vêm das colunas reais do documento (proporcionais), não fixas em partes iguais:

```html
<figure class="table" style="width:100%;">
  <table class="ck-table-resized" style="border-style:solid;">
    <colgroup>
      <col style="width:10.7768%;">
      <col style="width:14.4229%;">
      <col style="width:6.5358%;">
    </colgroup>
    <tbody>
      <tr>
        <td style="border-color:#000000;">Conteúdo</td>
        <td style="border-color:#000000;" colspan="2">Célula mesclada</td>
      </tr>
    </tbody>
  </table>
</figure>
```

### CKEditor 4

```html
<table border="1" cellpadding="1" cellspacing="1" style="width:100%;">
  <colgroup><col style="width:10.7768%;"> ... </colgroup>
  <tbody>
    <tr>
      <td>Conteúdo</td>
      <td colspan="2">Célula mesclada</td>
    </tr>
  </tbody>
</table>
```

---

## Como funciona internamente

Um arquivo `.docx` é um ZIP contendo XMLs no formato **OOXML** (padrão Microsoft). A ferramenta usa o **JSZip** para abrir esse ZIP no navegador e lê o `word/document.xml` com o **DOMParser** nativo.

```
.docx (ZIP)
└── word/
    └── document.xml  ← lido pelo DOMParser
         ├── <w:p>       → <p style="text-align:...">
         ├── <w:tbl>     → <figure class="table"> (CK5) ou <table border="1"> (CK4)
         ├── <w:tblGrid> → <colgroup> com as larguras reais de cada coluna
         ├── <w:tc>      → <td colspan="N">  (colspan via w:gridSpan)
         ├── <w:r>       → <strong>/<em>/<u>  (runs de mesma formatação são unidos)
         └── <w:jc>      → text-align: center | right | justify
```

Pontos-chave da conversão, todos pensados para o **TCPDF** do IXC:

- **Espaços** entre palavras permanecem quebráveis (para o PDF quebrar a linha no lugar certo); só sequências de 2+ espaços viram non-breaking.
- **Runs de mesma formatação são unidos** antes de virar `<strong>/<em>/<u>`, para o espaço não ficar solto entre tags (o TCPDF descartaria).
- **Larguras de coluna** vêm do `w:tblGrid` real, preservando as proporções do Word.

Não há Mammoth, não há backend, não há build step — apenas HTML, CSS e JavaScript puro.

---

## Tecnologias

| Tecnologia | Versão | Uso |
|---|---|---|
| HTML + CSS | — | Interface e layout |
| JavaScript | ES2020+ | Lógica de conversão e UI |
| [JSZip](https://stuk.github.io/jszip/) | 3.10.1 | Leitura do `.docx` como ZIP no navegador |
| DOMParser | Nativo | Interpretação do XML OOXML |

---

## Uso local (opcional)

A ferramenta é um único arquivo HTML e roda direto do disco:

1. Baixe o `index.html`
2. Abra no Chrome ou Edge

Para servir em rede local durante testes:

```bash
python3 -m http.server 8080
# Acesse: http://localhost:8080/index.html
```

> **Atenção:** em HTTP puro (sem HTTPS), o botão **Copiar HTML** usa um fallback via `execCommand`, que funciona normalmente. A API `navigator.clipboard` exige HTTPS ou `localhost`.

---

## Estrutura do repositório

```
index.html           # a ferramenta completa — único arquivo, sem dependências locais
CLAUDE.md            # contexto técnico detalhado (arquitetura, regras e histórico de bugs)
README.md            # este arquivo
```

---

## Limitações conhecidas

| Limitação | Situação |
|---|---|
| Imagens dentro do `.docx` | Não convertidas |
| Listas numeradas / marcadores | Convertidas como parágrafos simples |
| Tamanho e família de fonte | Não preservados |
| Cor de texto e recuo de parágrafo | Não preservados |
| Mesclagem vertical (rowspan) | Detectada e ignorada |
| Arquivos `.doc` (formato legado) | Não suportados — salve como `.docx` no Word antes |

---

<div align="center">

**Ferramenta interna do ecossistema IXC Provedor · [conversor.ixcsoft.net](https://conversor.ixcsoft.net/)**

</div>
