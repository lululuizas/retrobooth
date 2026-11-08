# retrobooth (Fotomática)

Cabine de fotos para o navegador. Ela simula as tiras de photo booth de máquina (P&B ou sépia, 4 poses) e o retrato granulado com flash de filme. A câmera do celular ou do computador abre ao vivo, e o filtro roda em tempo real via WebGL.

- Repositório: `lululuizas/retrobooth` (branch `main`)
- Site publicado (GitHub Pages, raiz da `main`): https://lululuizas.github.io/retrobooth/
- Dona: Luíza (designer de produto, com base em front-end). A interface e os textos são em português do Brasil.
- Origem: criado em uma conversa com o Claude em nov/2026. Existe também uma versão como artefato do Claude, chamada "Fotomática". Lá a câmera ao vivo fica bloqueada pelo sandbox. O site no GitHub Pages é a versão principal.

## Referências visuais (o que o filtro tenta imitar)

1. Tira de cabine clássica: 4 quadros verticais, fundo de cortina pregueada, P&B com leve tom quente, borda preta fina, fotografada sobre papel kraft verde-oliva.
2. Tiras sépia presas com fita crepe na parede: tom marrom mais forte, pretos lavados.
3. Retrato único: luz de flash dura no rosto, grão de filme bem visível, vinheta forte, contraste alto.

## Estrutura

```
index.html             # o app inteiro: HTML + CSS + JS inline, sem build
manifest.webmanifest   # PWA: nome, cores, ícones, display standalone
icon-192.png           # ícone do app (any maskable)
icon-512.png
apple-touch-icon.png   # ícone para "Adicionar à Tela de Início" no iOS
```

Não há framework, bundler nem dependências. A única dependência externa são as fontes do Google Fonts. Para testar, basta abrir o `index.html`. Para fazer deploy, basta dar push na `main` (o Pages publica em 1 a 2 minutos).

## Arquitetura do `index.html`

Tudo fica dentro de uma IIFE no `<script>` do fim da página.

| Parte | O que faz |
|---|---|
| `PRESETS` | Os 3 filtros prontos (`prata`, `sepia`, `grao`), cada um com os mesmos 9 parâmetros |
| `SLIDERS` | Define os controles: chave, rótulo, min/max/step e o formatador do valor exibido |
| `state` | Estado único: `mode` (`cabine`/`retrato`), `preset`, `p` (parâmetros atuais), `shots` (fotos capturadas), `live`, `facing`, `mirror`, `running`, `cancel` |
| `VS` / `FS` | Shaders WebGL. O fragment shader é o filtro inteiro |
| `class Film` | Wrapper WebGL. `render(src, sw, sh, w, h, p, mirror, seed)` desenha qualquer imagem/vídeo com o filtro em um canvas de tamanho `w×h`, fazendo crop tipo "cover" |
| `live` / `off` | Duas instâncias de `Film`: `live` para o visor (`#preview`) e `off` para a revelação final e as miniaturas dos presets, em canvas fora da tela |
| `makeSample()` | Gera em canvas uma imagem de exemplo (cortina + silhueta) para o app não abrir vazio antes da câmera |
| `loop()` | `requestAnimationFrame`. Redesenha o visor a ~30 fps quando a câmera está ao vivo, ou só quando algo muda (`dirty`) |
| `startCamera()` / `camFailed()` | `getUserMedia` (ideal 1280px, `facingMode`). Se falhar, mostra a mensagem do `.gate` e cai para o input de arquivo |
| `runSequence()` | Contagem 3-2-1, flash, som de obturador (Web Audio) e `snapshot()`. 4 poses no modo cabine, 1 no retrato. O botão vira "Parar" durante a sequência |
| `addFiles()` / `loadFile()` | Fallback sem câmera ao vivo: o botão vermelho abre `#fileCam` (`capture="user"`) e "Galeria" abre `#fileGal` (múltiplas fotos). As imagens são reduzidas para no máximo 1600px |
| `develop()` | Monta a revelação em `outC` e mostra em `#outImg` como JPEG. Cabine: 4 quadros de 600×800 com borda de 20px e espaço de 16px, resultando em 640×3288. Retrato: 1200×1500. Roda com debounce de 120ms (`scheduleOut`) a cada mudança de slider |
| Salvar | Usa um `<a download>` com o nome `fotomatica-{modo}-{AAAAMMDD-HHMM}.jpg` |

Ponto importante: as fotos guardam a imagem original (`state.shots[].src`), não a filtrada. Por isso mexer nos sliders depois da captura muda a revelação. Não quebre isso.

## O filtro (fragment shader), em ordem

1. Crop "cover" + `zoom` (enquadre) + espelhamento opcional
2. Luminância com pesos `0.36, 0.5, 0.14` (puxa os vermelhos, deixando a pele mais clara, como em filme ortocromático)
3. `soft`: mistura com um desfoque de 8 amostras (halation suave)
4. `ev`: exposição em stops (`exp2`)
5. `flash`: hotspot central no rosto + bloom nas altas luzes
6. `con`: curva S logística (`k = 1 + con*2.2`)
7. `vig`: vinheta radial, mais pesada embaixo
8. `fade`: levanta os pretos
9. `grain`: value noise + hash, mais forte nos meios-tons e escalado pela resolução
10. `tone`: mistura P&B → sépia (sombras marrom-escuras, altas luzes creme)

## Presets

| Chave | Nome na UI | ev | con | grain | vig | flash | soft | tone | fade | zoom |
|---|---|---|---|---|---|---|---|---|---|---|
| `prata` | Prata (P&B de cabine) | 0.20 | 0.45 | 0.30 | 0.35 | 0.35 | 0.25 | 0.18 | 0.12 | 1.25 |
| `sepia` | Sépia (tira envelhecida) | 0.30 | 0.40 | 0.35 | 0.30 | 0.40 | 0.30 | 0.90 | 0.22 | 1.25 |
| `grao` | Granulado (retrato com flash) | 0.10 | 0.75 | 0.75 | 0.65 | 0.60 | 0.35 | 0.06 | 0.00 | 1.10 |

Ao trocar de modo sem fotos capturadas: Retrato seleciona `grao` (se estava em `prata`) e Cabine volta para `prata` (se estava em `grao`).

O slider de grão mostra ISO (100 a 6400) e o de exposição mostra EV.

## Design

- Conceito: à esquerda fica a "fachada" escura da cabine (visor, presets, botão vermelho, ajustes). À direita, a tira revelada presa com fita crepe sobre papel kraft verde-oliva. No celular (< 880px), tudo vira uma coluna.
- Tokens em `:root`: `--kraft #8b8d5a`, `--ink #1b1a12`, `--ink-2`, `--ink-3`, `--paper #f2ede1`, `--muted #a29d86`, `--shutter #d4452c` (botão de disparo), `--tape #ebe4cd`.
- Fontes: Big Shoulders Display (letreiro/títulos), Archivo (corpo), IBM Plex Mono (rótulos técnicos, contagem, valores dos sliders).
- O tema é único e claro de propósito (`color-scheme: light`). Não há modo escuro.
- Textos: em português, diretos, no presente. Os botões dizem a ação ("Salvar foto", "Refazer", "Trocar câmera").

## Restrições e cuidados

- A câmera ao vivo exige HTTPS (ou localhost). No GitHub Pages funciona. Abrindo o arquivo pelo `file://`, depende do navegador.
- `<video>` precisa de `playsinline muted autoplay` para o iOS não abrir em tela cheia.
- O espelhamento vale só para o visor e para as fotos da câmera frontal. Fotos vindas da galeria nunca são espelhadas.
- Ao trocar para a câmera traseira, o espelhamento desliga sozinho.
- O WebGL usa `preserveDrawingBuffer: true` no canvas fora da tela, porque o `drawImage` copia dele depois do render.
- Sem service worker: o app instala na tela inicial (manifest + ícones), mas não funciona offline.
- O CSS tem `[hidden]{display:none!important}`. Sem essa regra, elementos com `display:grid` (`.count`, `.gate`) aparecem mesmo com o atributo `hidden`. Não remova.
- O `<head>` é limpo (sem o wrapper do artefato) e o `<html>` tem `lang="pt-BR"`. O código da capability `downloads` do artefato foi removido em nov/2026, então o arquivo não roda mais igual dentro do artefato do Claude.
- Acessibilidade: todos os controles são `<button>`/`<input>` reais, com `:focus-visible` em vermelho e `aria-pressed` nos toggles. O flash respeita `prefers-reduced-motion`.

## Pendências conhecidas

- No iOS, o `<a download>` de um blob pode abrir a imagem em vez de baixar. A mensagem de status já orienta a tocar e segurar a foto. Uma melhoria seria usar `navigator.share({ files })` quando disponível.

## Ideias que surgiram

- Calibrar o preset Granulado com uma foto real da Luíza na mesma luz em que ela vai usar a cabine.
- Carimbo de data opcional no rodapé da tira, como nas máquinas antigas.
- Service worker simples para uso offline.
