# Projeto Final - Seleção da Temperatura e Temporizador

Firmware em linguagem C para o microcontrolador PIC18F4550. O sistema mede a temperatura de um sensor LM35, exibe o valor em um display LCD HD44780, controla um temporizador selecionável por dois botões e aciona um LED que representa a resistência de um forno.

**Autor:** Vinicius Gryczak Domingos - 15698383

## **Visão geral do funcionamento**

O programa consiste principalmente de um laço principal com interrupções. Antes do pressionamento de algum dos botões, o display LCD mostra 00s na primeira linha e OFF na segunda. Uma vez que um botão é pressionado, inicia-se uma contagem regressiva, e o microcontrolador lê o sinal vindo de um potenciômetro e o mostra no display como temperatura. Uma vez encerrada a contagem, o sistema é desligado. A diferença entre os botões está no tempo de contagem, que pode ser de 10 ou 60 segundos.

## **Hardware / Esquemático**

O circuito é composto pelo PIC18F4550, um display LCD HD44780 em modo 4 bits, dois botões, um potenciômetro, uma fonte externa de 1 V para a referência do ADC e um LED.

### **Mapeamento de pinos**

<img width="1191" height="635" alt="image" src="https://github.com/user-attachments/assets/43a72cca-b83d-48fb-9bcd-01e5b5bf974e" />

## **Estrutura do firmware**

### **1\. Configuração do LCD**

O _sbit_ no início do arquivo associa os pinos lógicos do LCD às portas físicas do PORTB, e define os respectivos registradores de direção. O LCD é controlado em modo de 4 bits.

### **2\. Variáveis globais**

char countdown_60 = 60; // contador do modo de 60 s

char countdown_10 = 10; // contador do modo de 10 s

unsigned short tmr1_ticks = 0; // acumulador de "ticks" do Timer1

char current_mode = 0; // 1 = modo 60 s, 2 = modo 10 s

bit system_active; // 1 = forno/sistema ligado, 0 = desligado

bit flag_btn1, flag_btn2; // controle de borda dos botões

_system_active_ é a flag que define se o sistema está sendo executado ou em repouso. As flags _flag_btn1_ e _flag_btn2_ evitam que um botão mantido pressionado dispare a ação várias vezes.

### **3\. Configuração e leitura do ADC**

A configuração é feita em main():

ADCON1 = 0x1B; // AN0-AN3 analógicos; Vref+ externo (AN3); Vref- = Vss

ADCON2 = 0b10001010; // resultado justificado à direita; Tacq = 2 TAD; clock = Fosc/32

ADCON0 = 0b00000001; // canal AN0 selecionado; ADC ligado

CMCON = 7; // comparadores desligados

Utiliza-se uma referência de 1 V no Vref+ (AN3). Como o LM35 fornece 10 mV/°C, na faixa de 0 a 100 °C sua saída varia de 0 a 1 V.

A leitura é feita manualmente:

unsigned int read_ADC_manual() {

ADCON0.GO_DONE = 1; // dispara a conversão

while(ADCON0.GO_DONE == 1); // aguarda a conversão terminar

return (ADRESH << 8) | ADRESL;

}

**Conversão para temperatura:**

temp_raw = ((unsigned long)adc_value \* 1000) / 1023;

Como _adc_value_ vai de 0 a 1023 representando 0 a 100 °C, multiplicar por 1000 e dividir por 1023 produz a temperatura multiplicada por 10. Isso permite obter uma casa decimal sem usar o ponto flutuante.

### **4\. Timers**

Considerando o clock de 8 MHz do kit:

- **Timer0**: Fazendo com que _TMR0H:TMR0L = 0xE17C_, temos uma execução a cada 1 segundo. É usado no modo de 60 s, diminuindo por 1 _countdown_60_ a cada interrupção.

- **Timer1**: Fazendo com que _TMR1H:TMR1L = 0x0BDC_, temos uma execução a cada 250 ms. A ISR acumula as execuções em _tmr1_ticks_ e decrementa _countdown_10_ a cada 4 ticks (1 segundo).

Os timers iniciam desligados e apenas ligam após um dos botões serem pressionados.

### **5\. Interrupções (ISR)**

A rotina _interrupt()_:

- **Interrupt-on-change do PORTB:** Examina o estado das portas, aplica _Delay_ms(20)_ como anti-bouncing e verifica os botões em RB6 e RB7. Uma vez detectada a borda de subida do Botão 1, para o Timer1, reinicia _countdown_60 = 60_, define _current_mode = 1_, liga _system_active_ e inicia o Timer0. O Botão 2 é análogo, mas para o Timer0.

- **Timer0:** Recarrega o Timer0, decrementa _countdown_60_, e desliga o Timer0 e zera _system_active_ ao chegar a 0.

- **Timer1:** Recarrega o Timer1, incrementa _tmr1_ticks_ e decrementa _countdown_10_ cada um segundo. Ao chegar a zero, desliga o Timer1 e zera _system_active_.

As interrupções são habilitadas em _main()_ com _RBIE_, _TMR0IE_, _TMR1IE_, além de _PEIE_ e _GIE_.

### **6\. Loop principal**

A cada ciclo o programa seleciona o tempo a exibir: _time_val = (current_mode == 1) ? countdown_60 : countdown_10;_. Se o _system_active_ estiver ligado, ele olha o ADC, calcula _temp_raw_, decide o estado do LED, e monta as duas linhas do display. O LED apaga e mostra 00s e OFF no display se o sistema estiver desligado.

### **7\. Formatação do display:**

Temos buffers _text_line1\[17\]_ e _text_line2\[17\]_, inicializados com espaços em branco e terminador nulo. Cada caractere é convertido de dígito para ASCII somando '0':

text_line2\[0\] = (temp_raw / 1000) + '0'; // centena (0 ou 1, para 100.0)

text_line2\[1\] = ((temp_raw / 100) % 10) + '0'; // dezena

text_line2\[2\] = ((temp_raw / 10) % 10) + '0'; // unidade

text_line2\[3\] = '.';

text_line2\[4\] = (temp_raw % 10) + '0'; // casa decimal

text_line2\[5\] = 223; // símbolo de grau (°)

text_line2\[6\] = 'C';

O código 223 corresponde ao símbolo de grau (°) na tabela de caracteres do HD44780. O zero à esquerda é suprimido para temperaturas abaixo de 100 °C.

O _Lcd_Out_ é chamado uma única vez por ciclo.

### **8\. Lógica do LED (resistência do forno)**

if (temp_raw > 500) { // > 50,0 °C

LED_FORNO = 1;

} else {

LED_FORNO = 0;

}

Como _temp_raw_ é a temperatura × 10, o 50 °C corresponde a _temp_raw = 500_. O LED, ligado ao pino RD0 através de um resistor de 220 Ω, acende sempre que a temperatura ultrapassa 50 °C e apaga caso contrário.

**Funcionamento do Sistema:**

- **Estado inicial (botões não pressionados):**

<img width="1116" height="640" alt="image" src="https://github.com/user-attachments/assets/18d02ddb-af80-4265-b9f9-9ab693fb2b90" />


- **Botão 1 pressionado (60s) e temperatura mínima (0 °C):**

<img width="1107" height="607" alt="image" src="https://github.com/user-attachments/assets/2fcc4032-0243-4607-9bcb-be7b38cda366" />


Note-se que o LED está desligado, já que a temperatura é menor a 50 °C.

- **Botão 2 pressionado (10s) e temperatura máxima (100 °C):**

<img width="1136" height="603" alt="image" src="https://github.com/user-attachments/assets/a51f98ea-cc6f-4709-b733-2fa02821f0bb" />


Note-se também que o LED está ligado, já que a temperatura é maior a 50 °C.

**Registro de Compilação do Código**

<img width="1452" height="180" alt="Screenshot 2026-06-22 213738" src="https://github.com/user-attachments/assets/fc051d14-4233-4995-bdb2-36b89b3e6895" />
