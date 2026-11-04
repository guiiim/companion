# Companheiro ESP32-S3

Um companheiro robótico de mesa com **olhos animados em um display colorido**, que percebe quando alguém chega ou se aproxima, **obedece a comandos de voz em português**, responde com **sons de robô** e toca **músicas em ordem aleatória**, mostrando o nome do arquivo abaixo dos olhos.

> **Status:** projeto em fase de planejamento. Este documento descreve a proposta, a lista de componentes, as ligações e o plano de execução.

## Funcionalidades

- **Olhos expressivos** em display IPS colorido, com piscadas, olhar ao redor e humores (feliz, bravo, triste, surpreso, sonolento, dormindo).
- **Relógio**: a hora é obtida por NTP na inicialização e exibida no lugar dos olhos.
- **Detecção de presença** com dois sensores: o PIR percebe a chegada de alguém e o VL53L0X mede a distância para reações graduais e carinho.
- **Comandos de voz offline**, em português, sem botões: uma palavra de ativação seguida de um comando curto.
- **Sons de robô** (bips, chilreios e tons) para cada situação, gerados por código ou tocados a partir de arquivos de áudio.
- **Música**: o comando "música" toca os MP3 guardados na memória flash em ordem aleatória, com o nome do arquivo em letras pequenas abaixo dos olhos.
- **Modo sono**: sem presença por 2 minutos, os olhos fecham e o backlight reduz.

## Componentes

| Item | Função | Observação |
|---|---|---|
| ESP32-S3 DevKitC-1 (N16R8 ou N8R8, com PSRAM) | Cérebro | Wi-Fi, dois núcleos, aceleração para IA |
| Display TFT IPS 1,54" ST7789 240x240 (SPI) | Olhos e relógio | Muitos módulos não têm pino CS |
| Microfone INMP441 (I2S) | Comandos de voz | Digital, sem ruído analógico |
| Amplificador MAX98357A + alto-falante 4 a 8 Ω, 3 W (~40 mm) | Sons e música | I2S |
| Sensor VL53L0X (ToF) | Distância, aproximação e carinho | I2C, alcance até ~1,2 m |
| Sensor PIR (AM312 ou HC-SR501) | Chegada de pessoas e modo sono | AM312 usa 3,3 V; HC-SR501 usa 5 V |
| Capacitor eletrolítico 470 a 1000 µF / 10 V | Estabiliza a alimentação do amplificador | Entre VIN e GND do MAX98357A |
| Fonte USB 5 V / 2 A | Alimentação | O áudio tem picos de corrente |
| Jumpers, protoboard e caixa | Montagem | Impressa em 3D ou de papelão |

Opcional: módulo RTC DS3231, para um relógio independente de Wi-Fi.

## Diagrama de blocos

```
                        ┌─────────────────────┐
  PIR ──────────────────┤                     ├──── ST7789 (olhos)  [SPI]
  VL53L0X ──── I2C ─────┤     ESP32-S3        │
  INMP441 ──── I2S0 ────┤                     ├──── MAX98357A ──► Alto-falante [I2S1]
                        │  Wi-Fi (só NTP)     │
                        └─────────────────────┘
                              USB 5V / 2A
```

## Ligações

Nos modelos com PSRAM octal, os pinos GPIO26 a 37 são usados internamente. Os pinos abaixo evitam essa faixa e os pinos de boot. Os números são uma sugestão: **confira o pinout da sua placa antes de soldar**.

| Componente | Pino do módulo | GPIO do ESP32-S3 |
|---|---|---|
| ST7789 | VCC / GND | 3V3 / GND |
| | SCL (SCK) | GPIO12 |
| | SDA (MOSI) | GPIO11 |
| | DC | GPIO9 |
| | RES | GPIO8 |
| | BLK (backlight) | GPIO7 (PWM, controla o brilho) |
| | CS | sem ligação (`TFT_CS = -1`), se o módulo não tiver |
| INMP441 | VDD / GND | 3V3 / GND |
| | SCK | GPIO15 |
| | WS | GPIO16 |
| | SD | GPIO17 |
| | L/R | GND |
| MAX98357A | VIN / GND | 5V / GND (com capacitor) |
| | BCLK | GPIO18 |
| | LRC | GPIO21 |
| | DIN | GPIO47 |
| | SD / GAIN | livres (SD aberto = ligado) |
| | Saída + / − | Alto-falante |
| VL53L0X | VIN / GND | 3V3 / GND |
| | SDA | GPIO4 |
| | SCL | GPIO5 |
| PIR | OUT | GPIO6 |
| | VCC / GND | 5V (HC-SR501) ou 3V3 (AM312) / GND |

## Sensores de presença

| Sensor | O que detecta | Papel no projeto |
|---|---|---|
| PIR | Calor corporal em movimento (cerca de 100° e 3 a 7 m), só liga/desliga | Acorda o companheiro e controla o modo sono |
| VL53L0X | Distância a laser, de poucos cm a ~1,2 m, com a pessoa parada | Reações graduais, gestos e carinho |

Lógica combinada:

```
PIR dispara                  → acorda (se dormindo) e olha ao redor
ToF < 80 cm                  → olhos acompanham
ToF < 30 cm                  → curioso
ToF < 10 cm                  → surpreso
ToF < 6 cm por > 2 s         → carinho (se houver música tocando, ela para)
Sem PIR e sem ToF por 2 min  → sonolento → dormindo
Palavra de ativação          → acorda mesmo no sono (microfone sempre ativo)
```

## Visor

O display 240x240 mostra os olhos, o relógio e o nome da música em reprodução.

| Estado | Olhos / tela |
|---|---|
| Repouso | Olhos ciano que piscam a cada 3 a 6 s e olham de lado de vez em quando |
| Chegou alguém (PIR) | Arregalam, depois ficam felizes, com chilreio de saudação |
| Pessoa se aproximando (ToF) | Acompanham; curiosos a 30 cm; surpresos a menos de 10 cm |
| Carinho (mão parada a ~5 cm por 2 s) | Olhos fechados sorrindo e som de ronronar; se houver música tocando, ela para |
| Palavra de ativação ouvida | Olhos bem abertos com anel pulsante |
| Comando reconhecido | Olhos felizes e som de confirmação |
| Comando não entendido | Olhos confusos e um som de dúvida |
| Sem presença por 2 min | Sonolento, depois dormindo com Zzz e backlight reduzido |
| Relógio | Hora grande e data abaixo, na cor do tema |
| Tocando música | Olhos normais com o nome do arquivo em letras pequenas logo abaixo |

Cores por humor: ciano ou verde (normal), vermelho (bravo), rosa (apaixonado), azul (sonolento).

**Como desenhar:** com LovyanGFX ou TFT_eSPI (SPI a 40 a 80 MHz). Os olhos são desenhados com `fillRoundRect` (corpo), círculos (brilho) e retângulos pretos (pálpebras), redesenhando só a área que muda. Como alternativa, um sprite (`TFT_eSprite`) de cerca de 115 KB na PSRAM serve de buffer.

**Nome da música:** exibido sem a extensão `.mp3`, em uma linha pequena abaixo dos olhos. Se passar da largura da tela, o texto rola devagar. Use uma fonte com suporte a acentos.

## Voz

O microfone INMP441 é digital (I2S) e o ESP32 o lê por DMA, sem travar o resto do programa. Configuração típica: 16 kHz, mono, amostras de 32 bits.

| Nível | O que faz | Dificuldade |
|---|---|---|
| 1. Detecção de som | Calcula o volume (RMS) e reage a palmas ou barulho | Fácil, 100% no ESP32 |
| 2. Palavra de ativação e comandos curtos | Reconhece "Ei, Bolinha", "mostrar hora", "dormir" | Médio |

**Idioma:** português, com modelo treinado no [Edge Impulse](https://edgeimpulse.com). Grave de 50 a 100 amostras de cada comando e limite o vocabulário a cerca de 8 a 15 palavras, curtas e bem diferentes entre si.

Uma palavra de ativação (por exemplo, "Ei, Bolinha") abre uma janela de cerca de 5 segundos para o comando, o que evita disparos por conversas na sala.

### Comandos

| Comando | Ação | Resposta sonora |
|---|---|---|
| "Mostrar hora" | Troca olhos por relógio | Bip duplo |
| "Olhos" | Volta aos olhos | Bip agudo |
| "Dormir" | Olhos fecham e backlight apaga | Som descendente |
| "Acordar" | Reativa tudo | Som ascendente |
| "Feliz" / "Bravo" / "Triste" | Muda humor e cor dos olhos | Som correspondente |
| "Volume mais" / "Volume menos" | Ajusta o volume | Bip no novo volume |
| "Música" | Toca todos os MP3 em ordem aleatória e mostra o nome do arquivo abaixo dos olhos | Bip de confirmação |
| "Parar" | Interrompe a música e volta aos olhos normais | Tom descendente curto |

## Sons

O companheiro não fala: ele usa apenas **sons de robô**, gerados por código no ESP32 (ondas com variação de frequência) ou tocados a partir de arquivos de áudio.

| Evento | Som |
|---|---|
| Ligar | Três notas ascendentes rápidas |
| Palavra de ativação ouvida | Bip curto e agudo |
| Comando reconhecido | Duas notas ascendentes (confirmação) |
| Comando não entendido | Tom que sobe e desce (dúvida) |
| "Mostrar hora" | Bip duplo |
| "Olhos" | Bip agudo |
| "Dormir" | Tom descendente suave |
| "Acordar" | Tom ascendente suave |
| Humor feliz | Chilreio rápido e agudo |
| Humor bravo | Tom grave e áspero |
| Humor triste | Tom descendente lento |
| Chegou alguém (PIR) | Chilreio de saudação |
| Pessoa muito perto (ToF) | Bip agudo e rápido (susto) |
| Carinho | Ronronar: tom grave com vibração |
| "Volume mais" / "Volume menos" | Bip no novo volume |

Também é possível tocar sons de robô criados em um editor de áudio. Salve-os como WAV mono de 16 bits (a 16 kHz, cerca de 32 KB por segundo) na memória flash (LittleFS) e toque-os pelo MAX98357A com a biblioteca [ESP32-audioI2S](https://github.com/schreibfaul1/ESP32-audioI2S).

## Música

Ao ouvir "música", o companheiro toca todos os MP3 guardados na memória flash, em ordem aleatória, até receber o comando "parar" ou o gesto de carinho.

- **Armazenamento:** os MP3 ficam na flash interna do ESP32-S3, em uma partição LittleFS, sem módulo extra. Configure o esquema de partições para reservar a maior parte dos 16 MB aos arquivos (por exemplo, cerca de 12 MB).
- **Capacidade:** a 128 kbps, cada minuto ocupa cerca de 1 MB, ou seja, cerca de 12 minutos de música. A 64 kbps, cerca de 25 minutos.
- **Gravação dos arquivos:** os MP3 são enviados pelo computador, com a ferramenta de upload do sistema de arquivos do PlatformIO ou da Arduino IDE.
- **Ordem aleatória:** ao iniciar, o ESP32 lista os MP3, embaralha a lista e toca uma faixa depois da outra, sem repetir. Quando a lista acaba, embaralha de novo e continua.
- **Modo sono:** fica desativado enquanto a música toca.
- **Parar a música:** pelo comando "parar" ou aproximando a mão do VL53L0X, parada a cerca de 5 cm por 2 segundos (o mesmo gesto do carinho). O gesto funciona mesmo com a música alta; o comando de voz pode falhar se o alto-falante estiver perto do microfone, então mantenha o volume moderado.
- **Qualidade:** o alto-falante de 40 mm e 3 W não reproduz graves e toca em mono, então serve como música ambiente. Para mais qualidade, use um alto-falante maior.

> **Expansão:** para guardar mais músicas, é possível adicionar um módulo de cartão microSD (SPI) e um cartão microSD, que aumentam muito a capacidade.

## Software

- **Ambiente:** Arduino IDE ou PlatformIO.
- **Gráficos:** LovyanGFX ou TFT_eSPI.
- **Áudio:** ESP32-audioI2S para reprodução e o driver I2S do ESP-IDF (`driver/i2s.h` ou `i2s_std.h`) para capturar o microfone.
- **Voz:** modelo de palavras-chave treinado no Edge Impulse.
- **Relógio:** NTP com `pool.ntp.br` e fuso `<-03>3`, via `configTzTime`, na inicialização. O Wi-Fi só é usado para isso.

### Organização das tarefas

O ESP32-S3 tem dois núcleos:

- **Núcleo 0:** captura de áudio (I2S), reconhecimento de voz e reprodução de som.
- **Núcleo 1:** animação dos olhos, leitura dos sensores e máquina de estados.

As tarefas se comunicam por filas do FreeRTOS: a tarefa de voz envia o comando reconhecido e a de interface reage a ele. Durante a música, o decodificador de MP3 e o modelo de voz rodam ao mesmo tempo, então é preciso testar o desempenho e ajustar a distribuição entre os núcleos se houver falhas no reconhecimento ou na animação.

## Cuidados de montagem

- Microfone longe do alto-falante e, de preferência, voltado para a frente com um furo próprio.
- VL53L0X na frente, atrás de uma pequena janela, sem tampar o feixe e longe de luz solar direta.
- PIR com a cúpula para fora, ou com a abertura da caixa coberta por plástico fino.
- Alimente tudo pela mesma fonte e mantenha o GND em comum. Se ouvir estalos, aumente o capacitor.
- Use fonte USB de 5 V / 2 A, por causa dos picos de corrente do áudio.
- Evite os pinos de boot (GPIO0, 3, 45 e 46 no S3) para periféricos.
- Para uso portátil, dá para usar uma bateria 18650 com módulo TP4056, lembrando que o consumo com Wi-Fi e áudio é alto.

## Plano de execução

| Fase | Entrega |
|---|---|
| 1. Display e relógio | Olhos animados com piscadas e humores, tela de hora, troca por código |
| 2. Sensores | PIR e VL53L0X ligados à máquina de estados, com modo sono |
| 3. Som | MAX98357A com os efeitos de cada estado |
| 4. Música | MP3 na memória flash, reprodução aleatória, parada por carinho e nome do arquivo no visor, testados por comando pela porta serial |
| 5. Voz | Captura do INMP441, detecção de volume (palmas) e depois o modelo de comandos, incluindo "música" e "parar" |

A voz fica por último, porque é a fase que mais exige ajuste. Nas fases 1 a 4, use comandos simulados pela porta serial para testar tudo sem o microfone.

## Próximos passos

- [ ] Gravar de 50 a 100 amostras de cada comando em português e treinar o modelo no Edge Impulse.
- [ ] Confirmar o pinout exato da placa ESP32-S3 adquirida.
- [ ] Implementar as fases 1 a 4 (olhos coloridos, relógio, sensores, sons e música, com fila de comandos pronta para a voz).

## Licença

Defina a licença do projeto (por exemplo, MIT) e adicione o arquivo `LICENSE` ao repositório.
