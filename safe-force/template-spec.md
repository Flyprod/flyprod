# Safe Force — Especificação do Template Templated.io

## Contexto do cliente
**Sectores:** Salvamento aquático · Formação · Vigilância · Controlo de acesso  
**Tom:** Profissional, direto, confiança técnica. Nunca informal ou excessivamente publicitário.  
**Handle Instagram:** a preencher (ex: @safeforce.pt)

---

## Dimensões
- **Formato:** Feed Instagram vertical — **1080 × 1350 px** (4:5)
- Alternativa quadrada 1080 × 1080 disponível se necessário, mas 4:5 ocupa mais espaço no feed

---

## Layers — contrato do payload (nomes exatos a usar no editor Templated.io)

| Nome da layer | Tipo | Descrição | Notas de design |
|---|---|---|---|
| `imagem_fundo` | image | Imagem de fundo gerada via Higgsfield (pool aprovado) | `object_fit: cover`, `object_position: center`. Gerar em 1080×1350. |
| `overlay` | rectangle | Gradiente/sobreposição escura sobre a imagem de fundo | Opacidade ~55–70%. Garante legibilidade do texto. Cor: #000000 ou azul escuro SafeForce. |
| `logo` | image | Logótipo do Safe Force | Canto superior esquerdo ou direito. `object_fit: contain`. |
| `titulo` | text | Headline principal do post (gerado por Claude) | Fonte bold, tamanho grande, cor branca. Máx 8 palavras. |
| `subtitulo` | text | Desenvolvimento do título (gerado por Claude) | Fonte regular, tamanho médio, cor branca/cinza claro. Máx 20 palavras. |
| `cta` | text | Call to action (gerado por Claude ou fixo) | Ex: "Sabe mais no link da bio" · "Fala connosco" · "Inscrições abertas". Fonte bold pequena. |
| `handle` | text | @ do cliente no Instagram | Canto inferior. Ex: `@safeforce.pt`. Cor: branca com opacidade. |

---

## Layout visual (wireframe em texto)

```
┌─────────────────────────────────┐
│  [logo]                         │  ← canto sup. esq.
│                                 │
│                                 │
│   [imagem_fundo + overlay]      │
│                                 │
│                                 │
│  [titulo]                       │  ← alinhado à esq., ~60% altura
│  [subtitulo]                    │
│                                 │
│  [cta]                ──────    │  ← fundo diferenciador (ex: badge)
│                  [handle]       │  ← canto inf. dir.
└─────────────────────────────────┘
```

---

## Paleta recomendada (Safe Force)
- Fundo overlay: `#0A1628` (azul marinho profundo) ou `#000000`
- Texto principal: `#FFFFFF`
- Destaque/CTA: `#1E90FF` (azul elétrico) ou `#FF6B00` (laranja urgência — usar com moderação)
- Handle: `#FFFFFF` com opacidade 60%

---

## Tipos de imagem Higgsfield para o pool (por sector)

| Sector | Sugestões de prompts Higgsfield |
|---|---|
| Salvamento aquático | Nadador-salvador em posição de alerta junto ao mar; piscina profissional; operação de resgate em água |
| Formação | Sala de formação com equipamento de segurança; instrutor com grupo; simulação de emergência |
| Vigilância | Posto de controlo noturno profissional; ronda de segurança exterior; câmeras e painel de monitorização |
| Controlo de acesso | Entrada corporativa com barreira; leitor biométrico; segurança em porta de edifício |

**Regra:** gerar em 1080×1350, sem texto dentro da imagem, fundo limpo que aguente overlay escuro.

---

## Payload Templated.io (exemplo)

```json
{
  "template": "{{template_id_safe_force}}",
  "format": "jpg",
  "async": false,
  "layers": {
    "imagem_fundo": {
      "image_url": "{{imagem_url_cloudinary}}",
      "object_fit": "cover",
      "object_position": "center"
    },
    "logo": {
      "image_url": "{{logo_url_cloudinary}}"
    },
    "titulo": {
      "text": "{{titulo}}"
    },
    "subtitulo": {
      "text": "{{subtitulo}}"
    },
    "cta": {
      "text": "{{cta}}"
    },
    "handle": {
      "text": "@safeforce.pt"
    }
  }
}
```

---

## Passos para criar o template no editor Templated.io

1. Criar novo template em templated.io → "New Template"
2. Definir canvas: 1080 × 1350 px
3. Adicionar layers pela ordem: `imagem_fundo` (base) → `overlay` → `logo` → `titulo` → `subtitulo` → `cta` → `handle`
4. **Nomear cada layer exatamente como listado acima** (é o contrato do payload)
5. Anotar o `template_id` gerado — vai para a coluna `template_id` da Sheet
6. Testar com render manual via API antes de ligar ao n8n
