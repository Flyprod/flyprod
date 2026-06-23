# Google Sheet — Esquema (Safe Force · Piloto)

## Nome sugerido do ficheiro
`Flyprod — Posts Instagram`

## Nome da aba
`Safe Force`

---

## Colunas (ordem exata)

| Coluna | Exemplo | Notas |
|---|---|---|
| `cliente` | Safe Force | Nome do cliente. Fixo por aba (pode ser coluna ou metadado). |
| `template_id` | `abc123xyz` | ID do template Templated.io criado para este cliente. |
| `ig_user_id` | `17841400000000000` | Instagram User ID (obtido via Graph API). |
| `contexto_cliente` | _(ver ficheiro contexto-claude.md)_ | Briefing completo que alimenta o Claude. Pode ser numa coluna longa ou num separador dedicado. |
| `imagem_url` | `https://res.cloudinary.com/...` | URL Cloudinary do pool aprovado. Preenchido no Fluxo A. |
| `titulo` | _(vazio → preenchido por Claude)_ | Headline gerada pelo Claude. Máx 8 palavras. |
| `subtitulo` | _(vazio → preenchido por Claude)_ | Subtítulo gerado pelo Claude. Máx 20 palavras. |
| `cta` | `Fala connosco` | Gerado pelo Claude ou fixo por cliente. |
| `caption` | _(vazio → preenchido por Claude)_ | Legenda completa do post Instagram (até 2200 chars). Inclui hashtags. |
| `estado` | `rascunho` | **Valores válidos:** `rascunho` / `aprovado` / `publicado` |
| `data_publicacao` | `2024-02-15` | Data alvo de publicação. Pode ser usado para trigger agendado. |
| `fb_page_id` | `123456789` | ID da Página Facebook do cliente. |
| `caption_fb` | _(opcional)_ | Caption específica para Facebook. Se vazio, usa o campo `caption`. |
| `ig_media_id` | `17854360000000000` | Devolvido pela Meta API após publicação IG. Preenchido pelo workflow. |
| `fb_post_id` | `123456789_987654321` | Devolvido pela Meta API após publicação FB. Preenchido pelo workflow. |
| `data_publicado` | `2024-02-15 10:30` | Timestamp real de publicação. Preenchido pelo workflow. |
| `notas` | _(opcional)_ | Campo livre para anotações humanas (ex: "aprovado com edição manual"). |

---

## Fluxo de estado

```
[Fluxo A: asset batch]
  Higgsfield → Cloudinary → preencher imagem_url → estado: rascunho

[Fluxo B: publicação - trigger: estado = aprovado]
  Claude gera titulo + subtitulo + cta + caption
  → escreve de volta na sheet (colunas Claude)
  → muda estado para: aguarda_aprovacao (ou mantém rascunho)

[Revisão humana]
  Lê titulo + subtitulo + cta + caption na sheet
  → muda estado para: aprovado

[Fluxo B parte 2 - trigger: estado = aprovado]
  Templated render → imagem final
  → Meta /media → /media_publish
  → escreve ig_media_id + data_publicado
  → muda estado para: publicado
```

**Nota:** O Fluxo B pode ser um único workflow n8n com dois modos de execução (ou dois workflows separados), com o gate de aprovação como ponto de pausa humana.

---

## Configuração recomendada na Sheet

- **Linha 1:** cabeçalhos (nomes das colunas acima)
- **Validação de dados** na coluna `estado`: lista pendente com `rascunho`, `aguarda_aprovacao`, `aprovado`, `publicado`
- **Formatação condicional:** linha verde quando `estado = publicado`, amarela quando `aguarda_aprovacao`
- **Partilha:** acesso de editor para a conta de serviço usada pelo n8n (Google Sheets OAuth)
