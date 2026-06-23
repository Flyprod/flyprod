# Flow B — Especificação n8n (Publicação Safe Force)

## Visão geral

Dois modos de execução no mesmo workflow (ou dois workflows separados):

- **Modo 1 — Gerar copy:** lê linhas `rascunho` com `imagem_url` preenchido → Claude gera copy → escreve na Sheet → muda estado para `aguarda_aprovacao`
- **Modo 2 — Publicar:** lê linhas `aprovado` → Templated render → Meta publish → log na Sheet → muda estado para `publicado`

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
[HTTP Request — Meta: Criar container]
  POST https://graph.facebook.com/v21.0/{{ig_user_id}}/media
  Params:
    image_url: {{render_url}}
    caption: {{caption}}
    access_token: {{$credentials.metaAccessToken}}
        ↓
[Code — Extrair creation_id]
        ↓
[Wait — 5 segundos]
  (Meta recomenda esperar antes de publicar)
        ↓
[HTTP Request — Meta: Publicar]
  POST https://graph.facebook.com/v21.0/{{ig_user_id}}/media_publish
  Params:
    creation_id: {{creation_id}}
    access_token: {{$credentials.metaAccessToken}}
        ↓
[Google Sheets — Update Row]
  Atualiza: ig_media_id, data_publicado
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
| `metaAccessToken` | Header Auth | Meta Graph API Explorer (permissão `instagram_content_publish`) |
| Google Sheets | OAuth2 | n8n built-in Google OAuth |

---

## Versão Meta Graph API

Usar **v21.0** (versão estável no momento da implementação — confirmar em developers.facebook.com/docs/graph-api/changelog se necessário).

Permissões necessárias:
- `instagram_content_publish`
- `instagram_basic`
- `pages_show_list`

---

## Notas de implementação

1. **`async: false` no Templated** — recebe o URL do render na resposta imediata. Sem webhook necessário para o piloto.
2. **Logo Safe Force** — fazer upload para Cloudinary uma única vez e fixar o URL no workflow (ou na Sheet como coluna `logo_url`).
3. **Rate limit Meta** — máximo 25 posts por 24h por conta Instagram. Para o piloto não é problema.
4. **Erro handling** — o node de erro escreve o estado `erro` na Sheet para reprocessamento manual.
5. **Teste antes de ligar** — testar o Modo 2 com `async: false` e um `ig_user_id` de conta de teste antes de apontar para a conta real do Safe Force.
