# retrobooth (Fotomática)

Cabine de fotos para o navegador. Ela simula as tiras de photo booth de máquina (P&B ou sépia, 4 poses) e a foto única granulada com flash de filme, em pé (4:5) ou deitada (5:4). A câmera do celular ou do computador abre ao vivo, e o filtro roda em tempo real via WebGL.

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
| `state` | Estado único: `mode` (`cabine`/`retrato`/`horizontal`), `preset`, `p` (parâmetros atuais), `shots` (fotos capturadas), `live`, `facing`, `running`, `cancel`, `screen` (`camera`/`print`) |
| `AR` / `MODES` | Proporção do visor por modo (3:4, 4:5, 5:4) e textos por modo (botão, alt, título da revelação) |
| `VS` / `FS` | Shaders WebGL. O fragment shader é o filtro inteiro |
| `class Film` | Wrapper WebGL. `render(src, sw, sh, w, h, p, seed)` desenha qualquer imagem/vídeo com o filtro em um canvas de tamanho `w×h`, fazendo crop tipo "cover" |
| `fitVF()` | Só no celular (< 880px), na tela Câmera: mede tudo o que não é o visor e define a largura do `.vf` para a cabine caber em `innerHeight`, botão vermelho incluído. Reserva 24px extras em tela de toque para a barra flutuante do Safari. O painel de filtro conta só pela linha do resumo. Mínimo de 220px de altura |
| `showScreen()` / `newPhoto()` | Troca entre as telas Câmera e Revelação via `body[data-screen]`. Ao entrar na Revelação no celular, faz `history.pushState`, para o voltar do navegador levar de volta à Câmera. "Nova foto" limpa as fotos e volta |
| `vfNote()` | Mensagem curta sobre o visor (usa o `#hint`), para o progresso da galeria ("Foto 2 de 4"), já que o status fica na outra tela no celular |
| `live` / `off` | Duas instâncias de `Film`: `live` para o visor (`#preview`) e `off` para a revelação final e as miniaturas dos presets, em canvas fora da tela |
| `makeSample()` | Gera em canvas uma imagem de exemplo (cortina + silhueta) para o app não abrir vazio antes da câmera |
| `loop()` | `requestAnimationFrame`. Redesenha o visor a ~30 fps quando a câmera está ao vivo, ou só quando algo muda (`dirty`) |
| `startCamera()` / `camFailed()` | `getUserMedia` (ideal 1280px, `facingMode`). Se falhar, mostra a mensagem do `.gate` e cai para o input de arquivo |
| `runSequence()` | Contagem 3-2-1, flash, som de obturador (Web Audio) e `snapshot()`. 4 poses no modo cabine, 1 nos outros. O botão vira "Parar" durante a sequência. No fim, se há pelo menos uma foto, vai para a tela Revelação |
| `addFiles()` / `loadFile()` | Fallback sem câmera ao vivo: o botão vermelho abre `#fileCam` (`capture="user"`) e "Galeria" abre `#fileGal` (múltiplas fotos). As imagens são reduzidas para no máximo 1600px. Esconde o aviso `.gate`. Vai para a Revelação quando completa a tira (4) ou a foto única |
| `develop()` | Monta a revelação em `outC` e mostra em `#outImg` como JPEG. Cabine: 4 quadros de 600×800 com borda de 20px e espaço de 16px, resultando em 640×3288. Retrato: 1200×1500. Horizontal: 1500×1200. Roda com debounce de 120ms (`scheduleOut`) a cada mudança de slider |
| Salvar | Nome do arquivo: `fotomatica-{modo}-{AAAAMMDD-HHMM}.jpg`. Em tela de toque (`pointer:coarse`) com `navigator.canShare({files})`, abre o menu de compartilhar, que no iOS tem "Salvar imagem". Fora disso, ou se o share falhar, usa `<a download>`. O JPEG (qualidade .94) é gerado pelo `develop()` e guardado em `outBlob`. O clique usa esse blob sem esperar nada, porque o iOS recusa o `share()` se vier depois de um passo assíncrono. Só gera de novo se houver revelação pendente (`outPending`) |

Ponto importante: as fotos guardam a imagem original (`state.shots[].src`), não a filtrada. Por isso mexer nos sliders depois da captura muda a revelação. Não quebre isso.

## O filtro (fragment shader), em ordem

1. Crop "cover" + `zoom` (enquadre). Não há espelhamento
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

Ao trocar de modo sem fotos capturadas: Retrato e Horizontal selecionam `grao` (se estava em `prata`) e Cabine volta para `prata` (se estava em `grao`).

O slider de grão mostra ISO (100 a 6400) e o de exposição mostra EV.

## Design

- Conceito: à esquerda fica a "fachada" escura da cabine (visor, formato, botão vermelho, filtro). À direita, a tira revelada presa com fita crepe sobre papel kraft verde-oliva.
- Ordem da fachada: letreiro, visor, formato (Cabine / Retrato / Horizontal), controles (Galeria, botão vermelho, Trocar), painel de filtro. O filtro fica por último, para que abrir o painel não empurre o botão vermelho para fora da tela.
- Painel de filtro (`#filters`): uma linha de resumo (miniatura + "Filtro PRATA") que abre os 3 presets e, dentro deles, "Ajustes finos" (`#adj`). Fica aberto no desktop e recolhido no celular (definido no carregamento).
- Celular (< 880px) em duas telas, via `body[data-screen]`: **Câmera** (só a fachada, cabendo em uma tela) e **Revelação** (título, tira, barra fixa com Salvar foto / Nova foto, status e o painel de filtro embaixo). O desktop ignora `data-screen` e mostra as duas colunas sempre.
- O aviso da câmera (`.gate`) é uma faixa compacta no pé do visor. Quando o visor tem menos de 250px de largura (container query), o texto some e fica só o botão.
- Tokens em `:root`: `--kraft #8b8d5a`, `--ink #1b1a12`, `--ink-2`, `--ink-3`, `--paper #f2ede1`, `--muted #a29d86`, `--shutter #d4452c` (botão de disparo), `--tape #ebe4cd`.
- Fontes: Big Shoulders Display (letreiro/títulos), Archivo (corpo), IBM Plex Mono (rótulos técnicos, contagem, valores dos sliders).
- O tema é único e claro de propósito (`color-scheme: light`). Não há modo escuro.
- Textos: em português, diretos, no presente. Os botões dizem a ação ("Salvar foto", "Nova foto", "Trocar"). O botão de trocar câmera tem ícone e `aria-label="Trocar câmera"`.

## Restrições e cuidados

- A câmera ao vivo exige HTTPS (ou localhost). No GitHub Pages funciona. Abrindo o arquivo pelo `file://`, depende do navegador.
- `<video>` precisa de `playsinline muted autoplay` para o iOS não abrir em tela cheia.
- Nada é espelhado, nem o visor nem as fotos (decisão da Luíza, nov/2026): o que aparece no visor é o que sai na foto, como numa cabine de verdade. Não recoloque o espelhamento sem pedir.
- O Horizontal (5:4) com o celular em pé recorta o meio da imagem vertical da câmera: usa a largura inteira e corta em cima e embaixo. Não há layout próprio para o celular deitado.
- O WebGL usa `preserveDrawingBuffer: true` no canvas fora da tela, porque o `drawImage` copia dele depois do render.
- Sem service worker: o app instala na tela inicial (manifest + ícones), mas não funciona offline.
- O CSS tem `[hidden]{display:none!important}`. Sem essa regra, elementos com `display:grid` (`.count`, `.gate`) aparecem mesmo com o atributo `hidden`. Não remova.
- O `<head>` é limpo (sem o wrapper do artefato) e o `<html>` tem `lang="pt-BR"`. O código da capability `downloads` do artefato foi removido em nov/2026, então o arquivo não roda mais igual dentro do artefato do Claude.
- Acessibilidade: todos os controles são `<button>`/`<input>` reais, com `:focus-visible` em vermelho e `aria-pressed` nos toggles. O flash respeita `prefers-reduced-motion`.

## Pendências conhecidas

- O salvar pelo menu de compartilhar (nov/2026) foi testado só com simulação no navegador. Falta confirmar num iPhone e num Android de verdade.
- As duas telas no celular foram testadas em simulação (375×812 e 375×560). Falta confirmar no iPhone que a barra flutuante do Safari não cobre o botão vermelho (os 24px reservados em `fitVF()` são uma estimativa).

## Ideias que surgiram

- Calibrar o preset Granulado com uma foto real da Luíza na mesma luz em que ela vai usar a cabine.
- Carimbo de data opcional no rodapé da tira, como nas máquinas antigas.
- Service worker simples para uso offline.
