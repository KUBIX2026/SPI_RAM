# SPIRAM_CTRL

Este espacio está reservado para la especificación de la información relevante del funcionamiento de la memoria RAM mediante protocolo `SPI`. 

![RAM Type Differentiation](../Images/RAM%20type%20description.png)

Con base en esta imagen se puede determinar el tipo de módulo con el que se va a trabajar, y en consecuencia, empezar la creación de su diagrama.
En este sistema la memoria `RAM` va a trabajar como almacenador de información referente a variables de ejecución de algún programa. 

# Consideraciones. 

1. La información relevante del protocolo se encuentra en [Protocolo SPI](../spiram_ctrl.v/SPI%20Protocol.md)
2. La memoria `RAM` es un dispositivo externo y no es parte de la `FPGA`, sin embargo, el controlador que se desarrolla sí está en la `FPGA`. Se hace necesaria la aclaración para denotar que existen dos dominios; `BUS LOCAL` (lado `CPU`) que trabaja con el reloj del sistema generado por el oscilador de cristal y `BUS EXTERNO` (lado periféricos) que trabaja con un reloj independiente comandado por el protocolo -`SPI` en nuestro caso- con base en la capacidad del periférico en cuestión.
3. El dispositivo que se va a usar consiste en una `RAM` `QUAD-SPI` (para más información respecto a cómo funciona esto remitirse a [Protocolo SPI](/SPI%20Protocol.md). Consiste en un chip [`HCS138`](../Datasheets/SN74HCS138.pdf) (05EG3/M8K3) y 4 memorias [`APS6404L-3SQ`](../Datasheets/APS6404L-3SQR.pdf).

## Especificaciones Técnicas

Un vistazo rápido de los datos presentes en los datasheets pero relevantes para el controlador se muestran en las tablas. 

### Componentes de Memoria

| Componente | Parámetro Técnico | Valor / Especificación | Relevancia para el Controlador Verilog / FPGA |
| :--- | :--- | :--- | :--- |
| **Memoria PSRAM**<br>(`APS6404L-3SQR`)<br>*(4 integrados)* | **Organización / Capacidad** | $64\text{ Mbit}$ ($8\text{ MB}$) por chip.<br>Total conjunto ($4 \times 8\text{ MB}$): **$32\text{ MB}$** ($256\text{ Mbit}$).  | Determina el tamaño total del mapa de memoria ($A[24:0]$) que el decodificador de la CPU debe direccionar. |
| | **Tensión de Trabajo ($V_{DD}$)** | $2.7\text{ V}$ a $3.6\text{ V}$ ($3.3\text{ V}$ nominal). | Define el nivel lógico de los I/O de la FPGA (`LVCMOS33`). Si la FPGA usa $1.8\text{ V}$ o $2.5\text{ V}$, requiere adaptación de nivel (*Level Shifter*). |
| | **Frecuencia de Reloj ($SCLK$)** | **Hasta $84\text{ MHz}$** en modo *Linear Burst* (cruza fronteras de página de $1\text{ KB}$).<br>**Hasta $133\text{ MHz}$** en *32-Byte Wrapped Burst* (a $3.0\text{ V}$). | El divisor de reloj o *Clock Enable* en Verilog debe garantizar que $SCLK \le 84\text{ MHz}$ para operaciones de ráfaga lineal continua.|
| | **Protocolo e Interfaz** | SPI ($1\times \text{I/O}$) / QPI Quad-SPI ($4\times \text{I/O}$, $SIO[3:0]$). | La FSM en Verilog debe manejar transiciones entre $1\text{ bit/ciclo}$ (comando) y $4\text{ bits/ciclo}$ (dirección/datos en QSPI). |
| | **Gestión de Refresco** | Automático e Interno (*Self-Managed PSRAM*).  | **Ventaja crítica:** No se necesita implementar un controlador DRAM con ciclos de refresco periódicos en Verilog; la memoria se comporta como SRAM estática.  |
| | **Límite $t_{CEM}$ (Burst Duration)** | Máximo $\approx 4\mu\text{s}$ a $8\mu\text{s}$ con `CE#` activado continuo.  | La FSM debe desmarcar `CE#` (ponerlo en alto) tras completar un bloque de lectura/escritura para que la PSRAM ejecute su autorrefresco interno. |
| **Decodificador**<br>(`SN74HCS138`) | **Función** | Decodificador / Demultiplexor de 3 a 8 líneas con entradas Schmitt-Trigger. | Actúa como selector de chip (*Chip Select*) enviando la señal `CE#` activa en bajo ($0\text{ V}$) a cada uno de los 4 chips de memoria según las líneas de selección. |
| | **Tensión de Trabajo ($V_{CC}$)** | $2.0\text{ V}$ a $6.0\text{ V}$ ($3.3\text{ V}$ o $5.0\text{ V}$ típicamente). | Garantiza compatibilidad de voltaje directo con los niveles $3.3\text{ V}$ provenientes de los pines de la FPGA y las memorias. |
| | **Tiempo de Propagación ($t_{pd}$)** | $\approx 9\text{ ns}$ a $15\text{ ns}$ (dependiendo de $V_{CC}$). | **Latencia del controlador:** Se debeañadir esta demora en el análisis de tiempos (*Setup/Hold*) de la señal `CE#` que sale de la FPGA antes de iniciar la primera instrucción $SCLK$. |
| | **Entradas Schmitt-Trigger** | Tolerancia e inmunidad al ruido en los flancos de conmutación. | Elimina rebotes o ruidos indeseados en las líneas de control que selecciona la CPU antes de llegar a la PSRAM. |

---

## Mapeo de Selección de Chip (Decodificador `SN74HCS138`)

| Selección Verilog (`SEL[1:0]`) | Salida Activa `SN74HCS138` | Memoria Habilitada | Rango de Direcciones Hexadecimal ($32\text{ MB}$ Total) |
| :---: | :---: | :---: | :---: |
| `2'b00` | $Y_0$ (`0V` / Bajo) | `APS6404L` Chip 0 ($8\text{ MB}$) | `0x0000_0000` — `0x007F_FFFF`  |
| `2'b01` | $Y_1$ (`0V` / Bajo) | `APS6404L` Chip 1 ($8\text{ MB}$) | `0x0080_0000` — `0x00FF_FFFF`  |
| `2'b10` | $Y_2$ (`0V` / Bajo) | `APS6404L` Chip 2 ($8\text{ MB}$) | `0x0100_0000` — `0x017F_FFFF`  |
| `2'b11` | $Y_3$ (`0V` / Bajo) | `APS6404L` Chip 3 ($8\text{ MB}$) | `0x0180_0000` — `0x01FF_FFFF`  |

## Mapeo físico del periférico

-Estamos trabajando en ello, el mapa se actualizará tan pronto como se confirmen las conexiones por ingeniería inversa-


# Diagrama de flujo 

Con base en la información presente en [Protocolo SPI](/SPI#20Protocol.md) y referenciando el hardware disponible se puede establecer el flujo del proceso que debe realizar el controlador durante la comunicación con la `CPU` y la `RAM`.

```mermaid
flowchart TD
    Start([Inicio / Reset del Sistema]) --> InitMode[Configurar controlador en modo 1-bit SPI]
    InitMode --> SendEnableQPI[Transmitir comando de habilitación Quad-SPI 0x35 por SIO0]
    SendEnableQPI --> SetQPIFlag[Establecer bus interno en modo 4-bits / QPI]
    SetQPIFlag --> Idle[Estado IDLE: Esperar solicitud de la CPU<br>Líneas CE# en alto / SCLK = 0]

    Idle --> CheckReq{¿CPU solicita acceso?<br>req == 1}
    CheckReq -- No --> Idle
    CheckReq -- Sí --> LatchBus[Capturar bus de la CPU:<br>Dirección A de 25 bits, Datos y WE]

    LatchBus --> DecodeChip[Decodificar bits superiores A24-A23<br>Enviar SEL a entradas de SN74HCS138]
    DecodeChip --> AssertCE[El SN74HCS138 baja la línea CS# a 0 V<br>de la memoria seleccionada]
    AssertCE --> SendOpcode[Enviar OpCode de operación por bus Quad SIO<br>0x03 Lectura / 0x02 Escritura]
    SendOpcode --> SendAddr[Enviar dirección relativa de 23 bits A22-A0<br>por el bus Quad SIO]

    SendAddr --> CheckWE{¿Tipo de operación?}

    CheckWE -- Escritura WE = 1 --> WriteData[Transferir n-bytes de datos<br>desde CPU hacia PSRAM por bus SIO]
    WriteData --> DeassertCE[Levantar línea CS# a 1 V<br>Finalizar transacción en memoria]

    CheckWE -- Lectura WE = 0 --> DummyCycles[Generar ciclos de reloj Dummy<br>Esperar recuperación analógica de PSRAM]
    DummyCycles --> ReadData[Muestrear datos desde PSRAM por bus SIO<br>en cada flanco ascendente de SCLK]
    ReadData --> SendToCPU[Cargar dato en bus de entrada de la CPU]
    SendToCPU --> DeassertCE

    DeassertCE --> SendAck[Generar señal ACK / Ready a la CPU]
    SendAck --> CheckTimer{¿Límite t_CEM excedido?<br>Burst mayor a 4 us}
    CheckTimer -- Sí --> PauseCE[Mantener CS# en 1 V durante 50 ns<br>Permitir autorrefresco interno de PSRAM]
    CheckTimer -- No --> Idle
    PauseCE --> Idle
```

1. **Óvalos (`Inicio / Reset`):** Indican el arranque del hardware desde que la FPGA recibe alimentación o una señal de reset.
2. **Rectángulos (`Procesos`):**
   * **`Configurar / Transmitir QPI`:** Ocurren una sola vez al encender el sistema para pasar las memorias a modo de 4 bits.
   * **`Decodificar bits A[24:23]`:** Transforma la dirección de la CPU en las señales `SEL[1:0]` que van hacia el `SN74HCS138`.
   * **`El SN74HCS138 baja la línea CS#`:** Habilita el chip de memoria correcto *antes* de enviar la dirección y los datos.
3. **Rombos (`Decisiones`):**
   * **`¿CPU solicita acceso?`:** Evalúa si la CPU colocó una dirección válida asignada a la RAM.
   * **`¿Tipo de operación?`:** Divide el camino entre el ciclo de escritura directa o el ciclo de lectura (que requiere *Dummy Cycles* de espera antes de capturar el dato).
   * **`¿Límite t_CEM excedido?`:** Verifica si la transferencia fue demasiado larga para liberar `CE#` y permitir que la memoria se autorrefresque.

# Diagrama de bloques [IA sugerido. Sujeto a revisión]

Con base en la información presentada en el diagrama de flujo previo, se hace el diagrama de bloques correspondiente a dicho sistema. 

```mermaid
graph LR
    subgraph CPU [Procesador RISC]
        ADDR_BUS[Bus de Direccion 32 bits]
        DATA_OUT[Bus de Datos Salida]
        DATA_IN[Bus de Datos Entrada]
        CTRL_BUS[Senales de Control WE y STB]
    end

    subgraph DECODER_SYS [Decodificador del Sistema]
        ADDR_DEC[Address Decoder CPU]
    end

    subgraph CTRL_MODULE [Controlador QSPI Verilog en FPGA]
        FSM[FSM Control y Clock Enable]
        DATA_REG[Shift Registers y Tri-State]
        ADDR_SPLIT[Divisor de Direcciones A24-A0]
    end

    subgraph HARDWARE_EXT [Componentes Externos en Tarjeta]
        DECODER_138[SN74HCS138 Decodificador 3 a 8]
        RAM0[APS6404L Chip 0 de 8MB]
        RAM1[APS6404L Chip 1 de 8MB]
        RAM2[APS6404L Chip 2 de 8MB]
        RAM3[APS6404L Chip 3 de 8MB]
    end

    %% Conexiones CPU hacia Decodificador de Direcciones
    ADDR_BUS --> ADDR_DEC
    ADDR_DEC -- Seleccion de Controlador REQ --> FSM

    %% Conexiones CPU hacia Controlador
    ADDR_BUS --> ADDR_SPLIT
    DATA_OUT --> DATA_REG
    DATA_REG --> DATA_IN
    CTRL_BUS --> FSM

    %% Conexiones internas y hacia externos desde el Controlador
    ADDR_SPLIT -- Seleccion de Chip SEL 2 bits --> DECODER_138
    FSM -- Reloj SCLK --> RAM0
    FSM -- Reloj SCLK --> RAM1
    FSM -- Reloj SCLK --> RAM2
    FSM -- Reloj SCLK --> RAM3

    DATA_REG <== Bus Quad SIO 4 bits ==> RAM0
    DATA_REG <== Bus Quad SIO 4 bits ==> RAM1
    DATA_REG <== Bus Quad SIO 4 bits ==> RAM2
    DATA_REG <== Bus Quad SIO 4 bits ==> RAM3

    %% Habilitaciones desde Decodificador SN74HCS138 a cada Memoria
    DECODER_138 -- Habilitacion CE0# --> RAM0
    DECODER_138 -- Habilitacion CE1# --> RAM1
    DECODER_138 -- Habilitacion CE2# --> RAM2
    DECODER_138 -- Habilitacion CE3# --> RAM3
```
