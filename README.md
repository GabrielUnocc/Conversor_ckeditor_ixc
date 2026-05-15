# Conversor DOCX → CKEditor | IXC Provedor

Ferramenta web para converter documentos Word (`.docx`) em HTML compatível com o **CKEditor 4** e **CKEditor 5** do sistema IXC Provedor, com bordas corretas e sem quebramento na impressão via TCPDF.

---

## Problema que resolve

Provedores de internet que usam o IXC Provedor precisam cadastrar modelos de contrato no editor de texto do sistema. Quando o contrato vem em Word e é colado diretamente no CKEditor, dois problemas acontecem:

- **Tabelas quebram na impressão** — o Word gera estilos `mso-*` incompatíveis, e o CKEditor 5 envolve tabelas com `<figure class="table">` que o TCPDF (gerador de PDF do IXC) não renderiza corretamente, exibindo o texto das células verticalmente, letra por letra
- **Formatação perdida** — espaços entre palavras, negrito, itálico, alinhamento e células mescladas são perdidos na conversão

Esta ferramenta resolve os dois problemas lendo o XML interno do arquivo Word diretamente, sem nenhuma conversão intermediária.

---

## Como usar

1. Baixe o arquivo `conversor_ixc.html`
2. Abra no navegador (Chrome ou Edge recomendados)
3. Selecione a versão do CKEditor usada no seu IXC (**CKEditor 5** ou **CKEditor 4**)
4. Arraste o arquivo `.docx` ou clique em **Selecionar arquivo**
5. Confira o resultado no **Preview**
6. Clique em **Copiar HTML**
7. No IXC Provedor: abra o modelo de contrato → clique no botão `</>` (código fonte) → selecione tudo → cole → OK → Salvar

---

## Funcionalidades

- **Suporte a CKEditor 4 e 5** — alterna entre os dois formatos com um clique, reconvertendo o arquivo automaticamente
- **Tabelas com bordas corretas** — gera o formato nativo de cada versão do editor, com `border-color:#000000` (CK5) ou `border="1"` (CK4)
- **Células mescladas** — preserva `colspan` e `rowspan` extraídos do XML real do Word
- **Formatação de texto** — negrito, itálico e sublinhado preservados
- **Alinhamento** — centralizado, direita e justificado lidos do parágrafo e aplicados ao HTML
- **Espaços corretos** — cada palavra e espaço são preservados, evitando palavras coladas
- **Histórico de conversões** — últimas 10 conversões ficam salvas na sessão
- **Funciona offline** — nenhum dado é enviado para servidores; tudo roda no browser

---

## Tecnologias

| Tecnologia | Uso |
|---|---|
| HTML + CSS | Interface e layout |
| JavaScript | Lógica de conversão e UI |
| [JSZip](https://stuk.github.io/jszip/) | Leitura do `.docx` (que é um ZIP) no browser |
| DOMParser (nativo) | Interpretação do XML interno do Word |

Não há backend, framework, build step ou instalação necessária.

---

## Como funciona

Um arquivo `.docx` é internamente um ZIP contendo XMLs no formato OOXML (padrão Microsoft). O arquivo principal é `word/document.xml`, que descreve todo o conteúdo do documento.

A ferramenta usa o **JSZip** para abrir esse ZIP diretamente no browser, lê o `document.xml` com o **DOMParser** nativo, e percorre a estrutura XML:

- `<w:p>` → parágrafos → `<p style="text-align:...">` 
- `<w:tbl>` → tabelas → `<figure class="table">` (CK5) ou `<table border="1">` (CK4)
- `<w:tr>` → linhas → `<tr>`
- `<w:tc>` → células → `<td>` com `colspan` lido do `w:gridSpan`
- `<w:r>` → runs de texto → texto com `<strong>`, `<em>`, `<u>`
- `<w:pPr><w:jc>` → alinhamento → `text-align` no CSS

---

## Deploy em VM (produção)

### Nginx

```bash
sudo apt install nginx
sudo cp conversor_ixc.html /var/www/html/
```

Acesse em: `http://IP-DA-VM/conversor_ixc.html`

### Python (rápido para testes)

```bash
python3 -m http.server 8080
```

Acesse em: `http://IP-DA-VM:8080/conversor_ixc.html`

> **Nota:** Em HTTP puro (sem HTTPS), a API de clipboard do browser é bloqueada por segurança. A ferramenta detecta isso automaticamente e usa um fallback via `execCommand` que funciona normalmente em HTTP.

---

## Estrutura do repositório

```
conversor_ixc.html   # ferramenta completa (único arquivo)
CLAUDE.md            # contexto técnico para o Claude
README.md            # este arquivo
```

---

## Formatos de saída

### CKEditor 5
```html
<figure class="table" style="width:100%;">
  <table class="ck-table-resized" style="border-style:solid;">
    <colgroup><col style="width:50%;"><col style="width:50%;"></colgroup>
    <tbody>
      <tr>
        <td style="border-color:#000000;">Conteúdo</td>
        <td style="border-color:#000000;" colspan="2">Mesclado</td>
      </tr>
    </tbody>
  </table>
</figure>
```

### CKEditor 4
```html
<table border="1" cellpadding="1" cellspacing="1" style="width:100%;">
  <tbody>
    <tr>
      <td>Conteúdo</td>
      <td colspan="2">Mesclado</td>
    </tr>
  </tbody>
</table>
```

---

## Limitações conhecidas

- Imagens dentro do `.docx` não são convertidas
- Listas numeradas e com marcadores são convertidas como parágrafos simples
- Estilos de fonte (tamanho, família) não são preservados — o CKEditor aplica seus próprios estilos
- Arquivos `.doc` (formato antigo, binário) não são suportados — apenas `.docx`

---

## Licença

MIT
