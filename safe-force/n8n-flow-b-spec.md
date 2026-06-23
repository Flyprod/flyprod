# Flow B — Especificação n8n (Publicação Safe Force)

## Visão geral

Dois modos de execução no mesmo workflow (ou dois workflows separados):

- **Modo 1 — Gerar copy:** lê linhas `rascunho` com `imagem_url` preenchido → Claude gera copy → escreve na Sheet → muda estado para `aguarda_aprovacao`
- **Modo 2 — Publicar:** lê linhas `aprovado` → Templated render → publica em **Instagram + Facebook em simultâneo** → log na Sheet → muda estado para `publicado`

**Publicação sincronizada Instagram + Facebook:** o mesmo render do Templated.io é publicado nas duas plataformas. O caption pode ser o mesmo ou ter variações ligeiras (ex: hashtags diferentes por plataforma).

---

## Nodes Flow B — Modo 1 (Gerar copy)

```
[Schedule / Manual Trigger]
        ↓
[Google Sheets — Read]
  Filtro: estado = "rascunho" AND imagem_url ≠ ""
        ↓
[IF — tem linhas?]
  Não → Stop
  Sim ↓
[Loop Over Items]
        ↓
[HTTP Request — Claude API]
  POST https://api.anthropic.com/v1/messages
  Headers:
    x-api-key: {{$credentials.claudeApiKey}}
    anthropic-version: 2023-06-01
    content-type: application/json
  Body:
    {
      "model": "claude-opus-4-8",
      "max_tokens": 1024,
      "thinking": {"type": "adaptive"},
      "system": "{{contexto_cliente da linha}}",
      "messages": [
        {
          "role": "user",
          "content": "Gera a copy para um post de Instagram da Safe Force.\nTEMA: {{tema_do_post}}\nDevolve apenas o JSON com titulo, subtitulo, cta e caption."
        }
      ]
    }
        ↓
[Code — Parse JSON da resposta Claude]
  Extrai o JSON do content[0].text
        ↓
[Google Sheets — Update Row]
  Atualiza: titulo, subtitulo, cta, caption
  Muda: estado → "aguarda_aprovacao"
```

---

## Nodes Flow B — Modo 2 (Publicar)

```
[Schedule / Manual Trigger]
        ↓
[Google Sheets — Read]
  Filtro: estado = "aprovado"
        ↓
[IF — tem linhas?]
  Não → Stop
  Sim ↓
[Loop Over Items]
        ↓
[HTTP Request — Templated.io Render]
  POST https://api.templated.io/v1/render
  Headers:
    Authorization: Bearer {{$credentials.templatedApiKey}}
    Content-Type: application/json
  Body:
    {
      "template": "{{template_id}}",
      "format": "jpg",
      "async": false,
      "layers": {
        "imagem_fundo": {
          "image_url": "{{imagem_url}}",
          "object_fit": "cover",
          "object_position": "center"
        },
        "logo": { "image_url": "{{logo_url}}" },
        "titulo": { "text": "{{titulo}}" },
        "subtitulo": { "text": "{{subtitulo}}" },
        "cta": { "text": "{{cta}}" },
        "handle": { "text": "@safeforce.pt" }
      }
    }
        ↓
[Code — Extrair render_url da resposta Templated]
        ↓
[HTTP Request — Instagram: Criar container]
  POST https://graph.facebook.com/v21.0/{{ig_user_id}}/media
  Params:
    image_url: {{render_url}}
    caption: {{caption}}
    access_token: {{$credentials.metaAccessToken}}
        ↓
[Code — Extrair ig_creation_id]
        ↓
        ├──────────────────────────────────────────────────────┐
        ↓                                                      ↓
[Wait — 5 segundos]                          [HTTP Request — Facebook: Publicar foto]
  (aguardar antes de publicar IG)              POST https://graph.facebook.com/v21.0/{{fb_page_id}}/photos
        ↓                                        Params:
[HTTP Request — Instagram: Publicar]               url: {{render_url}}
  POST https://graph.facebook.com/v21.0/{{ig_user_id}}/media_publish    message: {{caption_fb}}
  Params:                                          access_token: {{$credentials.metaAccessToken}}
    creation_id: {{ig_creation_id}}            ↓
    access_token: {{$credentials.metaAccessToken}}  [Code — Extrair fb_post_id]
        ↓                                      ↓
        └──────────────────────────────────────┘
                           ↓
[Google Sheets — Update Row]
  Atualiza: ig_media_id, fb_post_id, data_publicado
  Muda: estado → "publicado"
        ↓
[IF — Erro em algum passo?]
  Sim → Google Sheets Update: estado → "erro" + notas com mensagem de erro
```

---

## Credenciais necessárias no n8n

| Credencial | Tipo | Onde obter |
|---|---|---|
| `claudeApiKey` | Header Auth | console.anthropic.com |
| `templatedApiKey` | Header Auth | templated.io → API Keys |
| `metaAccessToken` | Header Auth | Meta Graph API Explorer |
| Google Sheets | OAuth2 | n8n built-in Google OAuth |

---

## Versão Meta Graph API

Usar **v21.0**.

Permissões necessárias:
- `instagram_content_publish` — publicar no Instagram
- `instagram_basic` — ler info do IG
- `pages_show_list` — listar páginas
- `pages_manage_posts` — publicar no Facebook Page
- `pages_read_engagement` — ler info da Page

---

## Schema da Sheet — colunas adicionais para Facebook

Adicionar à Sheet existente:

| Coluna | Exemplo | Notas |
|---|---|---|
| `fb_page_id` | `123456789` | ID da Página Facebook do cliente |
| `caption_fb` | _(opcional)_ | Caption específica para FB — se vazio usa `caption` do IG |
| `fb_post_id` | `123_456` | Devolvido pela API após publicação. Preenchido pelo workflow. |

Se `caption_fb` estiver vazio, o workflow usa o `caption` (o mesmo do Instagram).

---

## Notas de implementação

1. **Publicação paralela IG + FB** — os dois ramos correm em paralelo após o render Templated, sem dependência entre si.
2. **Mesmo render** — a imagem publicada no Instagram e no Facebook é a mesma (o URL do Templated render).
3. **Facebook endpoint** — `POST /{page-id}/photos` com `url` + `message`. Um único passo (sem container/publish como o IG).
4. **`async: false` no Templated** — recebe o URL do render na resposta imediata.
5. **Logo Safe Force** — upload para Cloudinary uma vez, URL fixo no workflow.
6. **Rate limits** — Instagram: 25 posts/24h. Facebook: sem limite rígido mas evitar mais de 5/dia por boas práticas.
7. **Erro handling** — o node de erro escreve `erro` na Sheet com a mensagem (IG ou FB) para reprocessamento manual.
