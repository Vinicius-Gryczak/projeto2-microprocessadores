// Projeto Final - Seleção da Temperatura e Temporizador

// Vinicius Gryczak Domingos - 15698383

// Configuração dos Pinos do LCD no PORTB

sbit LCD_RS at LATB4_bit;

sbit LCD_EN at LATB5_bit;

sbit LCD_D4 at LATB0_bit;

sbit LCD_D5 at LATB1_bit;

sbit LCD_D6 at LATB2_bit;

sbit LCD_D7 at LATB3_bit;

sbit LCD_RS_Direction at TRISB4_bit;

sbit LCD_EN_Direction at TRISB5_bit;

sbit LCD_D4_Direction at TRISB0_bit;

sbit LCD_D5_Direction at TRISB1_bit;

sbit LCD_D6_Direction at TRISB2_bit;

sbit LCD_D7_Direction at TRISB3_bit;

// Pino do LED (Resistência do forno)

sbit LED_FORNO at LATD0_bit;

sbit LED_FORNO_Dir at TRISD0_bit;

// Variáveis Globais

char countdown_60 = 60;

char countdown_10 = 10;

unsigned short tmr1_ticks = 0;

char current_mode = 0;

// Flag para controle do forno (1 = Ligado, 0 = Desligado)

bit system_active;

// Flags auxiliares para o tratamento do Interrupt-on-Change

bit flag_btn1;

bit flag_btn2;

// Rotina de Serviço de Interrupção (ISR)

void interrupt() {

    // Interrupção on Change PORTB

    if (RBIF_bit) {

        char portb_val = PORTB;

        Delay_ms(20);            // Anti-bouncing

        // Verifica Botão 1 (RB6)

        if (portb_val.B6 == 1 && flag_btn1 == 0) {

            flag_btn1 = 1;

            T1CON.TMR1ON = 0;     // Para o Timer 1

            countdown_60 = 60;

            current_mode = 1;

            system_active = 1;    // Liga o sistema

            TMR0H = 0xE1;         // Recarrega Timer 0 para 1 segundo

            TMR0L = 0x7C;

            T0CON.TMR0ON = 1;     // Inicia Timer 0

        } else if (portb_val.B6 == 0) {

            flag_btn1 = 0;

        }

        // Verifica Botão 2 (RB7)

        if (portb_val.B7 == 1 && flag_btn2 == 0) {

            flag_btn2 = 1;

            T0CON.TMR0ON = 0;     // Para o Timer 0

            countdown_10 = 10;

            tmr1_ticks = 0;

            current_mode = 2;

            system_active = 1;    // Liga o sistema

            TMR1H = 0x0B;         // Recarrega Timer 1

            TMR1L = 0xDC;

            T1CON.TMR1ON = 1;     // Inicia Timer 1

        } else if (portb_val.B7 == 0) {

            flag_btn2 = 0;

        }

        RBIF_bit = 0; // Limpa a flag de interrupção

    }

    // Interrupção do Timer 0 (Contagem 60s)

    else if (TMR0IF_bit) {

        TMR0IF_bit = 0;

        TMR0H = 0xE1;

        TMR0L = 0x7C;

        if (countdown_60 > 0) {

            countdown_60--;

        }

        if (countdown_60 == 0) {

            T0CON.TMR0ON = 0;

            system_active = 0;    // Desliga o forno ao final do tempo

        }

    }

    // Interrupção do Timer 1 (Contagem 10s)

    else if (TMR1IF_bit) {

        TMR1IF_bit = 0;

        TMR1H = 0x0B;

        TMR1L = 0xDC;

        tmr1_ticks++;

        if (tmr1_ticks >= 4) {    // 4 ticks de 250ms = 1 segundo

            tmr1_ticks = 0;

            if (countdown_10 > 0) {

                countdown_10--;

            }

            if (countdown_10 == 0) {

                T1CON.TMR1ON = 0;

                system_active = 0; // Desliga o forno ao final do tempo

            }

        }

    }

}


// Função Manual de Leitura do ADC

unsigned int read_ADC_manual() {

    ADCON0.GO_DONE = 1;         // Inicia a conversão

    while(ADCON0.GO_DONE == 1); // Aguarda o fim da conversão

    return (ADRESH << 8) | ADRESL; // Retorna o valor de 10 bits

}

// Função Principal
void main() {
    // Variáveis locais para matemática
    unsigned int adc_value;
    unsigned long temp_raw;
    char time_val;

    // Buffers de tela
    char text_line1[17] = "                ";
    char text_line2[17] = "                ";

    // 1. Configurações de Pinos Analógicos/Digitais e ADC
    ADCON1 = 0x1B;
    ADCON2 = 0b10001010;
    ADCON0 = 0b00000001;

    CMCON = 7;

    // 2. Direção de Portas
    TRISA.B0 = 1;
    TRISA.B3 = 1;

    TRISB6_bit = 1;
    TRISB7_bit = 1;

    LED_FORNO_Dir = 0;
    LED_FORNO = 0;

    // 3. Configuração de Interrupções
    INTCON.RBIE = 1;
    INTCON.TMR0IE = 1;
    PIE1.TMR1IE = 1;

    INTCON.PEIE = 1;
    INTCON.GIE = 1;

    // 4. Configuração dos Timers
    T0CON = 0x07;
    T1CON = 0xB0;

    // 5. Inicializações
    flag_btn1 = 0;
    flag_btn2 = 0;
    system_active = 0;

    Lcd_Init();
    Lcd_Cmd(_LCD_CLEAR);
    Lcd_Cmd(_LCD_CURSOR_OFF);

    // 6. Loop Principal
    while(1) {
        time_val = (current_mode == 1) ? countdown_60 : countdown_10;

        if (system_active) {
            // Leitura e cálculo do ADC
            adc_value = read_ADC_manual();
            temp_raw = ((unsigned long)adc_value * 1000) / 1023;

            // Lógica do LED
            if (temp_raw > 500) {
                LED_FORNO = 1;
            } else {
                LED_FORNO = 0;
            }

            // Linha 1
            text_line1[0] = (time_val / 10) + '0';
            text_line1[1] = (time_val % 10) + '0';
            text_line1[2] = 's';
            text_line1[3] = ' ';

            // Linha 2
            text_line2[0] = (temp_raw / 1000) + '0';
            text_line2[1] = ((temp_raw / 100) % 10) + '0';
            text_line2[2] = ((temp_raw / 10) % 10) + '0';
            text_line2[3] = '.';
            text_line2[4] = (temp_raw % 10) + '0';
            text_line2[5] = 223; // Símbolo de grau
            text_line2[6] = 'C';
            text_line2[7] = ' '; // Apaga resíduos

            // Remove o zero à esquerda da temperatura
            if (text_line2[0] == '0') text_line2[0] = ' ';

        } else {
            // Quando desligado
            LED_FORNO = 0;

            // Reseta a Linha 1
            text_line1[0] = '0';
            text_line1[1] = '0';
            text_line1[2] = 's';

            // Reseta a Linha 2 substituindo os números pela palavra OFF
            text_line2[0] = 'O';
            text_line2[1] = 'F';
            text_line2[2] = 'F';
            text_line2[3] = ' ';
            text_line2[4] = ' ';
            text_line2[5] = ' ';
            text_line2[6] = ' ';
        }

        // Envia as linhas completas
        Lcd_Out(1, 1, text_line1);
        Lcd_Out(2, 1, text_line2);

        Delay_ms(200);
    }
}
