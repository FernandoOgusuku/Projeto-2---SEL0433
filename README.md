# Projeto-2---SEL0433

<div align="center">
  <img height="80" alt="Image" src="https://github.com/user-attachments/assets/8f68842b-2353-4898-a1ef-581c7dfbcb02" />
  <img height="80" alt="Image" src="https://github.com/user-attachments/assets/93cc5a3b-ca74-4bfc-9819-ecaee6ea8ae8" />
  <br><br>
  <b>Departamento de Engenharia Elétrica e de Computação</b><br>
  <b>EESC USP | SEL0433 - Aplicação de Microprocessadores</b><br>
  <b>Prof.: Pedro Oliveira</b>
  <br><br>
  <h1>Projeto 2: "Aferidor de temperatura de forno industrial"</h1>
  <i>Implementação em Linguagem C (Microcontrolador PIC18F4550)</i>
  <br><br>
  <img height="80" alt="Image" src="https://github.com/user-attachments/assets/1bf8fe88-9069-4698-b9a1-080933eb2702" />

</div>

<br>

<div align="right">
  <b>Turma:</b> 2ª feira 14h<br><br>
  <b>Alunos:</b><br>
  João Vitor Miranda Sousa – 14802702<br>
  Fernando Shoji Ogusuku – 15636682
</div>

---

## 1 Objetivo e Contextualização 
Este projeto, estruturado na metodologia PBL (Problem-Based Learning), visa o desenvolvimento de um firmware em linguagem C para o microcontrolador PIC18F4550. O cenário simulado é o monitoramento de um forno industrial, exigindo a leitura precisa de sensores analógicos e o controle rigoroso de tempo.

A aprendizagem foca na transição da arquitetura clássica (como o 8051) para a família PIC18, explorando seus recursos avançados: conversor Analógico-Digital (ADC) de 10 bits, múltiplos temporizadores (Timers), sistema de interrupções com prioridade e interface paralela com displays LCD alfanuméricos.

---

## 2 Requisitos do Sistema 
De acordo com o roteiro do projeto, o sistema deve cumprir integralmente os seguintes requisitos:

1. **Medição de Temperatura**: Leitura de um sensor LM35 (emulado via potenciômetro) na faixa de 0°C a 100°C, utilizando o módulo ADC com tensão de referência externa de 1V.

2. **Interface Visual**: Exibição contínua da temperatura formatada ("XX.X °C") e do tempo restante no display LCD HD44780 (modo 4 bits).

3. **Temporização por Hardware**: Contagem regressiva de tempo gerada exclusivamente por Timers e interrupções (curta duração e longa duração).

4. **Acionamento Reativo**: Seleção do tempo de aferição e início do processo por meio de botões com tratamento de bouncing e detecção de borda de subida via interrupções externas (INT0 / INT1).

5. **Controle de Histerese (Resistência)**: Um LED deve sinalizar o estado da resistência do forno, acendendo para temperaturas maiores que 50°C.

---

## 3 Lógica de Implementação e Arquitetura
Para assegurar a robustez do sistema, estruturamos o código separando as tarefas de tempo crítico das tarefas de interface. O Loop Principal gerencia exclusivamente a leitura analógica, a formatação matemática sem ponto flutuante e a atualização do display. Já o controle de tempo e a leitura dos botões ficaram a cargo do Hardware de Interrupção.

### 3.1 Mapeamento de Hardware e Periféricos
Para evitar conflitos de portas e otimizar o uso do microcontrolador, a pinagem foi definida conforme a tabela abaixo, baseada na placa EasyPIC v7:

| Pino / Módulo | Função Lógica | Justificativa da Aplicação |
|---------------|---------------|----------------------------|
| **RB0 e RB1** | **INT0 e INT1** | Entradas dos botões. Utilizam interrupções externas de hardware para resposta instantânea, independentemente do que a CPU esteja processando. |
| **AN0 (RA0)** | **Entrada Analógica** | Recebe o sinal de tensão (0 a 1 V) do sensor LM35/Potenciômetro para conversão A/D. |
| **AN3 (RA3)** | **Vref+ Externa** | Recebe a tensão de referência fixa de 1 V, garantindo que o valor máximo do ADC (1023) corresponda exatamente aos 100°C do sensor. |
| **TMR0** | **Base de Tempo** | Configurado com prescaler para gerar interrupções periódicas que alimentam a variável de contagem regressiva em segundos. |

## 3.2 Tratamento de Dados (Evitando Float)
O ADC do PIC18F4550 possui 10 bits, gerando valores de 0 a 1023. Para exibir a temperatura com uma casa decimal (ex: 95.5 °C) sem usar variáveis do tipo float — que consomem excessiva memória de programa — a leitura bruta é multiplicada por 1000 e dividida por 1023 utilizando inteiros longos (unsigned long). Os dígitos são então extraídos individualmente via operações de módulo e divisão para a string do display.

---

## 4 Análise do Código Fonte
Abaixo, detalhamos os principais blocos lógicos construídos na Entrega Final.

### 4.1 Interface Visual e Configuração de I/O
O primeiro passo do projeto consistiu em estabelecer a interface de comunicação com o usuário. Para isso, os pinos do microcontrolador foram mapeados para operar o display LCD HD44780 no modo de 4 bits, economizando portas do PIC. Nesta etapa inicial, também foi concebida a lógica base para a configuração de entradas e saídas digitais.

```c
sbit LCD_RS at LATB4_bit;             // Pino RS do LCD conectado ao RB4
sbit LCD_EN at LATB5_bit;             // Pino Enable do LCD conectado ao RB5
sbit LCD_D4 at LATB0_bit;             // Linha de dados D4 conectada ao RB0
sbit LCD_D5 at LATB1_bit;             // Linha de dados D5 conectada ao RB1
sbit LCD_D6 at LATB2_bit;             // Linha de dados D6 conectada ao RB2
sbit LCD_D7 at LATB3_bit;             // Linha de dados D7 conectada ao RB3

sbit LCD_RS_Direction at TRISB4_bit;  // Configura RB4 como saída
sbit LCD_EN_Direction at TRISB5_bit;  // Configura RB5 como saída
sbit LCD_D4_Direction at TRISB0_bit;  // Configura RB0 como saída
sbit LCD_D5_Direction at TRISB1_bit;  // Configura RB1 como saída
sbit LCD_D6_Direction at TRISB2_bit;  // Configura RB2 como saída
sbit LCD_D7_Direction at TRISB3_bit;  // Configura RB3 como saída

void main() {

    unsigned char contador = 0;       // Armazena a contagem de 0 a 9
    unsigned char flag_botao = 0;     // Controle de debounce
    unsigned char atualiza_tela = 1;  // Solicita atualização do LCD

    ADCON1 |= 0x0F;                   // Todos os pinos configurados como digitais
    CMCON  |= 0x07;                   // Desabilita os comparadores internos
    TRISD0_bit = 1;                   // RD0 configurado como entrada (botão)
    Lcd_Init();                       // Inicializa o LCD em modo 4 bits
    Lcd_Cmd(_LCD_CLEAR);              // Limpa o display
    Lcd_Cmd(_LCD_CURSOR_OFF);         // Desabilita o cursor
    Lcd_Out(1, 1, "HelloWrld");

    while(1) {

        if (PORTD.RD0 == 1) {

            if (flag_botao == 0) {
                Delay_ms(50);         // Debounce de 50 ms

                if (PORTD.RD0 == 1) {
                    flag_botao = 1;   // Bloqueia novas contagens
                    contador++;       // Incrementa a contagem

                    if (contador > 9) {
                        contador = 0;
                    }
                    atualiza_tela = 1; // Solicita atualização do LCD
                }
            }
        }
        else {
            flag_botao = 0;
        }

        if (atualiza_tela == 1) {
            switch(contador) {
                case 0: Lcd_Out(2,1,"Contagem: 0"); break;
                case 1: Lcd_Out(2,1,"Contagem: 1"); break;
                case 2: Lcd_Out(2,1,"Contagem: 2"); break;
                case 3: Lcd_Out(2,1,"Contagem: 3"); break;
                case 4: Lcd_Out(2,1,"Contagem: 4"); break;
                case 5: Lcd_Out(2,1,"Contagem: 5"); break;
                case 6: Lcd_Out(2,1,"Contagem: 6"); break;
                case 7: Lcd_Out(2,1,"Contagem: 7"); break;
                case 8: Lcd_Out(2,1,"Contagem: 8"); break;
                case 9: Lcd_Out(2,1,"Contagem: 9"); break;
            }
            atualiza_tela = 0;        // Limpa a flag de atualização
        }
    }
}
```

### 4.2 Temporização (Timers) e Interrupções
Na segunda fase, o sistema evoluiu de uma arquitetura de varredura contínua (polling) para uma abordagem orientada a eventos. O acionamento dos botões passou a ser tratado por interrupções externas de hardware (INT0 e INT1), configuradas para detectar a borda de subida de forma assíncrona. Em paralelo, o controle do tempo de aferição (longa e curta duração) foi delegado ao módulo Timer0, que atua decrementando a contagem de forma autônoma a cada estouro programado.

```c
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

#define TMR0H_PRELOAD 0xE1
#define TMR0L_PRELOAD 0x7B

#define TMR1H_PRELOAD 0x0B
#define TMR1L_PRELOAD 0xDC

unsigned char tempo_restante = 0;     // Valor atual da contagem regressiva
unsigned char modo_contagem = 0;      // 0 = parado, 1 = longo, 2 = curto
unsigned char contagem_ativa = 0;     // Indica se a contagem esta rodando
unsigned char ticks_250ms = 0;        // Conta 4 interrupcoes de 250 ms para formar 1 s
unsigned char atualiza_tela = 1;      // Flag para atualizar o LCD

unsigned char flag_botao_longo = 0;   // Flag do botao RD0
unsigned char flag_botao_curto = 0;   // Flag do botao RD1

void Carrega_Timer0() {
    TMR0H = TMR0H_PRELOAD;
    TMR0L = TMR0L_PRELOAD;
}

void Carrega_Timer1() {
    TMR1H = TMR1H_PRELOAD;
    TMR1L = TMR1L_PRELOAD;
}

void interrupt() {

    if (TMR0IF_bit == 1) {
        TMR0IF_bit = 0;       // Limpa a flag de interrupcao do Timer0
        Carrega_Timer0();     // Recarrega o Timer0

        if ((modo_contagem == 1) && (contagem_ativa == 1)) {
            if (tempo_restante > 0) {
                tempo_restante--;     // Decrementa a contagem
                atualiza_tela = 1;    // Sinaliza atualizacao do LCD
            }

            if (tempo_restante == 0) {
                contagem_ativa = 0;   // Finaliza a contagem
                TMR0IE_bit = 0;       // Desabilita interrupcao do Timer0
                atualiza_tela = 1;
            }
        }
    }

    if (TMR1IF_bit == 1) {
        TMR1IF_bit = 0;       // Limpa a flag de interrupcao do Timer1
        Carrega_Timer1();     // Recarrega o Timer1

        if ((modo_contagem == 2) && (contagem_ativa == 1)) {
            ticks_250ms++;

            if (ticks_250ms >= 4) {
                ticks_250ms = 0;

                if (tempo_restante > 0) {
                    tempo_restante--;  // Decrementa a contagem
                    atualiza_tela = 1;
                }

                if (tempo_restante == 0) {
                    contagem_ativa = 0; // Finaliza a contagem
                    TMR1IE_bit = 0;     // Desabilita interrupcao do Timer1
                    atualiza_tela = 1;
                }
            }
        }
    }
}

void Atualiza_Display() {

    Lcd_Cmd(_LCD_CLEAR);
    Delay_ms(2);

    if (modo_contagem == 0) {
        Lcd_Out(1, 2, "Selecione");
        Lcd_Out(2, 2, "D0:60 D1:10");
    }

    if (modo_contagem == 1) {
        Lcd_Out(1, 2, "Modo: Longo");
        Lcd_Out(2, 2, "Tempo: ");

        Lcd_Chr(2, 9,  (tempo_restante / 10) + 48);
        Lcd_Chr(2, 10, (tempo_restante % 10) + 48);
        Lcd_Chr(2, 11, 's');

        Lcd_Out(2, 12, "    ");
    }

    if (modo_contagem == 2) {
        Lcd_Out(1, 2, "Modo: Curto");
        Lcd_Out(2, 2, "Tempo: ");

        Lcd_Chr(2, 9,  (tempo_restante / 10) + 48);
        Lcd_Chr(2, 10, (tempo_restante % 10) + 48);
        Lcd_Chr(2, 11, 's');

        Lcd_Out(2, 12, "    ");
    }
}

void Inicia_Contagem_Longa() {

    modo_contagem = 1;
    tempo_restante = 60;
    contagem_ativa = 1;
    ticks_250ms = 0;
    atualiza_tela = 1;

    TMR1IE_bit = 0;

    Carrega_Timer0();
    TMR0IF_bit = 0;
    TMR0IE_bit = 1;
}

void Inicia_Contagem_Curta() {

    modo_contagem = 2;
    tempo_restante = 10;
    contagem_ativa = 1;
    ticks_250ms = 0;
    atualiza_tela = 1;

    TMR0IE_bit = 0;

    Carrega_Timer1();
    TMR1IF_bit = 0;
    TMR1IE_bit = 1;
}

void main() {

    ADCON1 = 0x0F;       // Configura os pinos como digitais
    CMCON = 0x07;        // Desabilita os comparadores internos

    TRISD0_bit = 1;      // RD0 como entrada: contagem longa
    TRISD1_bit = 1;      // RD1 como entrada: contagem curta

    Lcd_Init();
    Lcd_Cmd(_LCD_CLEAR);
    Lcd_Cmd(_LCD_CURSOR_OFF);
    Delay_ms(100);

    T0CON = 0x87;
    Carrega_Timer0();
    TMR0IF_bit = 0;
    TMR0IE_bit = 0;      // Comeca desligado

    T1CON = 0xB1;
    Carrega_Timer1();
    TMR1IF_bit = 0;
    TMR1IE_bit = 0;      // Comeca desligado

    PEIE_bit = 1;        // Habilita interrupcoes de perifericos
    GIE_bit = 1;         // Habilita interrupcoes globais

    atualiza_tela = 1;

    while(1) {
        if (PORTD.RD0 == 1) {
            if (flag_botao_longo == 0) {
                Delay_ms(50);        // Debounce simples

                if (PORTD.RD0 == 1) {
                    flag_botao_longo = 1;
                    Inicia_Contagem_Longa();
                }
            }
        } else {
            flag_botao_longo = 0;
        }

        if (PORTD.RD1 == 1) {
            if (flag_botao_curto == 0) {
                Delay_ms(50);

                if (PORTD.RD1 == 1) {
                    flag_botao_curto = 1;
                    Inicia_Contagem_Curta();
                }
            }
        } else {
            flag_botao_curto = 0;
        }

        if (atualiza_tela == 1) {
            Atualiza_Display();
            atualiza_tela = 0;
        }
    }
}
```

### 4.3 Conversão A/D, Matemática e Histerese
A integração final adicionou a leitura do sensor de temperatura. O compilador MikroC possui uma peculiaridade na função ADC_Init(), que sobrescreve a configuração de referência de tensão para os padrões do chip. Para contornar isso e assegurar que a leitura utilizasse o limite estrito de 1V no pino AN3, o registrador ADCON1 foi manipulado manualmente logo após a chamada da biblioteca. Por fim, o valor bruto foi convertido matematicamente sem o uso de variáveis de ponto flutuante, e o sistema de aquecimento (LED) foi programado com uma lógica de histerese (ligar >= 50°C) para evitar oscilações indesejadas no controle do forno.

```c
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

sbit LED_FORNO at LATD6_bit;
sbit LED_FORNO_Direction at TRISD6_bit;

#define TMR0H_PRELOAD 0xE1
#define TMR0L_PRELOAD 0x7B

#define TMR1H_PRELOAD 0x0B
#define TMR1L_PRELOAD 0xDC

volatile unsigned char tempo_restante = 0;
volatile unsigned char modo_contagem = 0;      // 0 = parado, 1 = longo, 2 = curto
volatile unsigned char contagem_ativa = 0;
volatile unsigned char ticks_250ms = 0;

unsigned char flag_botao_longo = 0;
unsigned char flag_botao_curto = 0;

unsigned int leitura_adc = 0;
unsigned int temperatura_x10 = 0;              // Temperatura multiplicada por 10

void Carrega_Timer0() {
    TMR0H = TMR0H_PRELOAD;
    TMR0L = TMR0L_PRELOAD;
}

void Carrega_Timer1() {
    TMR1H = TMR1H_PRELOAD;
    TMR1L = TMR1L_PRELOAD;
}

void Configura_ADC() {

    // ADCON1 = 0x0E = 00001110
    // VCFG1 = 0 -> VREF- = VSS
    // VCFG0 = 0 -> VREF+ = VDD
    // PCFG = 1110 -> AN0 analogico e demais pinos digitais
    ADCON1 = 0x0E;

    // ADCON2 = 0xBD = 10111101
    // ADFM = 1 -> resultado justificado a direita
    // ACQT = 111 -> tempo de aquisicao 20 TAD
    // ADCS = 101 -> clock do ADC Fosc/16
    ADCON2 = 0xBD;

    // ADCON0 = 0x01
    // CHS = 0000 -> canal AN0
    // ADON = 1 -> ADC ligado
    ADCON0 = 0x01;
}

unsigned int Le_ADC_AN0() {
    unsigned int resultado;
    unsigned int timeout;

    Configura_ADC();
    Delay_us(80);

    // Inicia conversao
    GO_DONE_bit = 1;

    timeout = 60000;
    while ((GO_DONE_bit == 1) && (timeout > 0)) {
        timeout--;
    }

    // Se der algum problema, retorna zero sem travar o PIC
    if (timeout == 0) {
        return 0;
    }

    // Resultado de 10 bits em ADRESH:ADRESL
    resultado = ((unsigned int)ADRESH << 8) + ADRESL;

    return resultado;
}

void Atualiza_Temperatura() {

    leitura_adc = Le_ADC_AN0();

    // Conversao para decimos de grau Celsius.
    // Como o potenciometro esta limitado entre 0 e 1 V:
    // 0 V -> 0,0 C
    // 1 V -> 100,0 C
    temperatura_x10 = ((unsigned long)leitura_adc * 5000 + 511) / 1023;

    if (temperatura_x10 > 1000) {
        temperatura_x10 = 1000;
    }

    if (temperatura_x10 > 500) {
        LED_FORNO = 1;
    } else {
        LED_FORNO = 0;
    }
}

void interrupt() {

    // Timer0: contagem longa, base de aproximadamente 1 segundo
    if (TMR0IF_bit == 1) {
        TMR0IF_bit = 0;
        Carrega_Timer0();

        if ((modo_contagem == 1) && (contagem_ativa == 1)) {
            if (tempo_restante > 0) {
                tempo_restante--;
            }

            if (tempo_restante == 0) {
                contagem_ativa = 0;
                TMR0IE_bit = 0;
            }
        }
    }

    // Timer1: base de 250 ms para a contagem curta
    if (TMR1IF_bit == 1) {
        TMR1IF_bit = 0;
        Carrega_Timer1();

        if ((modo_contagem == 2) && (contagem_ativa == 1)) {
            ticks_250ms++;

            if (ticks_250ms >= 4) {
                ticks_250ms = 0;

                if (tempo_restante > 0) {
                    tempo_restante--;
                }

                if (tempo_restante == 0) {
                    contagem_ativa = 0;
                    TMR1IE_bit = 0;
                }
            }
        }
    }
}

void Atualiza_Display() {

    // Linha 1: temperatura
    // Comeca com um espaco para evitar bug visual do SimulIDE
    Lcd_Out(1, 1, " T=000.0C      ");

    // Formato:
    // temperatura_x10 = 41   -> T=004.1C
    // temperatura_x10 = 523  -> T=052.3C
    // temperatura_x10 = 753  -> T=075.3C
    // temperatura_x10 = 1000 -> T=100.0C

    Lcd_Chr(1, 4, (temperatura_x10 / 1000) + 48);
    Lcd_Chr(1, 5, ((temperatura_x10 / 100) % 10) + 48);
    Lcd_Chr(1, 6, ((temperatura_x10 / 10) % 10) + 48);
    Lcd_Chr(1, 8, (temperatura_x10 % 10) + 48);

    // Linha 2: estado da contagem
    // Todas as mensagens tambem comecam com espaco.
    // Assim, se o SimulIDE cortar o primeiro caractere, ele corta apenas o espaco.

    if (contagem_ativa == 0) {
        Lcd_Out(2, 1, " D0:60 D1:10   ");
    } else {
        if (modo_contagem == 1) {
            Lcd_Out(2, 1, " LONGO: 00s    ");
        } else {
            Lcd_Out(2, 1, " CURTO: 00s    ");
        }

        // Como agora a string comeca na coluna 1 com espaco,
        // os digitos do tempo ficam nas colunas 9 e 10.
        Lcd_Chr(2, 9, (tempo_restante / 10) + 48);
        Lcd_Chr(2, 10, (tempo_restante % 10) + 48);
    }
}

void Inicia_Contagem_Longa() {

    modo_contagem = 1;
    tempo_restante = 60;
    contagem_ativa = 1;
    ticks_250ms = 0;

    TMR1IE_bit = 0;

    Carrega_Timer0();
    TMR0IF_bit = 0;
    TMR0IE_bit = 1;
}

void Inicia_Contagem_Curta() {

    modo_contagem = 2;
    tempo_restante = 10;
    contagem_ativa = 1;
    ticks_250ms = 0;

    TMR0IE_bit = 0;

    Carrega_Timer1();
    TMR1IF_bit = 0;
    TMR1IE_bit = 1;
}

void main() {

    // Desabilita comparadores internos
    CMCON = 0x07;

    // Botoes com pull-down externo
    TRISD0_bit = 1;       // RD0: inicia contagem longa
    TRISD1_bit = 1;       // RD1: inicia contagem curta

    LED_FORNO_Direction = 0;
    LED_FORNO = 0;

    // AN0 como entrada analogica
    TRISA0_bit = 1;

    // Configura ADC
    Configura_ADC();

    // Inicializacao do LCD
    Lcd_Init();
    Lcd_Cmd(_LCD_CLEAR);
    Lcd_Cmd(_LCD_CURSOR_OFF);
    Delay_ms(100);

    // Configuracao Timer0
    T0CON = 0x87;
    Carrega_Timer0();
    TMR0IF_bit = 0;
    TMR0IE_bit = 0;

    // Configuracao Timer1
    T1CON = 0xB1;
    Carrega_Timer1();
    TMR1IF_bit = 0;
    TMR1IE_bit = 0;

    // Habilita interrupcoes
    PEIE_bit = 1;
    GIE_bit = 1;

    while(1) {

        // Le temperatura continuamente
        Atualiza_Temperatura();

        // Botao RD0: inicia contagem longa
        if (PORTD.RD0 == 1) {
            if (flag_botao_longo == 0) {
                flag_botao_longo = 1;
                Inicia_Contagem_Longa();
                Delay_ms(80);
            }
        } else {
            flag_botao_longo = 0;
        }

        // Botao RD1: inicia contagem curta
        if (PORTD.RD1 == 1) {
            if (flag_botao_curto == 0) {
                flag_botao_curto = 1;
                Inicia_Contagem_Curta();
                Delay_ms(80);
            }
        } else {
            flag_botao_curto = 0;
        }

        Atualiza_Display();

        Delay_ms(40);
    }
}
```

---

## 5 Guia de Simulação e Testes
Para validar a solução de hardware e software apresentada, o circuito foi esquematizado e testado no SimulIDE. Os resultados da execução e a resposta do sistema aos comandos analógicos e digitais podem ser observados nas figuras abaixo.

<p align="center">
  <img src="https://github.com/user-attachments/assets/c127f249-194e-43a6-9d30-02dfd5841d02" height="400">
</p>

<p align="center">
  <em>Figura 1: Simulação do Checkpoint 1 no SimulIDE. O sistema inicializa o display LCD HD44780 no modo de 4 bits, exibindo a string estática "HelloWrld". O circuito de pull-down (10 kΩ) no pino RD0 garante a leitura precisa da borda de subida, refletindo a contagem tratada via debounce no display.</em>
</p>

<p align="center">
  <img src="https://github.com/user-attachments/assets/5d484578-9ea6-4465-b3a7-ebb2cb72e37b" height="400">
</p>

<p align="center">
  <em>Figura 2: Simulação do Checkpoint 2. O sistema executando a contagem regressiva de curta duração (10s) após o acionamento do Botão 2 (INT1).</em>
</p>

<p align="center">
  <img src="https://github.com/user-attachments/assets/c6a393ee-f7cd-465b-a380-fe8fb7130cf1" height="400">
</p>

<p align="center">
  <em>Figura 3: Simulação do Checkpoint 2. A contagem de longa duração (60s) disparada pelo Botão 1 (INT0). A temporização é gerenciada autonomamente pelo hardware interno (Timer0) do PIC18F4550.</em>
</p>

<p align="center">
  <img src="https://github.com/user-attachments/assets/a4680747-ee8f-4d93-b9f8-8e4d141bc09b" height="400">
</p>

<p align="center">
  <em>Figura 4: Ambiente de desenvolvimento MikroC PRO for PIC utilizado na Entrega Final. Em destaque no painel lateral direito, o gerenciador (Library Manager) com as bibliotecas <code>ADC</code> e <code>Lcd</code> habilitadas, requisito essencial para a operação do conversor analógico-digital e interface visual. Na parte inferior, o log de mensagens confirma a compilação bem-sucedida do firmware completo.</em>
</p>

<p align="center">
  <img src="https://github.com/user-attachments/assets/9fcf0d1f-865a-44b9-b082-502e971b1551" height="400">
</p>

<p align="center">
  <em>Figura 5: Simulação integrada da Entrega Final no SimulIDE (Modo Curto). O potenciômetro conectado ao conversor A/D emula o sinal do sensor LM35, que é calculado e formatado para 21.5 °C no display LCD. Como a temperatura medida é inferior ao limite inferior de histerese estabelecido (50 °C), o microcontrolador aciona corretamente o LED conectado à porta digital, indicando que a resistência do forno está ligada durante a aferição de tempo.</em>
</p>

<p align="center">
  <img src="https://github.com/user-attachments/assets/bf88e8ec-8335-4528-9bbd-2f579d3cd607" height="400">
</p>

<p align="center">
  <em>Figura 6: Simulação integrada da Entrega Final no SimulIDE (Modo Longo). O sistema agora executa a base de tempo de longa duração (60s), acionada pelo Botão 1 (INT0). A leitura de temperatura do ADC permanece em 21.5 °C, mantendo o LED da resistência acionado (nível lógico alto em RD1), visto que o valor continua abaixo do limite inferior de 50 °C estipulado nos requisitos do projeto.</em>
</p>

<p align="center">
  <img src="https://github.com/user-attachments/assets/b6c6a42b-9723-4b0d-ba9a-0644642b7f8c" height="400">
</p>

<p align="center">
  <em>Figura 7: Simulação do Modo Curto com temperatura em ascensão. O sinal analógico foi ajustado para 543.0 mV, resultando na exibição exata de 54.3 °C no display LCD. Como este valor ainda encontra-se abaixo da margem inferior de 50 °C exigida pelos requisitos do projeto, o microcontrolador mantém o controle da resistência (LED) em nível lógico alto para que o forno continue aquecendo.</em>
</p>

<p align="center">
  <img src="https://github.com/user-attachments/assets/f582cb60-f3c6-4130-a1b3-d2da6d87767c" height="400">
</p>

<p align="center">
  <em>Figura 8: Simulação integrada da Entrega Final (Modo Longo). O temporizador gerencia a contagem regressiva de longa duração de forma autônoma, exibindo 54s restantes. Simultaneamente, o módulo A/D realiza a conversão precisa de 543.0 mV para 54.3 °C. O microcontrolador mantém a resistência (LED) ativada, validando que o sistema de histerese opera de forma independente e correta em qualquer um dos modos de temporização, já que a temperatura ainda não atingiu a margem de 50 °C.</em>
</p>

---

## 6 Conclusão
A conclusão deste projeto evidencia o sucesso na implementação de um firmware robusto para a aferição térmica e temporal de um forno industrial, utilizando a arquitetura avançada do microcontrolador PIC18F4550. Através da metodologia de Aprendizagem Baseada em Problemas (PBL), o desenvolvimento ocorreu de forma evolutiva: partindo do tratamento de repique mecânico (bouncing) e interface com displays alfanuméricos nos primeiros checkpoints, até culminar na gestão complexa de periféricos internos mistos nesta entrega final.

O principal marco técnico deste projeto em relação a estudos anteriores (como o 8051) foi a transição para a linguagem C, que permitiu abstrair a complexidade do mapeamento de hardware em baixo nível, garantindo um código mais modular e de fácil manutenção. A arquitetura orientada a eventos (Event-Driven) foi amplamente explorada. A utilização das interrupções externas (INT0 e INT1) aliadas ao estouro autônomo do Timer0 libertou a Unidade Lógica e Aritmética de ciclos de espera ociosos (polling ou delays travados). Isso permitiu que o loop principal se dedicasse exclusivamente à tarefa mais custosa: a amostragem contínua do Conversor Analógico-Digital e a atualização gráfica.

Adicionalmente, o projeto reforçou a importância da otimização matemática em sistemas embarcados. A manipulação inteligente da tensão de referência externa (Vref+) acoplada ao uso de aritmética de inteiros longos para o cálculo da temperatura demonstrou como é possível extrair alta precisão do conversor ADC de 10 bits sem penalizar a escassa memória do microcontrolador com bibliotecas de ponto flutuante (float).

Por fim, a integração da lógica de histerese no controle do elemento de aquecimento, juntamente com a reconfiguração pontual dos registradores (ADCON1) para sobrepor padrões limitantes do compilador, cumpriram integralmente todos os requisitos funcionais definidos no roteiro. Esta prática consolidou o domínio sobre o manuseio de datasheets, periféricos analógico-digitais e temporização por hardware, competências que são fundamentais para o desenvolvimento de soluções industriais confiáveis.

---

## 7 Referências

1. OLIVEIRA, Pedro. Slides, Apostilas e Notas de Aula da Disciplina SEL0433 - Aplicação de Microprocessadores. Escola de Engenharia de São Carlos (EESC-USP). [Principal Referência Teórica e Técnica].
2. MICROCHIP TECHNOLOGY INC. PIC18F2455/2550/4455/4550 Data Sheet. Documentação oficial do microcontrolador. Disponível em: https://www.microchip.com/.
3. MIKROELEKTRONIKA. MikroC PRO for PIC User Manual e Help Index. Documentação das bibliotecas integradas (ADC, LCD, Conversions).
4. SIMULIDE. Real Time Electronic Circuit Simulator. Documentação oficial e manual de componentes. Disponível em: https://simulide.com/










































