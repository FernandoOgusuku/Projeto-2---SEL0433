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

5. **Controle de Histerese (Resistência)**: Um LED deve sinalizar o estado da resistência do forno, acendendo em temperaturas inferiores a 60°C e apagando ao ultrapassar 80°C.

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


### 4.3 Conversão A/D, Matemática e Histerese
A integração final adicionou a leitura do sensor de temperatura. O compilador MikroC possui uma peculiaridade na função ADC_Init(), que sobrescreve a configuração de referência de tensão para os padrões do chip. Para contornar isso e assegurar que a leitura utilizasse o limite estrito de 1V no pino AN3, o registrador ADCON1 foi manipulado manualmente logo após a chamada da biblioteca. Por fim, o valor bruto foi convertido matematicamente sem o uso de variáveis de ponto flutuante, e o sistema de aquecimento (LED) foi programado com uma lógica de histerese (ligar < 60°C; desligar > 80°C) para evitar oscilações indesejadas no controle do forno.

[INSERIR BLOCO DE CÓDIGO AQUI: Trecho da função main() mostrando a reconfiguração manual do ADCON1 = 0b00011110 e todo o bloco interno do while(1) contendo o cálculo da temperatura_bruta, a formatação e o if/else do LED]

---

## 5 Código Fonte Completo
Este é o firmware final integrado desenvolvido para o projeto em linguagem C:

[INSERIR BLOCO DE CÓDIGO AQUI: Código fonte completo (.c) em MikroC]

---

## 6 Guia de Simulação e Testes
Para validar a solução de hardware e software apresentada, o circuito foi esquematizado e testado no SimulIDE. Os resultados da execução e a resposta do sistema aos comandos analógicos e digitais podem ser observados nas figuras abaixo.

---

## 7 Conclusão
A conclusão deste projeto evidencia o sucesso na implementação de um firmware robusto para a aferição térmica e temporal de um forno industrial, utilizando a arquitetura avançada do microcontrolador PIC18F4550. Através da metodologia de Aprendizagem Baseada em Problemas (PBL), o desenvolvimento ocorreu de forma evolutiva: partindo do tratamento de repique mecânico (bouncing) e interface com displays alfanuméricos nos primeiros checkpoints, até culminar na gestão complexa de periféricos internos mistos nesta entrega final.

O principal marco técnico deste projeto em relação a estudos anteriores (como o 8051) foi a transição para a linguagem C, que permitiu abstrair a complexidade do mapeamento de hardware em baixo nível, garantindo um código mais modular e de fácil manutenção. A arquitetura orientada a eventos (Event-Driven) foi amplamente explorada. A utilização das interrupções externas (INT0 e INT1) aliadas ao estouro autônomo do Timer0 libertou a Unidade Lógica e Aritmética de ciclos de espera ociosos (polling ou delays travados). Isso permitiu que o loop principal se dedicasse exclusivamente à tarefa mais custosa: a amostragem contínua do Conversor Analógico-Digital e a atualização gráfica.

Adicionalmente, o projeto reforçou a importância da otimização matemática em sistemas embarcados. A manipulação inteligente da tensão de referência externa (Vref+) acoplada ao uso de aritmética de inteiros longos para o cálculo da temperatura demonstrou como é possível extrair alta precisão do conversor ADC de 10 bits sem penalizar a escassa memória do microcontrolador com bibliotecas de ponto flutuante (float).

Por fim, a integração da lógica de histerese no controle do elemento de aquecimento, juntamente com a reconfiguração pontual dos registradores (ADCON1) para sobrepor padrões limitantes do compilador, cumpriram integralmente todos os requisitos funcionais definidos no roteiro. Esta prática consolidou o domínio sobre o manuseio de datasheets, periféricos analógico-digitais e temporização por hardware, competências que são fundamentais para o desenvolvimento de soluções industriais confiáveis.

---

## 8 Referências

1. OLIVEIRA, Pedro. Slides, Apostilas e Notas de Aula da Disciplina SEL0433 - Aplicação de Microprocessadores. Escola de Engenharia de São Carlos (EESC-USP). [Principal Referência Teórica e Técnica].
2. MICROCHIP TECHNOLOGY INC. PIC18F2455/2550/4455/4550 Data Sheet. Documentação oficial do microcontrolador. Disponível em: https://www.microchip.com/.
3. MIKROELEKTRONIKA. MikroC PRO for PIC User Manual e Help Index. Documentação das bibliotecas integradas (ADC, LCD, Conversions).
4. SIMULIDE. Real Time Electronic Circuit Simulator. Documentação oficial e manual de componentes. Disponível em: https://simulide.com/










































