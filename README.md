# Resumen coloquio 

## Introduccion a ARM

ARM significa Advance RISC machine, es un tipo de arquitetura de procesadores.

Se dividen en distintos tipos de cortex: 

- A: Aplicacion
- R: Sistemas de tiempo real
- M: Microcontroladores
- X: para Alto rendimiento

### Principales caracteristicas

- Son procesadores de propiedad intelectual
- Es de 32 bits
- Consume menos energia que un x86
- Arquitectura RISC Load/Store
- Las interrumpciones salvan el estado automaticamente
- Mapa de memoria fijo
- Tabla de vector de direcciones
- Tabla de vector de interrupciones
- Compatible con Thumb-2
- 13 registros de propositos generales
- Stack pointer, program counter, link register
- 4GB de memoria
- 120 Mhz de velociadad maxima

## Puertos 
La placa lpc1769 posee 4 de 32 bits puertos de propositos generales (GPIO), para la correcta configuracion se detallan los siguientes registros.

- PINSEL: Debes configurar 2 bits, ya que tiene 4 funciones, por defecto esta en 00 y es la funcion de puerto GPIO, en este caso elegimos esa.

- PINMODE: Debes configurar 2 bits ya que posee 4 modos, por defecto esta en 00 y es el modo con resistencia de pull up, las demas son modo repetidor, sin resistencia, resistencia de pull down. 

En caso de haber seleccionado el modo GPIO se tiene en cuenta lo siguiente.

- FIODIR: Selecciona si es entrada o salida, 1 si es salida 0 en caso de entrada, por defecto esta en salida.

- FIOMASK: Si colocamos un bit en alto este se enmascara.

- FIOSET: Si colocamos un bit en alto le asiganamos ese valor.

- FIOCLR: Si colocamos un bit en alto limpiamos ese valor.

- FIOPIN: Sirve para leer el resultado del puerto.

### Ejemplo

```C
LPC_PINCON->PINSEL0  &= ~(0x3<<0); //Modo GPIO
LPC_PINCON->PINMODE0 &= ~(0x3<<0); //Resistencia pull up

LPC_GPIO0->FIODIR  |= (0x1<<0); //pin 1 salida
LPC_GPIO0->FIOMASK |= (0xFFFE<<0); //Enmascaramos los demas pines 
LPC_GPIO0->FIOSET  |= (0x1<<0); // ponemos en alto el pin 1 
LPC_GPIO0->FIOCLR  |= (0x1<<0); // ponemos en bajo el pin 1

uInt32_t buffer = LPC_GPIO0->FIOPIN; //Lee el puerto 0
```

## SysTick
Es un contador de sistema, por lo tanto se encuentra dedntro del core, sus principales caracteristicas son:

- Interrumpe cada intervalo de tiempo fijo
- Posee una rutina de interrupcion por NVIC
- Interrumpe cada: (valor+1)*(1/CCLK)
- Es un contador de 24 bits

Los registros que se deben tener en cuenta son:

- STCTRL: Registro de control y status del SysTick
El bit cero indica si lo habilita o deshabilita, el bit uno si interrumpe cuando llegue a cero, el bit dos elige el reloj, y el bit 16 indica que ha llegado a cero, se reinica cuando se lee

- STRELOAD: Valor con el cual se carga el SysTick

- STCURR: Valor actual del SysTick

- STCALIB: Valor calibrado de fabrica para que demore 10ms en interrumpir con un clock de 100Mhz 

Para inicializar el SysTick se debe tener en cuenta los siguientes pasos

- Deshabilitar el modulo de SysTick
- Programar el registro de recarga
- borrar el registro de valor actual
- Habilitar el modulo de Systick

### Ejemplo 

```C
SysTick->CTRL = 0; // Deshabilita el SysTick 
SysTick->LOAD = 999UL; // Seteamos para que interrumpa en 1ms
NVIC_SetPriority(SysTick_IRQn, 3); // Seteamos la prioridad
SysTick->VAL = 0;// Limpiamos el valor
SysTick->CTRL = 0x00000007; // Habilitamos el SysTick
NVIC_EnableIRQ(SysTick_IRQn); // Habilitamos la interrupción
```

## Interrupciones

Las interrupciones estan manejadas por el NVIC es decir un vector de interrupciones anidadas.

- Admite 35 interrupciones
- 32 niveles de prioridad
- Hasta 7 interrupciones anidadas
- Guarda automaticamente el contexto

Para saber donde estan ubicadas la direccion de la rutina de interrupcion hace 64 + 4 *n, donde n es el valor de identificacion de la rutina

Existen interrupciones del sistema que tienen maxima prioridad y son las exepciones

### Ejemplo de interrupcion por GPIO

Nota: se utiliza la interrupcion externa 3 para ejecutar la rutina de interrupcion y solo el puerto 0 y 2 puede interrumpir

```C
LPC_GPIO0->FIODIR |= (0x1); // P0.0 como salida
LPC_GPIO0->FIOSET |= (0x1); // P0.0 en 1 logico
LPC_GPIO2->FIODIR &= ~(0x3); // P2.0 y P2.1 como entrada

LPC_GPIOINT->IO2IntEnR |= (0x1); // Interrupcion por flanco de subida en P2.0
LPC_GPIOINT->IO2IntEnF |= (0x2); // Interrupcion por flanco de bajada en P2.1
LPC_GPIOINT->IO2IntClr |= (0x3); // Limpia las interrupciones pendientes
NVIC_EnableIRQ(EINT3_IRQn);      // Habilita las interrupciones externas
```

### Ejemlpo de interrupcion externa
```C
LPC_PINCON->PINSEL4 |= (0x5 << 22); // Existen pines exclusivos para estas interrupciones

//Configuracion de EINT1 y EINT2
LPC_SC->EXTINT   |= (0x6);  // Limpiamos las bandera de interrupcion de ambas EINT1 y EINT2.
LPC_SC->EXTMODE  |= (0x6);  // Establecemos ambas interrupciones por flanco.
LPC_SC->EXTPOLAR |= (0x4);  // Establecemos EINT1 por flanco de bajada y EINT2 por flanco de subida.
    
// Habilitamos las interrupciones por NVIC
NVIC_EnableIRQ(EINT1_IRQn);
NVIC_EnableIRQ(EINT2_IRQn);

return;
```

## Timers

La lpc posee 4 timer de 32 bits, los cuales a su ves tambien tienen un preescaler para aumentar la resolucion.

puede ser usado en modo match y en modo capture, hay 4 canales de modo match y 2 de capture.

para calcular el valor que hay que colocarle se debe poner tener en cuenta la siguiente formula: 

T_res = (Prescaler + 1)/ PCLK

### Ejemplo de timer en modo match
```C 
// Declaramos dos estructuras una para el prescaler y otra para el timer en si 

TIM_TIMERCFG_Type struct_prescaler;
TIM_MATCHCFG_Type struct_timer; 

// Configuramos el prescaler
struct_prescaler.PrescaleOption = TIM_PRESCALE_USVAL; // Configuramos el prescaler para que acepte microsegundos
struct_prescaler.PrescaleValue  = 100; // Configuramos al prescaler para que funcione a 100 microsegundos

// Configuramos el timer
struct_timer.MatchChannel = 0;// Seleccionamos el canal 0
struct_timer.IntOnMatch = ENABLE; // Habilitamos la interrupcion por match
struct_timer.StopOnMatch  = DISABLE; // No etenemos el timer al llegar al match
struct_timer.ResetOnMatch = ENABLE; // Reseteamos el timer al llegar al match
struct_timer.ExtMatchOutputType = TIM_EXTMATCH_NOTHING; // No hacemos nada con el pin de salida
struct_timer.MatchValue   = 1000;// Configuramos el match para que se produzca cada 1000 microsegundos

// Iniciamos el timer
TIM_Init(LPC_TIM0, TIM_TIMER_MODE, &struct_prescaler);
TIM_ConfigMatch(LPC_TIM0, &struct_timer);    

// Habilitamos el timer
TIM_Cmd(LPC_TIM0, ENABLE);

// Habilitamos la interrupcion del timer
NVIC_EnableIRQ(TIMER0_IRQn);
```

### Ejemlpo de timer en modo capture

```C
// Declaramos dos estructuras una para el prescaler y otra para el timer en si 

TIM_TIMERCFG_Type struct_prescaler;
TIM_CAPTURECFG_Type struct_timer; 

// Configuramos el prescaler
struct_prescaler.PrescaleOption = TIM_PRESCALE_TICKVAL; // Configuramos el prescaler para que acepte microsegundos
struct_prescaler.PrescaleValue  = 5000

struct_timer.CaptureChannel = 0;
struct_timer.FallingEdge = DISABLE;
struct_timer.IntOnCaption = ENABLE;
struct_timer.RisingEdge = ENABLE;

TIM_Init(LPC_TIM1, TIM_TIMER_MODE, (void*) &timCfg);
TIM_ConfigCapture(LPC_TIM1, &capCfg);
TIM_Cmd(LPC_TIM1, ENABLE);
``` 
## ADC

Las principales caracteristicas del ADC son las siguientes: 

- Conversor analógico-digital de aproximaciones sucesivas de 12-bits.
- Entrada multiplexada de 8 canales
- Modo Power-Down
- Rango de medición de Vrefn a Vrefp 
 (Normalmente 3[V]; no debe exceder el nivel de tensión de Vdda)
- Tasa de conversión de 12 bits de 200[KHz]
- Modo burst (ráfaga) para una o múltiples entradas
- Conversión opcional en transición de un pin de entrada o señal de match de un timer.

Para configurar cualquier periferico se deben seguir tres pasos principales

- Prenderlo
- Configurar su reloj
- Configurar el Pin

Luego se configura las demas caracteristicas propias de cada periferico en este caso el del ADC

Se puede configurar el ADC con el timer tambien en caso de que la frecuencia de muestreo no sea suficiente.

Tener en cuenta que la frecuencia de muestreo se divide por los canales que se use. 
### Ejemplo

```C
// Configuramos el pin
LPC_PINCON->PINSEL1 |= (1<<14);  //ADC0 Pin
LPC_PINCON->PINMODE1 |= (1<<15); //Neither

ADC_Init(LPC_ADC, 200000);
ADC_BurstCmd(LPC_ADC, DISABLE);
ADC_StartCmd(LPC_ADC, ADC_START_CONTINUOS);
ADC_ChannelCmd(LPC_ADC, 0, ENABLE);

ADC_IntConfig(LPC_ADC, ADC_ADINTEN0, ENABLE);
NVIC_EnableIRQ(ADC_IRQn);
``` 
## DAC

Las principales caracteristicas del DAC son: 

- Convertidor digital a analógico de 10 bits 
- Arquitectura de cadena de resistencia 
- Salida Bufereada.
- Modo de apagado 
- Velocidad seleccionable vs. Potencia
- Velocidad máxima de actualización de 1 MHz.

Para la configuracion principal se siguen los pasos de configuracion de perifericos mencionados anterior mente.

EL bit BIAS establece cual es el tiempo de establecimiento en relacion a la corriente.

### Ejemplo de configuraición con Timer

```C
// Se configura el pin aout para que sea usado por el dac 

// Se configura un timer en modo match con la frecuencia de muestreo necesaria 

DAC_Init (LPC_DAC); Solo se inicia el DAC

// Se puede habilitar o deshabilitar el bit bias segun convenga
```

### Ejemplo de configuraición con DMA

```C
// Se configura el pin aout para que sea usado por el dac 

// Se configura un DMA para que haga una trasnicion de memoria a peroferico

dacCfg.CNT_ENA = SET;
dacCfg.DMA_ENA = SET;
DAC_Init(LPC_DAC);
/*Set timeout*/
uint32_t tmp;
tmp = (PCLK_DAC_IN_MHZ * 1000000)/(frecuencia de la señal de hz * numero de muestras de la señal);
DAC_SetDMATimeOut(LPC_DAC, tmp);
DAC_ConfigDAConverterControl(LPC_DAC, &dacCfg);
```

## DMA

### Ejemplo de DMA con ADC

### Ejemplo de DMA con DAC
