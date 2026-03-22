# Documentação Técnica do Aseprite

## 1. Visão Geral do Projeto

**Aseprite** é um editor de sprites animados voltado para pixel art. Originalmente criado por David Capello, hoje é desenvolvido e mantido pela Igara Studio S.A. O programa é amplamente utilizado por artistas de jogos indie, desenvolvedores e artistas digitais para criar sprites, tilesets, ícones e animações no estilo pixel art.

### Principais casos de uso

- Criação e edição de sprites estáticos e animados
- Criação de tilesets e tilemaps para jogos
- Exportação de spritesheets para engines de jogos (Unity, Godot, GMS, etc.)
- Animações em GIF e WebP
- Automação via script Lua ou linha de comando (CLI)

---

## 2. Arquitetura do Código

O repositório está organizado nos seguintes módulos dentro de `src/`:

| Módulo | Caminho | Responsabilidade |
|---|---|---|
| `app` | `src/app/` | Lógica principal da aplicação, comandos, UI, scripting, CLI, exportação |
| `doc` | `src/doc/` | Modelo de dados do documento (sprite, layers, cels, palette, etc.) |
| `render` | `src/render/` | Renderização de frames, onion skin, dithering, quantização |
| `filters` | `src/filters/` | Filtros de imagem (brilho, contraste, matiz, convolução, etc.) |
| `ui` | `src/ui/` | Sistema de interface gráfica (widgets, eventos, layout) |
| `laf` | `laf/` | Biblioteca de baixo nível (janelas, eventos de OS, fontes, gfx) |
| `fixmath` | `src/fixmath/` | Aritmética em ponto fixo |
| `flic` | `src/flic/` | Leitor/escritor do formato FLIC (FLI/FLC) |
| `undo` | `src/undo/` | Histórico de undo/redo genérico |
| `observable` | `src/observable/` | Padrão observer (signals/slots) |
| `dio` | `src/dio/` | Detecção de formato de arquivo |
| `psd` | `src/psd/` | Leitor de arquivos PSD (Photoshop) |
| `tga` | `src/tga/` | Leitor/escritor TGA |
| `net` | `src/net/` | Acesso à rede (verificação de updates) |

### Fluxo geral

```
Entrada do usuário (mouse/teclado)
       ↓
   app/ui (widgets, editor)
       ↓
   app/commands (operações de alto nível)
       ↓
   app/cmd (objetos Cmd individuais, undo/redo)
       ↓
   doc (modelo do documento)
       ↓
   render (composição de imagem para exibição)
```

---

## 3. Modelo de Documento (`src/doc`)

### 3.1 Sprite (`doc/sprite.h`, `doc/sprite.cpp`)

O `Sprite` é o objeto raiz de qualquer documento no Aseprite. Ele contém:

- **Dimensões** (largura × altura, máximo 65535 × 65535 px)
- **Modo de cor**: `RGBA`, `Indexed` (paleta até 256 cores) ou `Grayscale`
- **Pixel ratio** (wide pixels, ex: 2:1 para efeitos retrô)
- **Frames**: lista de durações em milissegundos por frame
- **Camadas** (`LayerGroup` raiz contendo toda a hierarquia)
- **Paletas** (`PalettesList`): uma por frame ou uma global
- **Tags** (`Tags`): anotações de loops de animação
- **Slices** (`Slices`): regiões nomeadas para exportação/atlas
- **Tilesets** (`Tilesets`): conjuntos de tiles para tilemaps
- **Máscara/Seleção** (`Mask`): seleção ativa
- **User data**: propriedades customizadas (texto, cor, UUID)

### 3.2 Layer (`doc/layer.h`)

Hierarquia de tipos de camadas:

| Tipo | Classe | Descrição |
|---|---|---|
| Imagem Regular | `LayerImage` | Camada com cels de imagem rasterizada |
| Grupo | `LayerGroup` | Agrupa outras camadas; suporta blend mode e opacidade |
| Tilemap | `LayerTilemap` | Camada de mapa de tiles referenciando um `Tileset` |
| Background | `LayerImage` com flag `Background` | Camada de fundo; sem transparência; não pode ser reordenada |

**Flags de camada** (`LayerFlags`):
- `Visible` — camada visível
- `Editable` — camada editável
- `LockMove` — posição travada
- `Background` — é a camada de fundo
- `Continuous` — cels copiados são vinculados (linked cels)
- `Collapsed` — grupo recolhido na timeline
- `Reference` — camada de referência (rotoscopia); não é exportada

### 3.3 Cel (`doc/cel.h`, `doc/cel_data.h`)

Um `Cel` é a instância de uma imagem em uma camada num frame específico:

- Referência à `CelData` (imagem + posição)
- Posição `(x, y)` dentro do sprite
- Opacidade individual (0–255)
- **Linked cels**: vários cels compartilham o mesmo `CelData` (alteração em um reflete em todos)
- Cel vazio: camada existe no frame mas sem imagem

### 3.4 Image (`doc/image.h`)

Representa um buffer de pixels com suporte a 4 formatos:
- `IMAGE_RGB` — 4 bytes por pixel (RGBA)
- `IMAGE_GRAYSCALE` — 2 bytes por pixel (Value + Alpha)
- `IMAGE_INDEXED` — 1 byte por pixel (índice na paleta)
- `IMAGE_BITMAP` — 1 bit por pixel (usado internamente)

### 3.5 Palette (`doc/palette.h`)

- Até 256 entradas de cor (RGBA 32-bit cada)
- Múltiplas paletas por sprite (uma por frame ou uma global)
- Frame de início configurável
- Cor transparente configurável (para modo Indexed)

### 3.6 Mask / Seleção (`doc/mask.h`)

- Bitmap de 1 bit por pixel indicando pixels selecionados
- Coordenadas de origem (x, y) relativas ao sprite
- Pode ser guardada/carregada de arquivo
- Operações: selecionar tudo, inverter, modificar (dilatar, contrair, borda, suavizar), por cor

### 3.7 Tag (`doc/tag.h`, `doc/tags.h`)

- Nome da animação (ex: "idle", "walk", "attack")
- Frame inicial e final
- Direção: `Forward`, `Reverse`, `PingPong`, `PingPongReverse`
- Cor de identificação
- User data associado

### 3.8 Slice (`doc/slice.h`, `doc/slices.h`)

- Nome + `SliceKey` por frame (keyframes)
- `SliceKey` contém: `bounds` (retângulo), `center` (nine-patch), `pivot` (ponto de origem)
- Útil para nine-patch, atlas de UI e marcação de regiões de exportação

### 3.9 Tileset e Tilemap (`doc/tileset.h`, `doc/layer_tilemap.h`)

- `Tileset`: coleção de imagens de tiles com grade (`Grid`) configurável
- Tile índice 0 é sempre o tile vazio
- `LayerTilemap`: armazena indices de tiles em vez de imagens rasterizadas diretas
- Permite edição manual ou automática de tiles (hash table para detecção de duplicatas)

### 3.10 Brush (`doc/brush.h`)

- Tipos: `BrushType::kCircle`, `kSquare`, `kLine`, `kImage`
- Para brushes do tipo imagem: imagem customizada + máscara
- Padrão de brush (`BrushPattern`): `kNone`, `kAlignedToSource`, `kPaintBrush`
- Slots de brushes (`AppBrushes`): slots numerados persistidos entre sessões

### 3.11 User Data (`doc/user_data.h`)

- Texto livre (string)
- Cor associada
- UUID
- Propriedades customizadas chave-valor (texto, inteiro, booleano, ponto, retângulo, etc.)
- Disponível em: Sprite, Layer, Cel, Tag, Slice, Tileset, Tile

---

## 4. Sistema de Renderização (`src/render`)

### 4.1 Composição de frames (`render/render.h`, `render/render.cpp`)

O `Renderer` compõe o frame final a partir de todas as camadas visíveis, de baixo para cima:

1. Para cada camada visível (respeitando grupos e visibilidade)
2. Obtém o cel do frame atual
3. Renderiza a imagem do cel na posição correta
4. Aplica o blend mode e opacidade da camada
5. Composita no buffer de destino

Suporte a:
- Camadas de referência (renderizadas separadamente, não exportadas)
- Background checker (xadrez indicando transparência)
- Zoom e projeção
- Extras (seleção, guias, etc.)

### 4.2 Blend Modes (`doc/blend_mode.h`)

Modos de mistura disponíveis:

| Modo | Enum | Descrição |
|---|---|---|
| Normal | `NORMAL` | Alpha compositing padrão |
| Multiplicar | `MULTIPLY` | Escurece multiplicando cores |
| Tela | `SCREEN` | Clareia (inverso do Multiply) |
| Sobrepor | `OVERLAY` | Combina Multiply e Screen |
| Escurecer | `DARKEN` | Mantém o pixel mais escuro |
| Clarear | `LIGHTEN` | Mantém o pixel mais claro |
| Esquivar cor | `COLOR_DODGE` | Clareia dividindo |
| Queimar cor | `COLOR_BURN` | Escurece dividindo |
| Luz dura | `HARD_LIGHT` | Overlay com camadas invertidas |
| Luz suave | `SOFT_LIGHT` | Versão suave do Hard Light |
| Diferença | `DIFFERENCE` | Subtração absoluta |
| Exclusão | `EXCLUSION` | Similar à diferença, mais suave |
| Matiz HSL | `HSL_HUE` | Usa matiz da camada superior |
| Saturação HSL | `HSL_SATURATION` | Usa saturação da camada superior |
| Cor HSL | `HSL_COLOR` | Usa H+S da camada superior |
| Luminosidade HSL | `HSL_LUMINOSITY` | Usa luminosidade da camada superior |
| Adição | `ADDITION` | Soma os canais |
| Subtrair | `SUBTRACT` | Subtrai os canais |
| Dividir | `DIVIDE` | Divide os canais |

Modos internos especiais: `SRC`, `MERGE`, `NEG_BW`, `RED_TINT`, `BLUE_TINT`, `DST_OVER` (usados pelo onion skin e outros efeitos internos).

### 4.3 Onion Skinning (`render/onionskin_options.h`)

Renderiza frames anteriores e/ou posteriores semitransparentes sobre o frame atual:

- **Tipo** (`OnionskinType`):
  - `MERGE`: frames fantasmas em tons normais com opacidade reduzida
  - `RED_BLUE_TINT`: frames anteriores em vermelho, posteriores em azul
- **Posição** (`OnionskinPosition`): `BEHIND` (atrás do frame atual) ou `INFRONT`
- **Número de frames**: `prevFrames` e `nextFrames` configuráveis
- **Opacidade**: `opacityBase` e `opacityStep` (cada frame adicional fica mais transparente)
- Pode ser limitado a uma tag ou a uma camada específica

### 4.4 Dithering (`render/dithering_algorithm.h`)

Algoritmos de dithering para conversão de cor (especialmente RGBA→Indexed):

| Algoritmo | Enum | Descrição |
|---|---|---|
| Nenhum | `None` | Sem dithering; usa o índice mais próximo |
| Ordenado | `Ordered` | Matriz de Bayer; padrão regular |
| Antigo | `Old` | Algoritmo legado do Aseprite |
| Difusão de erro | `ErrorDiffusion` | Floyd-Steinberg; distribui erro para pixels vizinhos |

Matrizes de dithering são carregáveis via extensão (arquivos `.dither`).

### 4.5 Quantização de Cores (`render/quantization.h`)

Reduz o número de cores para caber numa paleta Indexed:
- Algoritmo **Median Cut** (`render/median_cut.h`): divide recursivamente o espaço de cores
- Mapeamento via **Octree** (`doc/octree_map.h`): estrutura de 8 níveis para busca do índice mais próximo
- Alternativa: `RgbMapRGB5A3` — mapa hash de 5 bits por canal

### 4.6 Gradientes (`render/gradient.h`, `render/gradient.cpp`)

- Gradiente linear e radial
- Interpolação de cor entre foreground e background
- Dithering aplicável ao gradiente

### 4.7 Zoom e Projeção (`render/zoom.h`, `render/projection.h`)

- `Zoom`: fator racional (numerador/denominador), ex: 1:4 (afastar), 4:1 (aproximar)
- `Projection`: combina zoom e pixel ratio para calcular coordenadas de tela↔sprite

---

## 5. Ferramentas de Desenho (`src/app/tools`)

O sistema de ferramentas é composto por quatro componentes combinados:

| Componente | Interface | Função |
|---|---|---|
| **Controller** | `controller.h` | Como o usuário gera pontos (freehand, ponto a ponto, 2 pontos, 4 pontos) |
| **Intertwiner** | `intertwine.h` | Como os pontos se conectam (linha, retângulo, elipse, bézier, pixel-perfect) |
| **Point Shape** | `point_shape.h` | Forma aplicada em cada ponto (pixel, brush, flood fill, spray) |
| **Ink** | `ink.h` | Como a cor é aplicada (pintura, apagar, sombra, gradiente, etc.) |

### 5.1 Controllers

| ID | Descrição |
|---|---|
| `freehand` | Captura todos os pontos enquanto o botão está pressionado |
| `point_by_point` | Cliques individuais definem pontos |
| `one_point` | Um único ponto (ex: balde de tinta) |
| `two_points` | Dois pontos (linha, retângulo, elipse) |
| `four_points` | Quatro pontos (curva de Bézier) |
| `line_freehand` | Linha em modo freehand (traçado livre) |

### 5.2 Intertwiners

| ID | Descrição |
|---|---|
| `none` | Cada ponto é independente |
| `first_point` | Apenas o primeiro ponto |
| `as_lines` | Liga os pontos com segmentos de linha (algoritmo de Bresenham) |
| `as_rectangles` | Forma retângulo entre dois pontos |
| `as_ellipses` | Forma elipse entre dois pontos |
| `as_bezier` | Curva cúbica de Bézier por quatro pontos |
| `as_pixel_perfect` | Modo pixel perfeito: remove pixels diagonais duplos automaticamente |

### 5.3 Point Shapes

| ID | Descrição |
|---|---|
| `none` | Nenhuma forma (usado internamente) |
| `pixel` | Aplica um único pixel |
| `tile` | Aplica um tile inteiro (modo tilemap) |
| `brush` | Aplica o brush atual (circular, quadrado, linha ou imagem customizada) |
| `floodfill` | Balde de tinta (flood fill contíguo ou global) |
| `spray` | Aerógrafo: dispersa pixels aleatoriamente em raio configurável |

### 5.4 Ink Types (Tipos de Tinta)

| Ink | Enum | Descrição |
|---|---|---|
| Simples | `SIMPLE` | Substitui a cor do pixel diretamente |
| Alpha Compositing | `ALPHA_COMPOSITING` | Composita a cor com alpha sobre o pixel existente |
| Copiar Cor | `COPY_COLOR` | Copia cor sem considerar alpha |
| Travar Alpha | `LOCK_ALPHA` | Pinta apenas onde há pixels opacos; preserva alpha |
| Sombreamento | `SHADING` | Desloca cores dentro da paleta (modo Indexed) ou usa shades |

Inks especiais (definidos em `tool_box.cpp`):

| Ink ID | Função |
|---|---|
| `paint_fg` / `paint_bg` | Pinta com cor de frente/fundo |
| `eraser` | Apaga pixels (torna transparente ou pinta com background) |
| `replace_fg_with_bg` | Substitui cor fg por bg no pixel |
| `pick_fg` / `pick_bg` | Eyedropper para cor de frente/fundo |
| `gradient` | Aplica gradiente |
| `blur` | Borra os pixels na área do brush |
| `jumble` | Embaralha os pixels (efeito smear) |
| `zoom` | Altera zoom ao clicar/arrastar |
| `scroll` | Move o viewport |
| `move` | Move cels/layers |
| `slice` / `move_slice` | Cria e move slices |
| `selection` | Ferramentas de seleção |
| `text` | Inserção de texto |

### 5.5 Dynamics (`app/tools/dynamics.h`)

Suporte a sensibilidade de pressão e velocidade para tablets:
- Mapeia **pressão**, **velocidade** ou **ângulo** para: tamanho do brush, opacidade, gradiente
- Curvas de sensibilidade configuráveis
- Funciona via eventos do sistema operacional (Wacom, etc.)

### 5.6 Symmetry (`app/tools/symmetry.h`, `app/tools/symmetry.cpp`)

- Modo horizontal: espelha cada pincelada horizontalmente
- Modo vertical: espelha verticalmente
- Eixo configurável (posição do eixo de simetria)
- Vários eixos simultâneos possíveis

---

## 6. Comandos e Operações (`src/app/commands`)

Cada comando herda de `Command` e implementa `onExecute()`. Todos os comandos são registrados em `commands_list.h` com a macro `FOR_EACH_COMMAND`.

### 6.1 Arquivo

| Comando | Descrição |
|---|---|
| `NewFile` | Cria novo sprite (diálogo de configuração) |
| `OpenFile` | Abre arquivo de sprite ou imagem |
| `SaveFile` | Salva o arquivo atual |
| `SaveFileAs` | Salva com novo nome |
| `SaveFileCopyAs` | Salva uma cópia sem alterar o nome atual |
| `CloseFile` | Fecha o documento atual |
| `CloseAllFiles` | Fecha todos os documentos abertos |
| `ReopenClosedFile` | Reabre o último arquivo fechado |
| `ClearRecentFiles` | Limpa a lista de arquivos recentes |
| `ExportSpriteSheet` | Exporta todas as frames como spritesheet |
| `ImportSpriteSheet` | Importa um spritesheet e cria frames |
| `RepeatLastExport` | Repete a última exportação sem diálogo |
| `ExportTileset` | Exporta um tileset como imagem |
| `OpenInFolder` | Abre a pasta do arquivo atual |
| `OpenWithApp` | Abre o arquivo com aplicativo associado |
| `Screenshot` | Captura screenshot do aplicativo |
| `Exit` | Sai do programa |

### 6.2 Edição

| Comando | Descrição |
|---|---|
| `Undo` | Desfaz a última operação |
| `Redo` | Refaz a operação desfeita |
| `UndoHistory` | Abre janela com histórico de undo |
| `Copy` | Copia a seleção para a área de transferência |
| `CopyMerged` | Copia a seleção com todas as camadas mescladas |
| `Cut` | Recorta a seleção |
| `Paste` | Cola da área de transferência |
| `PasteText` | Cola texto como pixels |
| `Clear` | Limpa os pixels selecionados |
| `ClearCel` | Limpa o cel atual completamente |
| `Fill` | Preenche a seleção com a cor de frente |
| `Stroke` | Aplica contorno na seleção |
| `ReplaceColor` | Substitui uma cor por outra |

### 6.3 Visualização

| Comando | Descrição |
|---|---|
| `Zoom` | Aumenta/diminui zoom |
| `FitScreen` | Ajusta o sprite à janela |
| `FullscreenMode` | Alterna modo tela cheia |
| `FullscreenPreview` | Prévia em tela cheia |
| `TiledMode` | Alterna modo de desenho em tile (repetição) |
| `GridSettings` | Configura o grid (tamanho, cor, snap) |
| `SnapToGrid` | Ativa/desativa snap ao grid |
| `SymmetryMode` | Ativa/desativa modo de simetria |
| `ShowGrid` | Mostra/oculta o grid |
| `ShowPixelGrid` | Mostra/oculta o grid de pixels (em alto zoom) |
| `ShowExtras` | Mostra/oculta elementos extras (bordas, guias) |
| `ShowSlices` | Mostra/oculta slices |
| `ShowLayerEdges` | Mostra/oculta bordas dos cels |
| `ShowSelectionEdges` | Mostra/oculta contorno da seleção |
| `ShowOnionSkin` | Ativa/desativa onion skinning |
| `ShowBrushPreview` | Mostra/oculta prévia do brush no cursor |
| `ShowTileNumbers` | Mostra/oculta números dos tiles |
| `ShowAutoGuides` | Mostra/oculta guias automáticas |
| `TogglePreview` | Mostra/oculta janela de prévia |
| `Timeline` | Mostra/oculta a timeline |
| `ToggleWorkspaceLayout` | Alterna layout do workspace |
| `DuplicateView` | Duplica a aba do mesmo sprite |
| `AdvancedMode` | Oculta/mostra painéis para modo de desenho limpo |
| `Home` | Vai para a aba de boas-vindas |

### 6.4 Camadas

| Comando | Descrição |
|---|---|
| `NewLayer` | Cria nova camada (normal, grupo, tilemap, background) |
| `RemoveLayer` | Remove a camada selecionada |
| `DuplicateLayer` | Duplica a camada atual |
| `MergeDownLayer` | Mescla a camada com a de baixo |
| `FlattenLayers` | Achata todas as camadas em uma |
| `LayerOpacity` | Altera opacidade da camada |
| `LayerVisibility` | Alterna visibilidade |
| `LayerLock` | Trava/destrava edição da camada |
| `LayerProperties` | Abre diálogo de propriedades |
| `BackgroundFromLayer` | Converte camada transparente em background |
| `LayerFromBackground` | Converte background em camada transparente |
| `ConvertLayer` | Converte entre tipos de camada (Image↔Tilemap) |
| `OpenGroup` / `SoloLayer` | Expande grupo / exibe apenas essa camada |
| `GotoNextLayer` / `GotoPreviousLayer` | Navega entre camadas |
| `ToggleOtherLayersOpacity` | Reduz opacidade das outras camadas |

### 6.5 Frames e Animação

| Comando | Descrição |
|---|---|
| `NewFrame` | Insere novo frame após o atual |
| `RemoveFrame` | Remove o frame atual |
| `CopyCel` | Copia o cel atual |
| `MoveCel` | Move o cel para outro frame/layer |
| `LinkCels` | Vincula cels (shared data) |
| `UnlinkCel` | Desvincula o cel atual |
| `ReverseFrames` | Inverte a ordem dos frames no intervalo |
| `FrameProperties` | Duração e propriedades do frame |
| `NewFrameTag` | Cria nova tag de animação |
| `RemoveFrameTag` | Remove tag |
| `FrameTagProperties` | Edita tag (nome, frames, direção, cor) |
| `SetLoopSection` | Define seção de loop para reprodução |
| `GotoFirstFrame` / `GotoLastFrame` | Navega para início/fim |
| `GotoNextFrame` / `GotoPreviousFrame` | Navega um frame |
| `GotoFrame` | Vai para frame específico |
| `GotoFirstFrameInTag` / `GotoLastFrameInTag` | Navega dentro de uma tag |
| `PlayAnimation` | Inicia/para reprodução da animação |
| `PlayPreviewAnimation` | Reprodução na janela de prévia |
| `SetPlaybackSpeed` | Altera velocidade de reprodução |
| `TogglePlayOnce` | Alterna reprodução única vs. loop |
| `TogglePlayAll` | Alterna reprodução de todas as tags |
| `TogglePlaySubtags` | Alterna reprodução de subtags |
| `ToggleRewindOnStop` | Volta ao início ao parar |
| `ToggleTimelineThumbnails` | Miniaturas na timeline |
| `CelOpacity` | Altera opacidade do cel |
| `CelProperties` | Propriedades do cel |

### 6.6 Seleção

| Comando | Descrição |
|---|---|
| `MaskAll` | Seleciona tudo |
| `DeselectMask` | Deseleciona |
| `ReselectMask` | Restaura última seleção |
| `InvertMask` | Inverte a seleção |
| `MaskByColor` | Seleciona por cor (tolerância configurável) |
| `MaskContent` | Seleciona pixels não-transparentes do cel |
| `ModifySelection` | Dilata, contrai, suaviza ou cria borda da seleção |
| `LoadMask` | Carrega seleção de arquivo |
| `SaveMask` | Salva seleção em arquivo |
| `MoveMask` | Move apenas a seleção (não os pixels) |
| `SelectionAsGrid` | Usa a seleção como grid |
| `NewSpriteFromSelection` | Cria novo sprite a partir da seleção |

### 6.7 Sprite

| Comando | Descrição |
|---|---|
| `SpriteProperties` | Cor transparente, pixel ratio, color profile |
| `SpriteSize` | Redimensiona o sprite (com filtro) |
| `CanvasSize` | Altera o tamanho da tela sem redimensionar |
| `AutocropSprite` | Corta automaticamente bordas transparentes |
| `CropSprite` | Corta ao tamanho da seleção |
| `ChangePixelFormat` | Converte entre RGBA, Indexed, Grayscale |
| `ColorQuantization` | Gera paleta otimizada a partir das cores |
| `Rotate` | Rotaciona sprite/seleção/cel (90°, 180°, livre) |
| `Flip` | Espelha horizontal ou verticalmente |
| `DuplicateSprite` | Cria cópia do sprite em nova aba |

### 6.8 Paleta

| Comando | Descrição |
|---|---|
| `PaletteEditor` | Abre editor de paleta |
| `LoadPalette` | Carrega paleta de arquivo |
| `SavePalette` | Salva paleta em arquivo |
| `PaletteSize` | Altera número de entradas |
| `SetPalette` | Define paleta ativa |
| `SelectPaletteColors` | Seleciona cores da paleta usadas na seleção |
| `SetPaletteEntrySize` | Tamanho de exibição das entradas |
| `SortPalette` (via `MoveColors`) | Ordena as cores da paleta |
| `AddColor` | Adiciona a cor atual à paleta |
| `MoveColors` / `CopyColors` | Move/copia entradas na paleta |
| `SwapCheckerboardColors` | Troca cores do xadrez de fundo |
| `SwitchColors` | Troca foreground e background |

### 6.9 Filtros de Imagem

| Comando | Descrição |
|---|---|
| `BrightnessContrast` | Brilho e contraste |
| `HueSaturation` | Matiz, saturação e luminosidade |
| `ColorCurve` | Curva de cor RGB/canal |
| `InvertColor` | Inverte as cores |
| `Outline` | Adiciona contorno ao sprite/layer |
| `ConvolutionMatrix` | Aplica matriz de convolução customizada |
| `Despeckle` | Remove ruído (filtro mediana) |
| `Apply` | Aplica filtro pendente |

### 6.10 Brush e Ferramentas

| Comando | Descrição |
|---|---|
| `ChangeBrush` | Alterna brush salvo por slot |
| `NewBrush` | Cria brush a partir da seleção atual |
| `DiscardBrush` | Descarta o brush personalizado atual |
| `PixelPerfectMode` | Ativa/desativa modo pixel perfeito |
| `SetInkType` | Define tipo de tinta ativo |
| `SetSameInk` | Sincroniza ink entre ferramentas fg/bg |
| `ChangeColor` | Altera cor de frente/fundo |
| `Eyedropper` | Captura cor do sprite |

### 6.11 Tileset/Tilemap

| Comando | Descrição |
|---|---|
| `TilesetMode` | Modo de edição do tileset (manual, auto, stack) |
| `ToggleTilesMode` | Alterna entre editar tiles e pixels |
| `SelectTile` | Seleciona tile atual |
| `MoveTiles` / `CopyTiles` | Move/copia tiles no tileset |

### 6.12 Slices

| Comando | Descrição |
|---|---|
| `DuplicateSlice` | Duplica o slice selecionado |
| `RemoveSlice` | Remove slice |
| `SliceProperties` | Edita propriedades do slice |

### 6.13 Scripting

| Comando | Descrição |
|---|---|
| `RunScript` | Executa um arquivo Lua |
| `DeveloperConsole` | Console interativo Lua |
| `OpenScriptFolder` | Abre pasta de scripts no explorador |
| `Debugger` | Abre o debugger de scripts |

### 6.14 Outros

| Comando | Descrição |
|---|---|
| `About` | Informações sobre o Aseprite |
| `KeyboardShortcuts` | Editor de atalhos de teclado |
| `Options` | Configurações gerais |
| `Refresh` | Força redesenho da UI |
| `RunCommand` | Executa um comando por ID (usado por scripts) |
| `Scroll` / `ScrollCenter` | Move a viewport |
| `GotoNextTab` / `GotoPreviousTab` | Navega entre abas |
| `Launch` | Lança URL no navegador |
| `OpenBrowser` | Abre URL interna |
| `ShowMenu` | Exibe menu por ID |
| `CopyPath` | Copia o caminho do arquivo para o clipboard |

---

## 7. Sistema de Undo/Redo (`src/app/cmd`, `src/undo`)

### 7.1 Arquitetura

```
Transaction
  └── CmdTransaction  (um por operação do usuário)
        └── CmdSequence  (lista de Cmd)
              └── Cmd[0], Cmd[1], ...  (mudanças atômicas)
```

- `Transaction` (`app/transaction.h`): agrupa uma operação do usuário. Ao chamar `commit()`, o `CmdTransaction` é adicionado ao `DocUndo`.
- `CmdTransaction` (`app/cmd_transaction.h`): herda de `CmdSequence`; armazena o label da operação, posição do sprite antes/depois, e range da timeline.
- `CmdSequence` (`app/cmd_sequence.h`): lista de `Cmd` executáveis, desfazíveis e refeitos em ordem.
- `Cmd` (`app/cmd.h`): operação atômica abstrata com `onExecute()`, `onUndo()`, `onRedo()`.
- `DocUndo` (`app/doc_undo.h`): lista de `CmdTransaction`; implementa undo/redo não-linear (árvore de estados).

### 7.2 Undo Não-Linear

O Aseprite suporta um histórico de undo em forma de árvore. Ao desfazer e então realizar uma nova operação, o histórico não é truncado imediatamente — o estado anterior pode ser restaurado pelo UndoHistory, permitindo navegar entre ramos de edição.

### 7.3 Principais objetos Cmd (`src/app/cmd/`)

| Cmd | Função |
|---|---|
| `AddCel` / `RemoveCel` | Adiciona/remove cel |
| `AddFrame` / `RemoveFrame` | Adiciona/remove frame |
| `AddLayer` / `RemoveLayer` | Adiciona/remove camada |
| `AddPalette` / `RemovePalette` | Adiciona/remove paleta |
| `AddTag` / `RemoveTag` | Adiciona/remove tag |
| `AddSlice` / `RemoveSlice` | Adiciona/remove slice |
| `AddTile` / `AddTileset` / `RemoveTileset` | Operações de tileset |
| `ClearCel` | Limpa imagem do cel |
| `ClearImage` | Limpa buffer de imagem |
| `ClearMask` | Limpa a máscara |
| `CopyCel` / `MoveCel` | Copia/move cel |
| `CopyFrame` | Duplica frame |
| `CopyRect` / `CopyRegion` | Copia região de imagem |
| `CropCel` | Recorta bounds de um cel |
| `FlipImage` / `FlipMask` / `FlipMaskedCel` | Espelha imagem/máscara |
| `FlattenLayers` | Achata camadas |
| `LayerFromBackground` / `BackgroundFromLayer` | Converte tipo de camada |
| `MoveLayer` | Reordena camada na hierarquia |
| `PatchCel` | Aplica um patch (região editada) a um cel |
| `RemapColors` | Remapeia índices de cor (paleta Indexed) |
| `RemapTilemaps` / `RemapTileset` | Remapeia tiles após reorganização |
| `AssignColorProfile` / `ConvertColorProfile` | Perfil de cor |

---

## 8. Formatos de Arquivo (`src/app/file`)

| Formato | Arquivo | Frames | Layers | Alpha | Paleta | Observações |
|---|---|---|---|---|---|---|
| **ASE/ASEPRITE** | `ase_format.cpp` | ✅ | ✅ | ✅ | ✅ | Formato nativo; suporta tags, slices, tilesets, user data, color profile |
| **PNG** | `png_format.cpp` | ❌ (1) | ❌ | ✅ | ❌ | RGBA ou indexed; exportação de frames individuais |
| **GIF** | `gif_format.cpp` | ✅ | ❌ | Parcial | ✅ | Animado; até 256 cores por frame; transparência binária |
| **WebP** | `webp_format.cpp` | ✅ | ❌ | ✅ | ❌ | Animado; alta compressão; suporte lossy e lossless |
| **JPEG** | `jpeg_format.cpp` | ❌ | ❌ | ❌ | ❌ | Sem transparência; compressão lossy |
| **BMP** | `bmp_format.cpp` | ❌ | ❌ | ❌ | ❌ | RGB sem compressão |
| **PCX** | `pcx_format.cpp` | ❌ | ❌ | ❌ | ✅ | Formato legado |
| **TGA** | `tga_format.cpp` | ❌ | ❌ | ✅ | ❌ | RGBA; sem compressão ou RLE |
| **FLI/FLC** | `fli_format.cpp` | ✅ | ❌ | ❌ | ✅ | Formato FLIC de animação; suporte legado |
| **ICO** | `ico_format.cpp` | ❌ | ❌ | ✅ | ❌ | Ícones do Windows; múltiplos tamanhos |
| **SVG** | `svg_format.cpp` | ❌ | ❌ | ✅ | ❌ | Apenas exportação; cada pixel vira um `<rect>` |
| **QOI** | `qoi_format.cpp` | ❌ | ❌ | ✅ | ❌ | Quite OK Image; compressão rápida e lossless |
| **PSD** | `psd_format.cpp` | ❌ | ✅ | ✅ | ❌ | Importação de camadas do Photoshop |
| **CSS** | `css_format.cpp` | ❌ | ❌ | ✅ | ❌ | Apenas exportação; gera CSS com `box-shadow` |

### 8.1 Formatos de Paleta (`src/app/file/palette_file.h`)

- `.pal` (RIFF Palette, Jasc)
- `.gpl` (GIMP Palette)
- `.hex` (lista de cores hexadecimais)
- `.act` (Adobe Color Table)
- `.ase` (paleta embutida em sprite)
- `.png` / `.bmp` (imagem como paleta)

---

## 9. Interface do Usuário (`src/app/ui`)

### 9.1 Layout principal

```
┌─────────────────────────────────────────────────┐
│  Barra de Menu                                  │
├──────┬──────────────────────────────────────────┤
│      │  Barra de Contexto (opções da ferramenta) │
│ Tool │──────────────────────────────────────────┤
│  bar │  Canvas / Editor (DocView)               │
│      │                                          │
├──────┴──────────────────────────────────────────┤
│  Timeline (frames × layers)                     │
├──────────────────────────────────────────┬──────┤
│  Color Bar (paleta + foreground/background)│Status│
└──────────────────────────────────────────┴──────┘
```

### 9.2 Canvas/Editor (`src/app/ui/editor/`)

- `Editor`: widget central de edição
- Gerencia zoom, scroll, pixel grid
- Modos de estado: `EditorState` (desenho, seleção, transformação, etc.)
- `TransformHandles`: alças de transformação (mover, escalar, rotacionar seleção)
- `PixelsMovement`: lógica de mover pixels selecionados com prévia em tempo real
- Suporte a múltiplos editores abertos para o mesmo sprite (sincronizados)

### 9.3 Timeline (`src/app/ui/timeline/`)

- Grade 2D: eixo X = frames, eixo Y = camadas
- Exibe cels como miniaturas (toggleável)
- Drag-and-drop de cels e frames
- Seleção de range de cels para operações em lote
- Tags coloridas sobre os frames
- Controles de reprodução integrados
- Configuração de onion skin acessível pelo ícone

### 9.4 Color Bar (`src/app/ui/color_bar.h`)

- Exibe a paleta atual com swatches clicáveis
- Seletor de foreground (clique esquerdo) e background (clique direito)
- Botão de troca de cores (switch)
- Abre seletores de cor avançados:
  - `ColorSpectrum`: espectro H×S com slider de valor
  - `ColorWheel`: roda de cores HSV/HSL
  - `ColorTintShadeTone`: grade tint/shade/tone
  - `ColorSliders`: sliders RGB/HSV/HSL/Gray

### 9.5 Context Bar (`src/app/ui/context_bar.h`)

Muda dinamicamente dependendo da ferramenta ativa. Pode exibir:
- Tipo/tamanho/ângulo de brush
- Spray: raio e velocidade
- Selection: modo (replace, add, subtract, intersect)
- Ink type
- Opacidade e fluxo
- Modo pixel perfect, simetria, contiguous fill
- Opções de transformação

### 9.6 Brush Popup (`src/app/ui/brush_popup.h`)

- Grade de brushes salvos nos slots 1–10
- Preview em tempo real ao passar o mouse
- Permite salvar, carregar e descartar brushes

### 9.7 Dynamics Popup (`src/app/ui/dynamics_popup.h`)

- Configuração de sensibilidade de pressão, velocidade e ângulo
- Curvas de resposta para tamanho, opacidade e gradiente

### 9.8 Diálogos de propriedades

- **Layer Properties**: nome, opacidade, blend mode, user data
- **Cel Properties**: posição, opacidade, z-index, user data
- **Frame Properties**: duração em ms
- **Sprite Properties**: cor transparente, pixel ratio, color profile, grid
- **Frame Tag Properties**: nome, frames, direção, cor, user data
- **Slice Properties**: nome, bounds, nine-patch center, pivot, user data

### 9.9 Export File Window (`src/app/ui/export_file_window.h`)

- Configuração de spritesheet: tipo (horizontal, vertical, packed, etc.)
- Opções de padding entre frames
- Exportação de JSON de dados (Aseprite, JSONArray, JSONHash)
- Seleção de layers e frames para exportar
- Redimensionamento de saída

### 9.10 Data Recovery View (`src/app/ui/data_recovery_view.h`)

- Lista sessões de backup recuperáveis
- Descrição com nome do arquivo e timestamp
- Botões: recuperar, deletar sessão

---

## 10. Scripting com Lua (`src/app/script`)

### 10.1 Engine (`app/script/engine.h`)

- Usa **Lua 5.4** embarcado
- Motor iniciado em `Engine::Engine()`; executa scripts via `eval()` ou arquivo
- API exposta ao Lua via `lua_register` / `luaL_newlib`
- Segurança: `Security` (`app/script/security.h`) exige permissão do usuário para acesso a arquivo e rede

### 10.2 Objetos disponíveis

| Objeto Lua | Arquivo | Descrição |
|---|---|---|
| `app` | `app_object.cpp` | Objeto global; acesso a sprite ativa, foreground/bg color, comandos |
| `app.fs` | `app_fs_object.cpp` | Sistema de arquivos (listar, criar, ler diretórios) |
| `app.os` | `app_os_object.cpp` | Info do SO (nome, versão) |
| `app.command` | `app_command_object.cpp` | Execução de comandos por ID |
| `app.theme` | `app_theme_object.cpp` | Acesso ao tema de UI |
| `Sprite` | `sprite_class.cpp` | Sprite ativo/aberto; layers, frames, tags, slices, tilesets, palettes |
| `Layer` | `layer_class.cpp` | Camada: nome, visibilidade, blend mode, cels, parent |
| `Cel` | `cel_class.cpp` | Cel: imagem, posição, opacidade, frame |
| `Frame` | `frame_class.cpp` | Frame: número, duração |
| `Image` | `image_class.cpp` | Acesso pixel a pixel, resize, draw, clone |
| `ImageSpec` | `image_spec_class.cpp` | Especificação (tamanho, modo de cor, color space) |
| `Palette` | `palette_class.cpp` | Paleta: número de entradas, getColor, setColor, resize |
| `Color` | `color_class.cpp` | Cor: rgba, hsv, hsl, gray, index |
| `ColorSpace` | `color_space_class.cpp` | Perfil de cor ICC |
| `Tag` | `tag_class.cpp` | Tag: nome, fromFrame, toFrame, direction, color |
| `Slice` | `slice_class.cpp` | Slice: nome, bounds, center, pivot |
| `Tileset` | `tileset_class.cpp` | Tileset: tiles, grid, addTile, getTile |
| `Selection` | `selection_class.cpp` | Seleção ativa: bounds, contains, select, deselect |
| `Range` | `range_class.cpp` | Range da timeline: layers, frames, cels selecionados |
| `Dialog` | `dialog_class.cpp` | Criação de diálogos de UI customizados com widgets |
| `Tool` | `tool_class.cpp` | Ferramenta ativa |
| `Grid` | `grid_class.cpp` | Grade: tamanho, origem |
| `Point`, `Size`, `Rectangle` | `*_class.cpp` | Estruturas geométricas |
| `Timer` | `timer_class.cpp` | Timer de intervalo para scripts assíncronos |
| `WebSocket` | `websocket_class.cpp` | Conexão WebSocket para integração externa |
| `Plugin` | `plugin_class.cpp` | Registro de plugins e extensão do menu |
| `JSON` | `json_class.cpp` | Parse e stringify de JSON |
| `Version` | `version_class.cpp` | Versão do Aseprite |

### 10.3 Sistema de Eventos (`app/script/events_class.cpp`)

Scripts podem se registrar em eventos:
- `app.events:on('sitechange', callback)` — quando o sprite/frame/layer ativo muda
- `app.events:on('beforecommand', callback)` — antes de um comando ser executado
- `app.events:on('aftercommand', callback)` — após execução de comando
- `sprite.events:on('change', callback)` — quando o sprite é modificado
- `sprite.events:on('filenamechange', callback)` — quando o nome do arquivo muda

### 10.4 Sistema de Plugins

- Scripts na pasta `%APPDATA%/Aseprite/scripts/` são carregados automaticamente
- Plugins podem ser instalados via extensões (pacotes `.aseprite-extension`)
- API `app.command.RunScript` executa script por ID
- Podem registrar itens de menu, atalhos e diálogos customizados

---

## 11. Interface de Linha de Comando (`src/app/cli`)

O Aseprite pode ser executado em modo headless para automação:

```bash
aseprite -b [opções] arquivo.ase
```

### 11.1 Principais flags

| Flag | Descrição |
|---|---|
| `-b` / `--batch` | Modo batch (sem UI) |
| `--save-as <arquivo>` | Salva com novo nome/formato |
| `--export-type <tipo>` | Tipo de saída (horizontal, vertical, packed, rows, columns) |
| `--sheet <arquivo>` | Exporta como spritesheet |
| `--sheet-type` | Tipo de layout do sheet |
| `--sheet-width`, `--sheet-height` | Tamanho máximo do sheet |
| `--sheet-padding`, `--inner-padding` | Espaçamentos |
| `--data <arquivo.json>` | Exporta dados JSON do sheet |
| `--format <formato>` | Formato JSON (aseprite, json-array, json-hash) |
| `--layer <nome>` | Exporta apenas a camada especificada |
| `--all-layers` | Inclui camadas ocultas |
| `--ignore-layer <nome>` | Exclui camada da exportação |
| `--frame-range <from,to>` | Intervalo de frames a exportar |
| `--tag <nome>` | Exporta apenas os frames de uma tag |
| `--scale <fator>` | Escala de saída |
| `--trim` | Recorta bordas transparentes |
| `--crop <x,y,w,h>` | Recorta região |
| `--split-layers` | Um arquivo por camada |
| `--split-tags` | Um arquivo por tag |
| `--split-frames` | Um arquivo por frame |
| `--filename-format <template>` | Template para nomes de arquivo de saída |
| `--list-layers` | Lista nomes das camadas |
| `--list-tags` | Lista tags de animação |
| `--list-slices` | Lista slices |
| `--script <arquivo.lua>` | Executa script Lua |
| `--script-param <chave=valor>` | Parâmetro para o script |
| `--color-mode <modo>` | Converte modo de cor |
| `--verbose` | Saída verbose |

### 11.2 Exemplos

```bash
# Exportar spritesheet
aseprite -b sprite.ase --sheet sheet.png --data sheet.json

# Exportar frames individuais
aseprite -b sprite.ase --save-as frame{frame}.png

# Executar script
aseprite -b --script meu_script.lua
```

---

## 12. Recuperação de Dados (`src/app/crash`)

### 12.1 Mecanismo de backup

- `BackupObserver` (`crash/backup_observer.h`): observa mudanças no documento
- A cada modificação e em intervalos regulares, serializa o estado dos documentos abertos em `%APPDATA%/Aseprite/sessions/<session_id>/`
- Cada backup contém: metadados do sprite, imagens dos cels, paletas, layers

### 12.2 Session (`crash/session.h`)

- Uma `Session` representa uma execução do Aseprite (identificada por PID)
- Ao iniciar, cria um diretório de sessão e grava o PID
- Ao fechar normalmente, chama `session.close()` que marca a sessão como concluída
- Se o processo termina abruptamente (crash), a sessão fica marcada como "crashed"

### 12.3 Recuperação na inicialização

- `DataRecovery` (`crash/data_recovery.h`): ao iniciar, busca sessões crashed em background thread
- Emite sinal `SessionsListIsReady` quando a lista está pronta
- Exibe `DataRecoveryView` na UI com a lista de arquivos recuperáveis
- O usuário pode recuperar (abre o sprite) ou deletar cada backup

---

## 13. Preferências e Configurações (`src/app/pref`)

### 13.1 Armazenamento

- Arquivo INI em `%APPDATA%/Aseprite/aseprite.ini` (Windows) ou `~/.config/aseprite/aseprite.ini`
- Gerado a partir de `pref.xml` via code generation (`src/gen/`)
- Acesso via `Preferences::instance()`

### 13.2 Categorias de preferências

| Categoria | Descrição |
|---|---|
| `GlobalPref` | Tema, idioma, zoom inicial, undo steps |
| `ToolPreferences` | Por ferramenta: brush, ink, opacidade, tamanho |
| `DocumentPreferences` | Por documento: grid, onion skin, tiled mode, zoom |
| Atalhos de teclado | Arquivo separado `user.aseprite-keys` |
| Extensões | Lista de extensões instaladas e estados |

---

## 14. Extensões (`src/app/extensions.h`)

### 14.1 Arquitetura

- Extensões são pacotes `.aseprite-extension` (ZIP com `package.json`)
- Gerenciadas por `Extensions` em `app/extensions.h`
- Instaladas em `%APPDATA%/Aseprite/extensions/`
- Carregadas na inicialização do app

### 14.2 Categorias de extensão

| Categoria | `Extension::Category` | Descrição |
|---|---|---|
| Keys | `Keys` | Arquivos de atalhos de teclado (`.aseprite-keys`) |
| Languages | `Languages` | Traduções (arquivos `.ini` com strings) |
| Themes | `Themes` | Temas visuais (XML de UI + sprites) |
| Scripts | `Scripts` | Scripts Lua (carregados automaticamente) |
| Palettes | `Palettes` | Paletas de cor adicionais |
| Dithering Matrices | `DitheringMatrices` | Matrizes customizadas para dithering ordenado |
| Multiple | `Multiple` | Extensão com mais de uma categoria |

---

## 15. Recursos Específicos de Pixel Art

### 15.1 Pixel Perfect Mode

Ao desenhar com a ferramenta lápis no modo pixel perfect (`as_pixel_perfect`), o intertwiner remove automaticamente pixels "L-shaped" que criam diagonais com aparência de escada dobrada, resultando em linhas diagonais mais limpas.

Arquivo: `src/app/tools/intertwiners.h` (classe `IntertwinedAsPixelPerfect`)

### 15.2 Wide Pixels (Pixel Ratio)

`PixelRatio` define a proporção de exibição dos pixels (ex: 2:1 = pixels duas vezes mais largos). Armazenado no sprite e considerado na projeção de renderização. Útil para efeitos de consoles antigos (CGA, NES, C64).

### 15.3 Tiled Mode (`filters/tiled_mode.h`, `app/commands/cmd_tiled_mode.cpp`)

- `TiledMode::NONE`, `X_AXIS`, `Y_AXIS`, `BOTH`
- Repete o sprite na tela nas direções ativas
- Permite desenhar padrões seamless com visualização imediata das bordas
- O brush aplicado fora das bordas "envolve" para o lado oposto

### 15.4 Reference Layers

Camadas com flag `Reference` (`LayerFlags::Reference`):
- Renderizadas no editor para guia visual (rotoscopia)
- **Não** incluídas na exportação final
- Útil para traçar sobre fotos ou animações de referência

### 15.5 Custom Brushes

- Qualquer região selecionada pode ser transformada em brush via `NewBrush`
- Brush de imagem (`BrushType::kImage`) usa a imagem como carimbo
- Padrão: `kAlignedToSource` (alinha ao grid) ou `kPaintBrush` (fixa ao brush)
- 10 slots de brushes persistidos entre sessões (`AppBrushes`)

### 15.6 Shading Ink

No modo Indexed ou com shades configurados:
- O shading ink desloca a cor dentro de uma ramp de shades
- Clicar com botão esquerdo avança na ramp (mais escuro/claro)
- Clicar com botão direito recua
- Shades são configuráveis pela Color Bar

### 15.7 Outline Effect (`filters/outline_filter.h`)

- Adiciona contorno ao redor dos pixels opacos
- Configurável: cor, espessura, contorno dentro/fora, cantos
- Aplicado como filtro não-destrutivo ou como operação permanente

### 15.8 Symmetry Tool

- Eixos horizontal e/ou vertical configuráveis
- Posição do eixo ajustável no contexto bar
- Cada pincelada é duplicada/espelhada em tempo real
- Pode ter múltiplos eixos ativos simultaneamente

---

## 16. Fluxo de Trabalho de Animação

### 16.1 Estrutura frame × camada

Cada frame tem duração própria em milissegundos. O cel de uma camada em um frame pode ser:
- **Com imagem**: cel com conteúdo próprio
- **Linked**: compartilha `CelData` com outro frame
- **Vazio**: sem cel (camada invisível naquele frame)

### 16.2 Tags de Animação

Tags agrupam frames em sequências nomeadas com direção configurável:
- `Forward`: reproduz do primeiro ao último frame da tag
- `Reverse`: reproduz do último ao primeiro
- `PingPong`: vai e volta
- `PingPongReverse`: vai e volta começando pelo final

### 16.3 Reprodução

- `Playback` (`doc/playback.h`): lógica de reprodução respeitando tags, direção e loop
- Loop tags: define a seção que repete ao usar `SetLoopSection`
- Velocidade ajustável globalmente (multiplicador)
- Opção de reproduzir apenas a tag ativa ou todas as tags em sequência

### 16.4 Onion Skinning

Configurado via `OnionskinOptions`:
- Número de frames anteriores e posteriores visíveis
- Opacidade decresce por `opacityStep` para cada frame adicional
- Pode ser restrito a uma tag ou a uma camada
- Dois modos: mesclado (tons normais) ou tint vermelho/azul

### 16.5 Exportação de Animação

- **GIF animado**: nativo, até 256 cores por frame com dithering
- **WebP animado**: qualidade superior, suporte a alpha total
- **Sprite Sheet**: frames empacotados em uma imagem + JSON de metadados
- **Sequência de PNG**: `frame001.png`, `frame002.png`, ...
- **FLC/FLI**: formato legado para compatibilidade

---

## 17. Gerenciamento de Cores

### 17.1 Modos de cor (`doc/color_mode.h`)

| Modo | Bytes/pixel | Descrição |
|---|---|---|
| `RGBA` | 4 | Canal vermelho, verde, azul e alfa |
| `Grayscale` | 2 | Valor de cinza e canal alfa |
| `Indexed` | 1 | Índice de 0–255 na paleta ativa |

### 17.2 Conversão entre modos

- `ChangePixelFormat` (`cmd_change_pixel_format.cpp`): converte o sprite inteiro
- RGBA→Indexed usa quantização (Median Cut) + dithering opcional
- Indexed→RGBA mapeia cada índice para RGBA da paleta

### 17.3 Perfis de Cor (Color Profiles)

- Suporte a perfis ICC (sRGB, Display P3, etc.)
- `AssignColorProfile`: atribui perfil sem converter os valores
- `ConvertColorProfile`: converte os valores dos pixels para o novo espaço
- Configurável por sprite em `SpriteProperties`

### 17.4 Operações de Paleta

- Ordenação por matiz, saturação, brilho, luminância, valor R/G/B
- Gradiente entre duas cores selecionadas
- Quantização automática das cores usadas no sprite
- Reversão da paleta

### 17.5 Alpha Channel

- Cel com opacidade: multiplica o alpha de cada pixel
- Layer com opacidade: multiplicação adicional
- Blend modes diferentes alteram como o alpha é composto
- Lock Alpha ink: preserva o alpha existente ao pintar

---

## 18. Módulos Auxiliares

### 18.1 Filtros (`src/filters/`)

| Filtro | Classe | Descrição |
|---|---|---|
| Brilho/Contraste | `BrightnessContrastFilter` | Ajuste linear de brilho e contraste por canal |
| Matiz/Saturação | `HueSaturationFilter` | Ajuste HSV/HSL; modos: HSV, HSL, HWB, Linear HSL |
| Curva de Cor | `ColorCurveFilter` | Curvas de ajuste R/G/B/alpha com spline |
| Inverter Cor | `InvertColorFilter` | 255 − canal para R/G/B |
| Contorno | `OutlineFilter` | Detecta bordas e aplica cor de contorno |
| Convolução | `ConvolutionMatrixFilter` | Aplica kernel customizado (blur, sharpen, emboss, etc.) |
| Mediana | `MedianFilter` | Remove ruído preservando bordas |
| Substituir Cor | `ReplaceColorFilter` | Substitui cor por outra com tolerância |

Todos implementam `Filter` (`filters/filter.h`) e são aplicados via `FilterManager` com suporte a preview em tempo real e aplicação em cels específicos ou em intervalos.

### 18.2 FixMath (`src/fixmath/`)

Biblioteca de ponto fixo (`.16` = 16 bits fracionários) para operações geométricas de precisão em ambiente onde floats eram problemáticos. Usada internamente pelo sistema de algoritmos de pixel (traçado de linhas, elipses).

### 18.3 Algoritmos de Documento (`src/doc/algorithm/`)

- `FloodFill`: preenchimento por inundação (4-conectado ou 8-conectado)
- `ResizeImage`: redimensionamento com filtros (Nearest, Bilinear, Bicubic, Lanczos, etc.)
- `FlipImage` / `FlipType`: espelhamento horizontal/vertical
- `Polygon`: rasterização de polígonos
- `StrokeSelection`: contorno de seleção

### 18.4 Sistema de Grid (`doc/grid.h`)

- Tamanho da célula (width × height)
- Origem (offset x, y)
- Snap to Grid: arredonda coordenadas para o grid mais próximo
- Exportação de tiles baseada no grid

---

*Documentação gerada a partir do código-fonte em `src/`. Para informações adicionais, consulte os arquivos de especificação em `docs/ase-file-specs.md` e a API de scripting oficial em https://www.aseprite.org/docs/scripting/.*
