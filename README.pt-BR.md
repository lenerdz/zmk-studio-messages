# ZMK Studio Messages

Este repositório contém as definições de mensagens usadas para interagir com dispositivos habilitados para o ZMK Studio.

## TODO

* Documentar o protocolo de transporte usado com essas mensagens
* Estratégia de lançamento/versionamento

## Mudanças do fork: mapas de cor dos LEDs

Este é o fork [lenerdz/zmk-studio-messages](https://github.com/lenerdz/zmk-studio-messages) de [zmkfirmware/zmk-studio-messages](https://github.com/zmkfirmware/zmk-studio-messages). Ele acrescenta um pequeno conjunto de requisições ao subsistema `keymap`, para que um cliente do ZMK Studio possa ler e editar os mapas de cor dos LEDs de cada camada do módulo [lenerdz/zmk-rgb-matrix](https://github.com/lenerdz/zmk-rgb-matrix), e as configurações de tela (OLED) dele, que incluem o layout das telas (veja [Mudanças do fork: configurações de tela](#mudanças-do-fork-configurações-de-tela)).

Toda mensagem e toda tag do upstream continuam iguais. Os novos membros dos campos `oneof` usam tags `>= 100` (o upstream usa números pequenos hoje), de modo que uma futura adição do upstream não pode colidir com eles. O ZMK guarda a tag da requisição em um `uint8_t`, então as tags precisam ficar abaixo de 256.

Um firmware que não inclui os handlers do `lenerdz/zmk-rgb-matrix` responde a essas requisições com o erro simples `RPC_NOT_FOUND`. É assim que um cliente descobre que o recurso não existe.

### Slots

Um mapa tem `slot_count` slots, e cada slot guarda uma cor de 24 bits `0xRRGGBB`. No teclado em que este fork é usado (um Corne), `slot_count` é 54:

| Slots | Significado |
|---|---|
| 0 a 41 | posições de tecla (a mesma numeração dos bindings do keymap) |
| 42 a 47 | underglow esquerdo, na ordem da fita de LEDs |
| 48 a 53 | underglow direito, na ordem da fita de LEDs |

Um mapa pertence a uma camada e é endereçado pelo id dessa camada (`Layer.id` do `get_keymap`), então ele acompanha a camada quando as camadas são reordenadas.

### Requisições

Todas as requisições são protegidas: o teclado precisa estar desbloqueado, como em qualquer edição do keymap.

| Requisição (tag) | Conteúdo | Resposta (tag) |
|---|---|---|
| `get_led_info` (100) | `bool` | `LedInfo`: `slot_count`, os ids das camadas que têm um mapa e `unsaved_changes` |
| `get_led_map` (101) | `uint32` id da camada | `LedMap`: o id da camada e exatamente 54 cores |
| `set_led_colors` (102) | `SetLedColorsRequest`: id da camada e até 16 `LedColorEntry` (`slot`, `rgb`) | `SetLedColorsResponse`: `OK`, `INVALID_LAYER`, `INVALID_SLOT` ou `BUSY` |
| `save_led_changes` (103) | `bool` | `SaveChangesResponse`, o mesmo tipo que o keymap usa |
| `discard_led_changes` (104) | `bool` | `bool` |
| `restore_led_defaults` (105) | `uint32` id da camada | `LedMap`: o mapa depois de voltar aos padrões do firmware |

A notificação `unsaved_led_changes_status_changed` (tag 100) carrega um `bool`: `true` quando algum mapa difere do que está salvo.

Uma edição vale no teclado na hora. Ela só é mantida entre reinicializações depois de um `save_led_changes`. O `discard_led_changes` devolve todos os mapas ao último estado salvo.

### Limites

Os tamanhos vêm do `keymap.options.in` (nanopb, sem heap). Os campos abaixo são vetores de capacidade fixa; `DisplayInfo.settings` é a única lista que não tem limite (veja [Mudanças do fork: configurações de tela](#mudanças-do-fork-configurações-de-tela)):

| Campo | Limite |
|---|---|
| `LedMap.colors` | exatamente 54 entradas |
| `LedInfo.layer_ids` | 15 entradas (ids de camada de 0 a 14) |
| `SetLedColorsRequest.entries` | 16 entradas (um cliente envia uma edição maior em várias requisições) |

## Mudanças do fork: configurações de tela

O mesmo fork acrescenta mais três requisições ao subsistema `keymap`, para as configurações de tela (OLED) que o módulo [lenerdz/zmk-rgb-matrix](https://github.com/lenerdz/zmk-rgb-matrix) guarda: o contraste de cada tela, quando as telas apagam, o aviso das luzes (toast) do [lenerdz/zmk-nice-oled](https://github.com/lenerdz/zmk-nice-oled) e, com o layout das telas OLED, quais widgets das telas dele são desenhados, onde, e qual animação a tela direita toca. As tags delas são 106 a 108, na mesma faixa reservada (`>= 100`), e um firmware sem os handlers responde a elas com `RPC_NOT_FOUND`, como nas requisições das luzes.

As configurações são uma lista de inteiros numerados. Para cada configuração que tem, o firmware informa o valor atual e a faixa, de modo que um cliente desenha um controle sem conhecer a configuração; o significado dos ids é fixo (tabela abaixo) para que um cliente também possa rotulá-los. Uma mudança vale no teclado na hora e o firmware a mantém entre reinicializações por conta própria, então não há requisição de salvar nem de descartar e não há notificação. Todas as requisições são protegidas: o teclado precisa estar desbloqueado.

A lista não tem tamanho fixo. O `DisplayInfo.settings` não tem `max_count` no `keymap.options.in`, então o nanopb o transforma em um campo com callback: o firmware escreve quantas entradas tiver, uma depois da outra, e um cliente lê todas as que chegam. O formato no fio é o de qualquer campo repetido, então o cliente TypeScript gerado não muda, e a resposta não aumenta a union que o ZMK Studio mantém na pilha.

| Requisição (tag) | Conteúdo | Resposta (tag) |
|---|---|---|
| `get_display_info` (106) | `bool` | `DisplayInfo`: a lista de `DisplaySetting` (`id`, `value`, `min`, `max`, `default_value`) |
| `set_display_setting` (107) | `SetDisplaySettingRequest`: `id` e `value` | `SetDisplaySettingResponse`: `OK`, `INVALID_ID` ou `INVALID_VALUE` |
| `restore_display_defaults` (108) | `bool` | `DisplayInfo`: as configurações depois de voltarem aos padrões do firmware |

Os ids que este fork define (o firmware do [lenerdz/zmk-rgb-matrix](https://github.com/lenerdz/zmk-rgb-matrix) é a referência; os ids ficam abaixo de 256, porque o firmware os envia a uma periférica em 8 bits):

| Id | Configuração | Faixa | Padrão |
|---|---|---|---|
| 1 | Contraste da tela esquerda (central) | 0 a 255 | 128 |
| 2 | Contraste da tela direita (periférica) | 0 a 255 | 128 |
| 3 | Quando a tela apaga depois que o teclado fica ocioso: 0 nunca, 1 só na bateria (fica acesa enquanto o USB está conectado), 2 sempre | 0 a 2 | 0 |
| 4 | Segundos sem atividade antes de a tela apagar; não menos que o tempo de ociosidade do teclado (`CONFIG_ZMK_IDLE_TIMEOUT`, 30 s por padrão); só vale quando o id 3 não é 0 | do tempo de ociosidade a 3600 | 60 |
| 5 | Aviso das luzes (toast) na tela direita: 0 desligado, 1 ligado (só tem efeito quando a metade direita é compilada com o widget do toast) | 0 a 1 | 1 |
| 6 | Quanto tempo o toast fica na tela, em milissegundos | 500 a 5000 | 1500 |

A partir do id 16 vêm as configurações do layout das telas OLED: um bloco de oito ids por widget, em `16 + 8 * n`. Em um bloco, `+0` diz se o widget é desenhado (0 ou 1, padrão 1), `+1` é o x dele e `+2` o y (pixels, de -160 a 320) e, para a animação da tela direita, `+3` é o tipo dela e `+4` o tempo de um ciclo. O firmware lista só os widgets com que foi compilado, e os padrões são o que o build diz (as opções `CONFIG_NICE_OLED_WIDGET_*` do [lenerdz/zmk-nice-oled](https://github.com/lenerdz/zmk-nice-oled)), então um cliente precisa lê-los do `default_value`.

| Ids | Widget | Tela |
|---|---|---|
| 16 a 20 | animação ou imagem parada: mostrar, x, y, tipo (0 crystal, 1 cat, 2 head, 3 spaceman, 4 pokemon, 5 vim, 6 vip_marcos, e de 7 a 65 as imagens que o `make_pictures.py` do zmk-nice-oled desenha), tempo de um ciclo em milissegundos (0, ou 100 a 60000; 0 é o tempo da animação) | direita |
| 112 a 115 | a mesma animação ou imagem parada: giro (0 a 3 quartos de volta), mistura (0 soma, 1 inverte), canto do pedaço que aparece (`x \| y << 8`) e o tamanho dele (`w \| h << 8`) | direita |
| 24 a 26 | bateria da metade direita | direita |
| 32 a 34 | ícone de Bluetooth | direita |
| 41 e 42 | toast das luzes: x e y da primeira linha dele (não tem interruptor próprio: o interruptor é o id 5) | direita |
| 48 a 50 | ícone de USB ou Bluetooth da saída ativa | esquerda |
| 56 a 58 | bateria | esquerda |
| 64 a 66 | os cinco pontos dos perfis Bluetooth | esquerda |
| 72 a 74 | número do perfil Bluetooth ativo | esquerda |
| 80 a 82 | nome da camada ativa | esquerda |
| 88 a 90 | velocímetro de WPM | esquerda |
| 96 a 98 | número de WPM | esquerda |
| 104 a 106 | gráfico de WPM | esquerda |

As coordenadas são as do canvas em que os widgets são desenhados, com uma exceção, a animação da tela direita. As condições de cada entrada e a conversão das coordenadas da animação estão na seção [OLED layout](https://github.com/lenerdz/zmk-rgb-matrix/blob/corne-fixes/README.md#oled-layout) do README do módulo.

Um cliente deve desenhar um controle a partir do `min`, do `max` e do `default_value` que o firmware informa, e não a partir desta tabela: a tabela diz o que cada id significa. Um firmware pode omitir uma configuração que não tem, e um cliente deve mostrar um id que não conhece como um número simples.

## Mudanças do fork: estado ao vivo e informações do firmware

O mesmo fork acrescenta duas requisições somente de leitura ao subsistema `keymap`, tags 109 e 110 (a mesma faixa reservada, `>= 100`), para que um cliente veja o que o teclado está fazendo agora e que firmware ele executa. O módulo [lenerdz/zmk-rgb-matrix](https://github.com/lenerdz/zmk-rgb-matrix) responde a elas (`CONFIG_KEYPAW_DEVICE_STATUS_STUDIO`), e um firmware sem os handlers responde `RPC_NOT_FOUND`, como nas outras requisições do fork. As duas são protegidas: o teclado precisa estar desbloqueado. Elas não mudam nada e não guardam nada: não há requisição de salvar e não há notificação, então um cliente que quer uma visão ao vivo pede `get_live_status` de novo e de novo.

| Requisição (tag) | Conteúdo | Resposta (tag) |
|---|---|---|
| `get_live_status` (109) | `bool` | `LiveStatus` |
| `get_firmware_info` (110) | `bool` | `FirmwareInfo` |

`LiveStatus` é o que as telas do teclado desenham agora:

| Campo (número) | Tipo | Significado |
|---|---|---|
| `battery_central` (1) | `uint32` | bateria da metade central, de 0 a 100; 255 quando não há leitura |
| `battery_peripheral` (2) | `uint32` | bateria da metade periférica, de 0 a 100; 255 quando ela ainda não informou |
| `peripheral_connected` (3) | `bool` | a central enxerga a periférica |
| `layer_index` (4) | `uint32` | a camada ativa mais alta, como o índice dela na ordem do keymap |
| `layer_name` (5) | `string` | o nome dessa camada; vazio quando ela não tem |
| `wpm` (6) | `uint32` | palavras por minuto agora |
| `output` (7) | `LiveOutput` | para onde vão as teclas: `LIVE_OUTPUT_NONE` (0), `LIVE_OUTPUT_USB` (1) ou `LIVE_OUTPUT_BLE` (2) |
| `ble_profile` (8) | `uint32` | o perfil Bluetooth selecionado, a partir de 0 |
| `ble_state` (9) | `LiveBleState` | o estado desse perfil: `LIVE_BLE_OPEN` (0, nenhum host pareado), `LIVE_BLE_PAIRED` (1, host não conectado) ou `LIVE_BLE_CONNECTED` (2) |
| `usb_state` (10) | `LiveUsbState` | o conector USB: `LIVE_USB_NONE` (0), `LIVE_USB_POWERED` (1, energia, mas o host ainda não configurou o teclado) ou `LIVE_USB_HID` (2, o host o usa) |
| `uptime_s` (11) | `uint32` | segundos desde que o teclado ligou |

`FirmwareInfo` diz o que o firmware é, e não muda enquanto o teclado funciona:

| Campo (número) | Tipo | Significado |
|---|---|---|
| `keyboard` (1) | `string` | o nome do teclado |
| `board` (2) | `string` | a placa |
| `zephyr` (3) | `string` | a versão do Zephyr |
| `build_id` (4) | `string` | o commit que foi compilado, curto; vazio quando não se sabe |
| `build_branch` (5) | `string` | a branch desse commit; vazio quando não se sabe |
| `build_number` (6) | `uint32` | o número da execução do CI que o compilou; 0 quando não se sabe |
| `build_date` (7) | `string` | o dia em que foi compilado, `yyyy-mm-dd` (UTC) |
| `flash_total` (8) | `uint32` | bytes de flash para o programa (a partição de código) |
| `flash_used` (9) | `uint32` | bytes dela que o programa ocupa |
| `ram_total` (10) | `uint32` | bytes de RAM |
| `ram_used` (11) | `uint32` | bytes dela que o programa ocupa assim que começa |

Os valores são os da metade central, porque o ZMK Studio fala com ela: tudo o que ela diz sobre a outra metade é se está conectada e o nível da bateria. As strings têm um tamanho no `keymap.options.in`, e o firmware corta o que não cabe: `layer_name` `CONFIG_ZMK_KEYMAP_LAYER_NAME_MAX_LEN` (o limite do nome em `SetLayerPropsRequest`), `keyboard` e `board` 32 bytes, `zephyr` 24, `build_id` 16, `build_branch` 48 e `build_date` 12 (cada uma com o seu fim de string). As mensagens novas não aumentam a union que o ZMK Studio mantém na pilha: com o nanopb, `Request` tem 140 bytes e `Response` 232, com e sem elas.

## Usando em uma config do ZMK


Fixe este fork no `west.yml` do repositório de configuração. O nome do projeto e o caminho precisam ser iguais ao da entrada do upstream no `app/west.yml` do ZMK, então esta entrada a substitui:

```yaml
    - name: zmk-studio-messages
      remote: lenerdz
      revision: <commit>
      path: modules/msgs/zmk-studio-messages
```
