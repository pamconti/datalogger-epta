# Datalogger Ambiental e Inercial com STM32F411

Projeto desenvolvido para o **Desafio do Processo Seletivo Payload** da **EPTA – Equipe de Propulsão e Tecnologia Aeroespacial (UFU)**.

Datalogger para payload de foguete que registra, durante todo o voo, as condições do ambiente (temperatura, umidade e pressão) e o movimento do veículo (aceleração e velocidade angular), gravando os dados em um cartão microSD no formato CSV.

> **Status:** em desenvolvimento.
> - [x] Configuração do microcontrolador (STM32CubeMX): pinagem, clock, periféricos, FatFs e RTC
> - [x] Projeto base compilando no STM32CubeIDE (0 erros, 0 warnings)
> - [ ] Esquemático (KiCad)
> - [ ] Layout da PCB (KiCad)
> - [ ] Firmware: drivers, gravação em CSV e tratamento de erros
> - [ ] Validação em Black Pill

---

## Sumário

1. [Visão geral do hardware](#visão-geral-do-hardware)
2. [Pinagem](#pinagem)
3. [Configuração de clock](#configuração-de-clock)
4. [Estrutura do repositório](#estrutura-do-repositório)
5. [Como compilar e gravar](#como-compilar-e-gravar)
6. [Validação](#validação)
7. [Decisões de projeto](#decisões-de-projeto)
8. [Referências](#referências)

---

## Visão geral do hardware

| Bloco | Componente | Interface |
|---|---|---|
| Microcontrolador | STM32F411CEU6 (UFQFPN48) | — |
| Temperatura / umidade / pressão | BME280 | I²C |
| IMU (acelerômetro + giroscópio) | LSM6DSO | SPI + pino de interrupção (INT1) |
| Armazenamento | Slot microSD | SDIO 4 bits |
| Alimentação | USB-C + LDO 3,3 V | — |
| Gravação e depuração | Conector SWD | SWDIO / SWCLK |
| Interface com o usuário | LED de status e botão | GPIO |

---

## Pinagem

Pinagem definida e validada no STM32CubeMX (arquivo [`firmware/firmware.ioc`](firmware/firmware.ioc)). A pinagem de LED, botão, cristais, USB e SWD foi escolhida para coincidir com a da placa **WeAct Black Pill (STM32F411CEU6)**, permitindo que o mesmo firmware rode na placa projetada e na Black Pill usada para validação.

| Função | Sinal | Pino |
|---|---|---|
| **SDIO (microSD)** | CK | PB15 |
| | CMD | PA6 |
| | D0 | PB7 |
| | D1 | PA8 |
| | D2 | PA9 |
| | D3 | PB5 |
| | Detecção de cartão (`SD_DETECT`) | PB1 (entrada com pull-up) |
| **SPI1 (IMU)** | SCK | PA5 |
| | MOSI | PA7 |
| | MISO | PB4 |
| | CS (`IMU_CS`) | PA4 |
| | INT1 (`IMU_INT1`) | PB0 (EXTI0, borda de subida) |
| **I2C1 (BME280)** | SCL | PB8 |
| | SDA | PB9 |
| **USB (CDC)** | D− | PA11 |
| | D+ | PA12 |
| **USART2 (debug)** | TX | PA2 |
| | RX | PA3 |
| **SWD** | SWDIO | PA13 |
| | SWCLK | PA14 |
| **Cristais** | HSE 25 MHz | PH0 / PH1 |
| | LSE 32,768 kHz | PC14 / PC15 |
| **Interface** | LED de status (`LED_STATUS`) | PC13 |
| | Botão (`BTN_USER`) | PA0 (entrada com pull-up) |

---

## Configuração de clock

| Parâmetro | Valor |
|---|---|
| Fonte do PLL | HSE (cristal de 25 MHz) |
| PLL | M = 25, N = 192, P = 2, Q = 4 |
| SYSCLK / HCLK | 96 MHz |
| APB1 (PCLK1) | 48 MHz |
| APB2 (PCLK2) | 96 MHz |
| Clock de 48 MHz (USB e SDIO) | 48 MHz |
| RTC | LSE (cristal de 32,768 kHz) |

O HCLK foi definido em 96 MHz, e não no máximo de 100 MHz, porque é a frequência que permite gerar exatamente 48 MHz para o USB e o SDIO a partir do mesmo PLL.

**Periféricos auxiliares:**
- **TIM2:** prescaler 95 → contagem de 1 µs; período de 2³² − 1 (estoura a cada ~71 min). Usado como base de tempo de alta resolução para os timestamps.
- **RTC:** calendário alimentado pelo LSE, para timestamps absolutos.
- **FatFs:** modo SD Card sobre SDIO com DMA (RX e TX), nomes longos habilitados (LFN), setor de 512 bytes.

---

## Estrutura do repositório

```
├── hardware/   Projeto do KiCad, Gerbers, BOM, arquivo de posicionamento e PDF do esquemático
├── firmware/   Projeto do STM32CubeIDE (gerado pelo STM32CubeMX)
├── docs/       Decisões de projeto, diagrama de blocos, testes e datasheets
├── scripts/    Scripts em Python para análise dos arquivos CSV
└── README.md
```

---

## Como compilar e gravar

### Requisitos
- [STM32CubeMX](https://www.st.com/stm32cubemx) 6.x (apenas para alterar a configuração do microcontrolador)
- [STM32CubeIDE](https://www.st.com/stm32cubeide)
- Gravador ST-Link V2

### Compilar
1. Clone este repositório.
2. No STM32CubeIDE: **File → Import → STM32 → STM32CubeMX/STM32CubeIDE Project**.
3. Em *Import source*, selecione a pasta `firmware/` e clique em **Finish**.
4. Selecione o projeto e compile com **Ctrl+B**.

### Gravar
1. Conecte o ST-Link ao conector SWD (SWDIO, SWCLK, GND, 3V3 e NRST).
2. No STM32CubeIDE, clique em **Run** (gravar) ou **Debug** (gravar e depurar).

### Alterar a configuração do microcontrolador
Abra `firmware/firmware.ioc` no STM32CubeMX, faça a alteração e clique em **GENERATE CODE**. O código escrito dentro dos blocos `USER CODE BEGIN` / `USER CODE END` é preservado.

---

## Validação

Hardware disponível para validação: **WeAct Black Pill (STM32F411CEU6)** e **ST-Link V2**. Os módulos dos sensores e do microSD não estavam disponíveis.

| Etapa | Método | Resultado |
|---|---|---|
| Compilação | STM32CubeIDE | ✅ 0 erros, 0 warnings |
| Gravação via SWD | Black Pill + ST-Link | ⏳ pendente |
| Blink do LED de status | Black Pill | ⏳ pendente |
| Clock (HSE + PLL a 96 MHz) | Black Pill | ⏳ pendente |
| Tratamento de erro: falha de sensor | Black Pill sem sensores | ⏳ pendente |
| Tratamento de erro: cartão ausente | Black Pill sem cartão | ⏳ pendente |
| Leitura do BME280 | — | ⚠️ sem módulo disponível |
| WHO_AM_I do IMU | — | ⚠️ sem módulo disponível |
| Gravação de CSV no SD | — | ⚠️ sem módulo disponível |

---

## Decisões de projeto

As justificativas de cada decisão, com as referências usadas, estão em [`docs/decisoes-de-projeto.md`](docs/decisoes-de-projeto.md).

---

## Referências

- STMicroelectronics. *STM32F411xC/E Datasheet* (DS10314).
- STMicroelectronics. *AN4488 – Getting started with STM32F4xxxx MCU hardware development*.
- STMicroelectronics. *UM1718 – STM32CubeMX for STM32 configuration and initialization C code generation*.
- STMicroelectronics. *LSM6DSO Datasheet*.
- Bosch Sensortec. *BME280 Datasheet*.
- WeAct Studio. *MiniF4 (Black Pill) – STM32F4x1*. https://github.com/WeActTC/MiniF4-STM32F4x1
- EPTA – UFU. *Desafio de Processo Seletivo Payload*, 11 de outubro de 2026.
