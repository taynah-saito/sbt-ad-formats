# +SBT — Pause Ad Formats (Web POC)

Proof of Concept de formatos de **pause ads** para o streaming +SBT, construída com **Video.js 8** e **Google Publisher Tag (GPT)** integrado ao **Google Ad Manager (GAM)** via **Native Styles**.

> Esta POC é uma referência técnica para a equipe de engenharia do cliente implementar os formatos em produção. Não é código pronto para produção: valores de Ad Unit, targeting, creatives e Native Styles devem ser configurados no GAM.

---

## Escopo e objetivo

Demonstrar, em um único arquivo HTML, os formatos de pause ad planejados para o ambiente **Web** do +SBT, com:

- Playback de VOD MP4 via Video.js 8.
- Request e renderização de anúncios via GPT/GAM (Native Styles + SafeFrame).
- Ad Unit compartilhado entre todos os formatos, diferenciados por targeting.
- Tratamento de **no-fill** que não expõe um layout vazio ao usuário.
- Console de eventos em tempo real com o ciclo de vida completo do GPT.
- Placeholders de key-values dinâmicos para integração futura com metadados do +SBT.

### Status atual

| Item | Status |
|---|---|
| Pause Ads (Overlay Central, Overlay Lateral, Split-Screen, Background) | Funcional |
| L-Bar | Em desenvolvimento (placeholder de UI, sem implementação) |
| Double Box | Em desenvolvimento (placeholder de UI, sem implementação) |
| Live HLS / SCTE-35 | Futuro |
| CTV | Futuro (a POC atual é Web-only) |

---

## Formatos disponíveis

| Formato | Descrição | Slot GPT | Targeting (`pause_ad_type`) |
|---|---|---|---|
| **Overlay Central** | Banner centralizado sobre o vídeo pausado | `ad-slot-pause` | `overlay` |
| **Overlay Lateral** | Creative ancorado à direita com fundo transparente | `ad-slot-pause-overlay-side` | `jequiti` |
| **Split-Screen** | Vídeo reduzido à esquerda + creative à direita | `ad-slot-pause-split` | `jequiti` |
| **Background** | Player reduzido ao canto + creative cobrindo o fundo | `ad-slot-pip` | `background` |

> L-Bar e Double Box aparecem no carrossel como formatos **em desenvolvimento**, sem código funcional por trás.

---

## Stack técnico

- **Video.js 8** — player de vídeo (VOD MP4)
- **Google Publisher Tag (GPT)** — request e renderização de ads
- **Google Ad Manager (GAM)** — ad server com Native Styles
- **SafeFrame** — renderização isolada dos creatives
- **DM Sans** — tipografia (Google Fonts)

Todas as bibliotecas são carregadas via CDN; não há build step nem dependências para instalar.

---

## Estrutura do repositório

```
.
├── plexus-ad-formats.html   # POC completa (single-file)
├── plexus-logo.png          # Logo utilizada no header
├── README.md
└── .gitignore
```

---

## Como executar

1. Faça clone do repositório.
2. Abra `plexus-ad-formats.html` em qualquer navegador moderno (Chrome, Firefox, Edge, Safari).
3. A POC inicializa com o formato **Overlay Central** selecionado.
4. Escolha um formato no carrossel, selecione a fonte de vídeo e clique em **Iniciar Ambiente de Teste**.
5. Dê **play** e em seguida **pause** o vídeo para disparar o pause ad.
6. O console de eventos (canto inferior) exibe o ciclo de vida do request GPT em tempo real.

Não há servidor necessário: a POC roda abrindo o arquivo diretamente no navegador.
Para testar com fontes remotas (VOD presets), é recomendado servir o arquivo via HTTP local para evitar restrições de CORS/mixed-content em alguns navegadores:

```bash
python3 -m http.server 8000
# acesse http://localhost:8000/plexus-ad-formats.html
```

---

## Configuração do GAM

### Ad Unit

Todos os pause ads utilizam o **mesmo Ad Unit**:

```
/23163430111/pause_ad
```

Cada formato define seu próprio slot GPT (com ID distinto) sobre o mesmo Ad Unit, diferenciando-se exclusivamente por targeting.

### Targeting

Cada slot envia, no momento do request:

| Key | Valor | Descrição |
|---|---|---|
| `env` | `web` | Ambiente de execução (Web-only nesta POC) |
| `pause_ad_type` | `overlay`, `jequiti`, `background` | Tipo de pause ad |

> Os valores `jequiti` (Overlay Lateral e Split-Screen) e `overlay` (Overlay Central) são definidos no código. As key-values correspondentes devem ser criadas no GAM e trafegar line items com targeting baseado nelas.

### Key-values dinâmicos (produção)

O código contém placeholders comentados em `requestGAMAd()`, prontos para preencher dinamicamente em produção com metadados do +SBT:

```js
// slot.setTargeting('program', '');
// slot.setTargeting('channel', '');
// slot.setTargeting('genre', '');
// slot.setTargeting('content_type', '');
// slot.setTargeting('device', '');
```

Para ativar em produção:

1. Descomentar as linhas em `requestGAMAd()`.
2. Preencher os valores dinamicamente a partir do CMS/metadata do +SBT (programa, canal, gênero, tipo de conteúdo, dispositivo).
3. No GAM: criar as key-values em **Inventory > Key-values** e trafegar line items com targeting baseado nelas.
4. Validar a consistência entre os valores enviados pelo player e os valores cadastrados no GAM.

---

## Native Style: click e destination URL

A clickabilidade do creative é configurada **no GAM Native Style**, não no código hospedeiro. O host apenas reserva o espaço e exibe o que o GAM renderiza dentro do SafeFrame.

Para o creative compartilhado entre Overlay Lateral e Split-Screen, o Native Style deve:

- Conter um campo de URL `DEST_URL` no Format.
- Envolver a imagem em um anchor usando a click macro do GAM:

```html
<div class="ctv-image-wrapper">
    <a href="%%CLICK_URL_UNESC%%%%DEST_URL%%" target="_blank">
        <img src="[%ImagePlaceHolder%]" alt="Ad" />
    </a>
</div>
```

- Confirmar que o line item/creative está configurado para o comportamento de clique desejado.

---

## Tratamento de No-Fill

O layout do pause ad **só é aplicado após o GAM confirmar que o ad veio** (`slotRenderEnded` com `isEmpty: false`). Esta abordagem **request-first** evita o flash visual de um layout vazio sendo exibido e revertido.

Se o GAM retornar vazio (no-fill):

- O layout **não** é aplicado.
- O usuário continua vendo o vídeo normalmente, sem alteração visual.
- O evento é logado no console como `No Ad Returned (No Fill)`.

Guard adicional: se o usuário retomar o vídeo antes da resposta do GAM chegar, o ad **não** é exibido após a resposta.

---

## Ciclo de vida do request

```
Usuário pausa o vídeo
  → requestGAMAd() dispara request ao GAM (clear + refresh do slot)
  → GPT envia targeting (env, pause_ad_type, key-values dinâmicos)
  → GAM responde via slotRenderEnded:
     → Com ad (isEmpty: false): applyAdLayout() aplica o layout + fadeIn animation
     → Vazio (no-fill): nada acontece, usuário continua no vídeo
  → Usuário retoma (play)
  → revertAdLayout() reverte o layout + clear do slot
```

A animação `fadeIn 0.25s ease` (com leve scale) é aplicada sobre o overlay para suavizar a entrada do ad.

---

## Eventos logados

A POC exibe um console de eventos em tempo real com o ciclo de vida completo do GPT e do playback:

```
Ad Requested: PAUSE AD OVERLAY CENTRAL
GAM Response Received: PAUSE AD OVERLAY CENTRAL | +18ms since request
Ad Rendered: PAUSE AD OVERLAY CENTRAL | Creative ID: 138569324388 | Line Item ID: 7388152451 | +94ms since request
Ad Fully Loaded: PAUSE AD OVERLAY CENTRAL | +342ms since request
```

```
Playback Started
Playback Resumed
Playback Seek
Video Paused — Triggering PAUSE AD OVERLAY CENTRAL
```

Esses logs são úteis para validar latência, diagnóstico de no-fill e auditoria do ciclo de vida do ad durante a implementação.

---

## Fullscreen

O fullscreen é aplicado no container do **stage** (não apenas no `<video>`), garantindo que os overlays de pause ad apareçam corretamente em tela cheia.

---

## Acessibilidade

Todos os overlays de anúncio incluem `role="dialog"` e `aria-label="Advertisement"`.

---

## Limitações conhecidas

- **Web-only:** a POC não cobre CTV.
- **VOD MP4 apenas:** HLS e live não estão implementados nesta versão.
- **L-Bar e Double Box:** presentes apenas como placeholders de UI, sem implementação funcional.
- **Single-file:** toda a POC vive em um único HTML para facilidade de entrega; em produção, separar CSS/JS é recomendado.
- **Sem telemetria de produção:** os logs são locais e voltados para validação, não para analytics.

---

## Próximos passos

- **Double Box com video ad (VAST)** — implementação via IMA SDK + SGAI.
- **L-Bar** — formato em L ao redor do vídeo.
- **Live HLS** — suporte a streaming ao vivo com SCTE-35.
- **CTV** — extensão para ambientes Connected TV.

---

## Segurança e configuração

- Não há credenciais nem secrets no repositório.
- O Ad Unit e os key-values são os únicos identificadores GAM expostos, e são públicos por natureza (visíveis no código do player em qualquer site que use GPT).
- Em produção, validar CSP, mixed-content e cookies de publicidade conforme a jurisdição e o consentimento do usuário.

---

## Arquivos

| Arquivo | Descrição |
|---|---|
| `plexus-ad-formats.html` | POC completa (single-file) |
| `plexus-logo.png` | Logo utilizada no header |
| `README.md` | Este documento |
| `.gitignore` | Ignora arquivos de SO, logs e `node_modules/` |
