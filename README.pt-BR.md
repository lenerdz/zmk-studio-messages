# ZMK Studio Messages

Este repositório contém as definições de mensagens usadas para interagir com dispositivos habilitados para o ZMK Studio.

## TODO

* Documentar o protocolo de transporte usado com essas mensagens
* Estratégia de lançamento/versionamento

## Mudanças do fork: mapas de cor dos LEDs

Este é o fork [lenerdz/zmk-studio-messages](https://github.com/lenerdz/zmk-studio-messages) de [zmkfirmware/zmk-studio-messages](https://github.com/zmkfirmware/zmk-studio-messages). Ele acrescenta um pequeno conjunto de requisições ao subsistema `keymap`, para que um cliente do ZMK Studio possa ler e editar os mapas de cor dos LEDs de cada camada do módulo [lenerdz/zmk-rgb-matrix](https://github.com/lenerdz/zmk-rgb-matrix), e as configurações de tela (OLED) dele (veja [Mudanças do fork: configurações de tela](#mudanças-do-fork-configurações-de-tela)).

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

Os tamanhos vêm do `keymap.options.in` (nanopb, sem heap e sem callbacks):

| Campo | Limite |
|---|---|
| `LedMap.colors` | exatamente 54 entradas |
| `LedInfo.layer_ids` | 15 entradas (ids de camada de 0 a 14) |
| `DisplayInfo.settings` | 16 entradas |
| `SetLedColorsRequest.entries` | 16 entradas (um cliente envia uma edição maior em várias requisições) |

## Mudanças do fork: configurações de tela

O mesmo fork acrescenta mais três requisições ao subsistema `keymap`, para as configurações de tela (OLED) que o módulo [lenerdz/zmk-rgb-matrix](https://github.com/lenerdz/zmk-rgb-matrix) guarda: o contraste de cada tela, quando as telas apagam e o aviso das luzes (toast) do [lenerdz/zmk-nice-oled](https://github.com/lenerdz/zmk-nice-oled). As tags delas são 106 a 108, na mesma faixa reservada (`>= 100`), e um firmware sem os handlers responde a elas com `RPC_NOT_FOUND`, como nas requisições das luzes.

As configurações são uma lista de inteiros numerados. Para cada configuração que tem, o firmware informa o valor atual e a faixa, de modo que um cliente desenha um controle sem conhecer a configuração; o significado dos ids é fixo (tabela abaixo) para que um cliente também possa rotulá-los. Uma mudança vale no teclado na hora e o firmware a mantém entre reinicializações por conta própria, então não há requisição de salvar nem de descartar e não há notificação. Todas as requisições são protegidas: o teclado precisa estar desbloqueado.

| Requisição (tag) | Conteúdo | Resposta (tag) |
|---|---|---|
| `get_display_info` (106) | `bool` | `DisplayInfo`: a lista de `DisplaySetting` (`id`, `value`, `min`, `max`, `default_value`) |
| `set_display_setting` (107) | `SetDisplaySettingRequest`: `id` e `value` | `SetDisplaySettingResponse`: `OK`, `INVALID_ID` ou `INVALID_VALUE` |
| `restore_display_defaults` (108) | `bool` | `DisplayInfo`: as configurações depois de voltarem aos padrões do firmware |

Os ids que este fork define (o firmware do [lenerdz/zmk-rgb-matrix](https://github.com/lenerdz/zmk-rgb-matrix) é a referência; os ids ficam abaixo de 16):

| Id | Configuração | Faixa | Padrão |
|---|---|---|---|
| 1 | Contraste da tela esquerda (central) | 0 a 255 | 128 |
| 2 | Contraste da tela direita (periférica) | 0 a 255 | 128 |
| 3 | Quando a tela apaga depois que o teclado fica ocioso: 0 nunca, 1 só na bateria (fica acesa enquanto o USB está conectado), 2 sempre | 0 a 2 | 0 |
| 4 | Segundos sem atividade antes de a tela apagar; não menos que o tempo de ociosidade do teclado (`CONFIG_ZMK_IDLE_TIMEOUT`, 30 s por padrão); só vale quando o id 3 não é 0 | do tempo de ociosidade a 3600 | 60 |
| 5 | Aviso das luzes (toast) na tela direita: 0 desligado, 1 ligado (só tem efeito quando a metade direita é compilada com o widget do toast) | 0 a 1 | 1 |
| 6 | Quanto tempo o toast fica na tela, em milissegundos | 500 a 5000 | 1500 |

Um cliente deve desenhar um controle a partir do `min`, do `max` e do `default_value` que o firmware informa, e não a partir desta tabela: a tabela diz o que cada id significa. Um firmware pode omitir uma configuração que não tem, e um cliente deve mostrar um id que não conhece como um número simples.

## Usando em uma config do ZMK

Fixe este fork no `west.yml` do repositório de configuração. O nome do projeto e o caminho precisam ser iguais ao da entrada do upstream no `app/west.yml` do ZMK, então esta entrada a substitui:

```yaml
    - name: zmk-studio-messages
      remote: lenerdz
      revision: <commit>
      path: modules/msgs/zmk-studio-messages
```
