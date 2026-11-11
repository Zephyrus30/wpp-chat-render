---
name: wpp-chat-render
description: >
  Converte texto bruto de conversa do WhatsApp (copiado e colado) em interface visual com balões
  exatamente como no WhatsApp: operadores à direita em verde, clientes à esquerda em cinza,
  mensagens de sistema centralizadas em itálico. Remove artefatos de IA da transcrição,
  normaliza nomes de participantes e separa corretamente cada fala. Gera HTML standalone (pode
  ser aberto no browser, salvo como arquivo, ou embutido num iframe de e-mail) e opcionalmente
  PDF via jsPDF. Usar sempre que o usuário colar um texto de conversa do WhatsApp e pedir para
  visualizar, formatar, exportar, embutir em e-mail, ou gerar PDF da conversa.
---

# WhatsApp Chat Render

## Persona: revisor de olho clínico

Antes de qualquer saída, leia a conversa inteira do começo ao fim como um revisor perfeccionista
faria. Não existe "parece certo" — existe "está correto ou não está". Percorra cada linha
verificando: quem fala aqui? isso é sistema ou mensagem? tem artefato de IA? o nome está
na forma mais longa? Só após essa leitura meticulosa, gere o HTML. Prefira demorar mais
e acertar do que sair rápido e errar um nome ou uma bolha.

Dado um texto bruto de conversa do WhatsApp, produza um HTML standalone com a conversa
renderizada em balões, fiel ao visual do WhatsApp.

## Como funciona o formato de entrada

O texto de entrada pode vir de três fontes principais:

**1. Exportação do WhatsApp** — linhas com data/hora:
```
12/03/2024, 14:32 - João Silva: Olá, boa tarde!
12/03/2024, 14:33 - CLIENTE: Boa tarde, preciso de ajuda
```

**2. Transcrição de chat de suporte** — sem timestamp, identificação por prefixo de nome:
```
João Silva: Olá, boa tarde!
CLIENTE RENAN VARGAS: Boa tarde, preciso de ajuda
Operador João Silva transferiu o atendimento para Maria.
```

**3. Chatbot/automação** — prefixos ALL CAPS com dois-pontos, sem operador humano:
```
CHATBOT: Olá, seja bem-vindo(a) ao canal de atendimento da *Base Incorporações!*
CLIENTE: Falar com o SAC
CHATBOT: Para qual área deseja atendimento?
CLIENTE: Assistência Técnica
```
Pode ter um bloco de preamble antes do chat, como:
```
"Ocorrência aberta pelo Chatbot em 14/10/2026 12:29:34

O contato foi feito pelo Chat - Whatsapp pelo número: 556182589818

Transcrição
CHATBOT: ...
```
O preamble inteiro ("Ocorrência aberta pelo Chatbot...") é uma mensagem de sistema.

A skill lida com todos os formatos, e com variações entre eles.

## Regras de identificação de participantes

### Operador humano (bolha verde, alinhado à direita)
Aparece como **"Nome Sobrenome:"** em capitalização mista — pelo menos duas palavras, cada uma
com inicial maiúscula e demais minúsculas. Exemplo: `João Silva:`, `Maria Aparecida:`

### CHATBOT (bolha verde com nome em azul-petróleo, alinhado à direita)
Aparece como `CHATBOT:` no início da linha — ALL CAPS com dois-pontos. É o bot de atendimento
automático. Renderizar do mesmo lado do operador (direita), mas com cor diferente no nome
(`#0891B2` / ciano) para distinguir de operador humano.

### Cliente (bolha cinza, alinhado à esquerda)
Aparece de duas formas:
- **ALL CAPS sem dois-pontos** no início de linha: `RENAN VARGAS GUIMARAES Boa tarde...`
- **`CLIENTE:`** no início de linha (formato chatbot): `CLIENTE: Falar com o SAC`

Aparece em **TODAS MAIÚSCULAS** no **início de uma linha**, seguido de texto (sem `:` imediato — exceto `CLIENTE:`).
Exemplos: `RENAN VARGAS GUIMARAES Boa tarde...`, `CLIENTE oi`

**Regra crítica de posição**: o padrão ALL CAPS só é reconhecido como speaker se estiver no
início da linha. Um nome de cliente mencionado dentro da fala de um operador
(ex: `Ellen Cardoso: Olá, NIVA DOS ANJOS LIMA, tudo bem?`) **não** é um novo speaker —
é parte da mensagem do operador. Nunca quebre uma mensagem de operador no meio.

### Mensagem de sistema (centralizada, fundo cinza, itálico)
Frases geradas automaticamente pelo sistema de chat. Padrões reconhecidos:
- Transferências: `Operador X transferiu o atendimento para Y.`
- Alterações de status: `Solicitação alterada para [status].`
- Desconexão: `A outra parte saiu ou foi desconectada. A conversa foi encerrada.`
- Saída do atendente: `O atendente saiu ou foi desconectado.`
- Preamble de chatbot: `"Ocorrência aberta pelo Chatbot em DD/MM/AAAA HH:MM:SS` seguido de
  até 6 linhas de contexto (número do contato, "Transcrição", etc.) — capturar bloco inteiro
- Abertura: `Aberto pelo Chat: WhatsApp | ...` — pode ter `*` no início (`*Aberto pelo Chat:`),
  pode ter o texto na linha seguinte (não na mesma linha que "Chat:"), e pode ter até 8 linhas
  de continuação incluindo linhas em branco. Capturar o bloco inteiro como sistema.
- Encerramento: `Atendimento Finalizado ...` — qualquer variação
- Transferência de bate-papo: `Transferência de bate-papo iniciada...`

## Artefatos de IA na transcrição

Sistemas de transcrição automática por IA frequentemente inserem "IA" no texto. Remova-os
seguindo estas regras — análise **toda a conversa primeiro**, depois aplique:

1. **`IA` antes de nome próprio** (espaço + inicial maiúscula): remover o `IA ` inteiro.
   Ex: `IA Fulano disse` → `Fulano disse`

2. **Sufixo `IA` em palavra** (letras minúsculas ou `*` antes): remover só o `IA`.
   Ex: `retornoIA` → `retorno`, `verificarIA` → `verificar`, `retorno*IA` → `retorno*`

3. **Sufixo `IA` em palavra maiúscula** (letras maiúsculas antes): remover só o `IA`, manter a base.
   Ex: `GUIMARAESIA` → `GUIMARAES`, `LOPESIA` → `LOPES`

4. **`letra+dígitoIA`** (extensão de arquivo ou código): remover o `IA` inteiro.
   Ex: `mp4IA` → `mp4`, `h264IA` → `h264`

5. **`dígitoIA` isolado** (número de unidade, não precedido de letra): substituir por `dígitoA`.
   Ex: `1IA` → `1A`, `Suite 3IA` → `Suite 3A`

**Regra crítica**: aplique remoção de IA **somente no texto das mensagens**, nunca nos nomes
dos participantes identificados como speakers. Identificar speakers antes de limpar.

## Normalização de nomes

Se o mesmo participante aparece com nome curto e longo na mesma conversa
(`RENAN VARGAS` e `RENAN VARGAS GUIMARAES`), use sempre a versão mais longa — é a que tem
mais informação. Isso evita que a mesma pessoa apareça como dois participantes diferentes.

**Deduplicação de nome repetido antes de `:`**: quando o sistema de chat insere o nome do
operador duplicado na mesma linha antes do `:` (ex: `Dionilia Dionilia Silva:`), remova a
primeira ocorrência. Isso **só vale para duplicações na mesma linha** (separadas por espaço
ou tab). Nunca colapsar nomes iguais que estão em linhas diferentes — ex: "Luane Lima" no
fim do preamble e "Luane Lima:" no início da primeira mensagem são ocorrências distintas.

## Normalização de formato antes da detecção de speakers

Antes de identificar quem fala, normalize o texto bruto para evitar falsos splits:

1. **Strip de espaço inicial por linha**: linhas com recuo (ex: `    GIOVANNI AVELAR mensagem`)
   devem ter o recuo removido para que o padrão ALL CAPS seja reconhecido no início da linha.

2. **Formato chatbot numa só linha**: se todas as falas estiverem numa linha única sem quebras,
   injete `\n` antes de cada label ALL CAPS seguido de `:` (ex: `CHATBOT: ... CLIENTE: ...`
   → `CHATBOT: ...\nCLIENTE: ...`). Só para labels de palavra única.

3. **Frases de sistema coladas a palavras anteriores**: injete `\n` antes de frases como
   "Solicitação alterada" ou "Você está sendo atendido" quando aparecerem sem quebra de linha
   após a fala anterior (ex: `SilvaSolicitação alterada` → `Silva\nSolicitação alterada`).

4. **"Você está sendo atendido por" seguido de operador na próxima linha**: quando a frase
   de sistema termina com "por" e a próxima linha é `Nome Sobrenome: mensagem`, injete o nome
   na mesma linha da frase de sistema E mantenha `Nome Sobrenome:` na linha seguinte:
   ```
   Você está sendo atendido por          →   Você está sendo atendido por Luane Lima
   Luane Lima: Bom dia!                      Luane Lima: Bom dia!
   ```

## Tratamento do preamble (texto antes do primeiro speaker)

Todo o texto que aparece antes do primeiro speaker identificado é **sempre** uma mensagem de
sistema — independente do conteúdo. Não tente separar partes do preamble em bolhas diferentes.
Renderize o preamble inteiro como um único bloco cinza centralizado.

Exemplos de preamble comuns:
- Bloco "Aberto pelo Chat:" com data, número de contato, protocolo e nome do operador
- Bloco "Ocorrência aberta pelo Chatbot" com data e transcrição

## Output: HTML standalone

Gere um arquivo `.html` completo e autocontido (sem dependências externas, exceto jsPDF se PDF
estiver ativado) com:

- `<meta charset="UTF-8">` e `<meta name="viewport" content="width=device-width">`
- Todo o CSS inline no `<head>`
- A conversa renderizada no `<body>`
- Botão "Baixar PDF" opcional (ver seção PDF)

### CSS dos balões (use exatamente estes estilos)

```css
body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  background: #ECE5DD;  /* fundo WhatsApp */
  margin: 0; padding: 16px;
  max-width: 680px; margin-inline: auto;
}
.wpp-chat {
  display: flex; flex-direction: column; gap: 7px;
}
.wpp-msg {
  max-width: 82%; padding: 7px 11px;
  border-radius: 12px;
  display: flex; flex-direction: column; gap: 2px;
  word-break: break-word;
  box-shadow: 0 1px 2px rgba(0,0,0,.13);
}
.wpp-msg.cliente {
  align-self: flex-start;
  background: #FFFFFF;
  border-bottom-left-radius: 2px;
}
.wpp-msg.operador {
  align-self: flex-end;
  background: #D9FDD3;
  border-bottom-right-radius: 2px;
}
.wpp-nome {
  font-size: 11px; font-weight: 700;
  color: #667085; letter-spacing: .01em;
}
.wpp-msg.operador .wpp-nome { color: #2E6B15; }
.wpp-texto {
  font-size: 13px; line-height: 1.5; color: #111827; white-space: pre-line;
}
.wpp-msg.sistema {
  align-self: center;
  background: rgba(0,0,0,0.09);
  border-radius: 8px;
  max-width: 92%;
  text-align: center;
  padding: 5px 12px;
}
.wpp-msg.sistema .wpp-texto {
  font-size: 11.5px; color: #555; font-style: italic;
}
/* Timestamp (se presente) */
.wpp-ts {
  font-size: 10px; color: #999;
  align-self: flex-end; margin-top: 1px;
}
```

### Estrutura HTML da conversa

```html
<div class="wpp-chat">
  <!-- Mensagem de sistema -->
  <div class="wpp-msg sistema">
    <span class="wpp-texto">Aberto pelo Chat: WhatsApp</span>
  </div>

  <!-- Mensagem do cliente (esquerda) -->
  <div class="wpp-msg cliente">
    <span class="wpp-nome">RENAN VARGAS GUIMARAES</span>
    <span class="wpp-texto">Olá, boa tarde!</span>
    <!-- <span class="wpp-ts">14:32</span>  — se tiver timestamp -->
  </div>

  <!-- Mensagem do operador (direita) -->
  <div class="wpp-msg operador">
    <span class="wpp-nome">João Silva</span>
    <span class="wpp-texto">Boa tarde! Em que posso ajudar?</span>
  </div>
</div>
```

Escape sempre os caracteres HTML (`&`, `<`, `>`, `"`) **antes** de aplicar formatação WhatsApp.
Depois aplique `wppFmt(escapedText)`:

```javascript
function wppFmt(s) {
  return s
    .replace(/```([\s\S]*?)```/g, '<code style="display:block;background:rgba(0,0,0,.07);padding:2px 6px;border-radius:3px;white-space:pre-wrap;font-size:12px">$1</code>')
    .replace(/`([^`\n]+)`/g, '<code style="background:rgba(0,0,0,.07);padding:0 3px;border-radius:3px;font-size:12px">$1</code>')
    .replace(/\*([^*\n]+)\*/g, '<b>$1</b>')
    .replace(/(^|[^A-Za-z0-9])_([^_\n]+)_(?=[^A-Za-z0-9]|$)/g, '$1<em>$2</em>')
    .replace(/~([^~\n]+)~/g, '<s>$1</s>');
}
```

**Atenção no italic**: o padrão `_texto_` do WhatsApp exige que o `_` esteja adjacente a
caractere não-alfanumérico. Use `(^|[^A-Za-z0-9])_([^_\n]+)_(?=[^A-Za-z0-9]|$)` em vez de
`_([^_\n]+)_` para evitar destruir underscores em nomes de arquivo como
`2026_09_02_11_19_32_427.mp4` (os underscores entre dígitos seriam tratados como marcadores
de itálico e removidos da exibição).

Ordem obrigatória: triple-backtick → backtick → `*bold*` → `_italic_` → `~riscado~`.

## Output: iframe para e-mail

Para embutir num e-mail como iframe, gere o HTML igual ao acima, mas:
- Remova o botão de PDF
- Adicione `<base target="_blank">` no `<head>`
- Use `scrolling="no"` e `height` fixo no iframe tag (sugerir ~`400px` ao usuário ou calcular
  aproximadamente `número_de_mensagens × 55px`)
- Forneça o tag `<iframe>` pronto:

```html
<iframe src="conversa.html" width="100%" height="400"
        scrolling="no" frameborder="0"
        style="border:none;border-radius:8px;overflow:hidden;">
</iframe>
```

## Output: PDF (opcional)

Se o usuário pedir PDF, adicione um botão no HTML que usa jsPDF (CDN) para gerar o PDF
no browser. Use a lógica de layout descrita abaixo.

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
<button onclick="gerarPdf()">Baixar PDF</button>
<script>
function gerarPdf() {
  const { jsPDF } = window.jspdf;
  const doc = new jsPDF({ unit: 'mm', format: 'a4' });
  const W = 170; // largura útil (210 - 2×20mm margem)
  const MAX_W = W * 0.78;
  const PAD_X = 3.5, PAD_Y = 3, NAME_H = 3.8, GAP = 2.5;
  const LINE_H = 4.5; // PDF_TEXT_LINE_H
  let y = 20;

  // Para cada mensagem no array `msgs` (estrutura: {speaker, text, isClient, isSystem})
  msgs.forEach(({ speaker, text, isClient, isSystem }) => {
    if (isSystem) {
      doc.setFont('helvetica', 'italic');
      doc.setFontSize(8);
      const lines = doc.splitTextToSize(text.replace(/\n+/g,' ').trim(), W * 0.68);
      const bh = (lines.length - 1) * (LINE_H - 0.3) + 2.8 + PAD_Y * 2;
      const bw = W * 0.68;
      const bx = 20 + (W - bw) / 2;
      if (y + bh > 280) { doc.addPage(); y = 20; }
      doc.setFillColor(225, 228, 234);
      doc.roundedRect(bx, y, bw, bh, 2, 2, 'F');
      doc.setTextColor(100, 110, 125);
      lines.forEach((l, i) => doc.text(l, bx + PAD_X, y + PAD_Y + 2.0 + i * (LINE_H - 0.3)));
      y += bh + GAP;
      return;
    }
    doc.setFont('helvetica', 'normal'); doc.setFontSize(9);
    const inner = MAX_W - PAD_X * 2;
    const tLines = doc.splitTextToSize(text.replace(/\n+/g,' ').trim(), inner);
    doc.setFont('helvetica', 'bold'); doc.setFontSize(8);
    const nLines = speaker ? doc.splitTextToSize(speaker, inner) : [];
    const nameH = nLines.length ? nLines.length * NAME_H + 1.5 : 0;
    const bh = nameH + (tLines.length - 1) * LINE_H + 3.0 + PAD_Y * 2;
    doc.setFont('helvetica', 'normal'); doc.setFontSize(9);
    const txtW = Math.max(0, ...tLines.map(l => doc.getTextWidth(l)));
    doc.setFont('helvetica', 'bold'); doc.setFontSize(8);
    const nmW = nLines.length ? Math.max(0, ...nLines.map(l => doc.getTextWidth(l))) : 0;
    const bubW = Math.min(MAX_W, Math.max(txtW, nmW) + PAD_X * 2 + 1);
    const bx = isClient ? 20 : 20 + W - bubW;
    if (y + bh > 280) { doc.addPage(); y = 20; }
    doc.setFillColor(...(isClient ? [240, 242, 245] : [217, 245, 200]));
    doc.roundedRect(bx, y, bubW, bh, 2.5, 2.5, 'F');
    let ty = y + PAD_Y + 2.3;
    if (nLines.length) {
      doc.setFont('helvetica', 'bold'); doc.setFontSize(8);
      doc.setTextColor(isClient ? 80 : 46, isClient ? 88 : 107, isClient ? 100 : 21);
      nLines.forEach((l, i) => doc.text(l, bx + PAD_X, ty + i * NAME_H));
      ty += nLines.length * NAME_H + 1.5;
    }
    doc.setFont('helvetica', 'normal'); doc.setFontSize(9);
    doc.setTextColor(25, 30, 40);
    tLines.forEach((l, i) => doc.text(l, bx + PAD_X, ty + i * LINE_H));
    y += bh + GAP;
  });

  doc.save('conversa.pdf');
}
</script>
```

## Procedimento de geração

1. Leia o texto de entrada completo antes de qualquer processamento.
2. Execute a limpeza de IA em toda a conversa primeiro (análise global).
3. Detecte todos os speakers e normalize nomes (versão mais longa).
4. Construa o array de mensagens `msgs` com `{speaker, text, isClient, isSystem}`.
5. Gere o HTML com os balões usando os estilos especificados.
6. Se o usuário pediu PDF, inclua o script jsPDF e o array `msgs` como JSON no HTML.
7. Se o usuário pediu iframe, inclua o tag `<iframe>` pronto com dimensões sugeridas.
8. Entregue como Artifact (HTML) para visualização imediata no browser.

## Código de referência — wpp-render.js

Esta seção contém a implementação JS usada pelo SAC app. Quando uma regra acima for
atualizada, atualize o bloco correspondente aqui e copie o arquivo para `wpp-render.js`.

```javascript
function _stripIaFromText(t) {
  return t
    .replace(/\bIA\s+(?=[A-ZÁÀÂÃÉÊÍÓÔÕÚÇ][A-ZÁÀÂÃÉÊÍÓÔÕÚÇa-záàâãéêíóôõúç])/g, '')  // IA antes de nome próprio
    .replace(/([A-Za-z])([0-9]+)IA\b/g, '$1$2')  // ext. arquivo: mp4IA → mp4
    .replace(/([0-9])IA\b/g, '$1A')               // 1IA → 1A (nº de unidade)
    .replace(/([a-záàâãéêíóôõúç?!.,;:()*])IA\b/g, '$1');  // palavraIA → palavra, palavra*IA → palavra*
}

function wppExtractSys(text) {
  const reSys = /(Operador\s+\S[^.]*?transferiu[^.]+?\.(?:\s*Voc[eê][^.]*?atendido[^.]*?\.)?|Solicita[cç][aã]o\s+alterada\s+para\s+[^\n.]{3,80}\.?|Você\s+está\s+sendo\s+atendido\s+(?:pelo\s+operador|por)\s+[^\n.]{2,60}\.?|A\s+outra\s+parte\s+saiu[^.]*?\.[^.]*?encerrada\.?|Transfer[eê]ncia\s+de\s+bate[- ]papo[^.]*?\.[^.]*?\.|O\s+atendente\s+saiu[^\n]*(?:\n(?:Protocolo\s+de\s+Atendimento:|Atendimento\s+Finalizado)[^\n]*)*|\*?Aberto\s+pelo\s+Chat:[^\n]*(?:\n(?!Você\s+está)[^\n]*){0,15}|Atendimento\s+Finalizado[^\n]*|"?Ocorr[eê]ncia\s+aberta\s+pelo\s+Chatbot[^\n]*(?:\n[^\n]*){0,6})/gi;
  const parts = []; let last = 0, m;
  while ((m = reSys.exec(text)) !== null) {
    const before = text.slice(last, m.index).trim();
    if (before) parts.push({ text: before, isSystem: false });
    parts.push({ text: m[0].trim(), isSystem: true });
    last = m.index + m[0].length;
  }
  const after = text.slice(last).trim();
  if (after) parts.push({ text: after, isSystem: false });
  return parts.length ? parts : [{ text, isSystem: false }];
}
```

Para adicionar um novo padrão de artefato de IA:
1. Adicione a regra em prosa na seção "Artefatos de IA" acima.
2. Adicione a linha de `.replace(...)` correspondente em `_stripIaFromText` acima.
3. Copie `wpp-render.js` atualizado para o servidor — apenas esse arquivo.

## Exemplo de saída esperada

**Entrada:**
```
Aberto pelo Chat: WhatsApp | Empreendimento X
João Silva: Bom dia, posso ajudar?
RENAN VARGAS GUIMARAES Sim, tenho uma dúvida sobre o apartamento.
João Silva: Claro! Qual a sua unidade?
RENAN VARGAS GUIMARAES Unidade 1A do bloco B.
```

**Saída HTML simplificada:**
- Sistema: "Aberto pelo Chat: WhatsApp | Empreendimento X" (centro, cinza)
- Operador João Silva: "Bom dia, posso ajudar?" (direita, verde)
- Cliente RENAN VARGAS GUIMARAES: "Sim, tenho uma dúvida sobre o apartamento." (esquerda, branco)
- Operador João Silva: "Claro! Qual a sua unidade?" (direita, verde)
- Cliente RENAN VARGAS GUIMARAES: "Unidade 1A do bloco B." (esquerda, branco)
