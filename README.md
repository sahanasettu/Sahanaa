#include <lpc17xx.h>
#include "keypad.h"
#include "lcd_header.h"

#define ROW_MASK (0xF0)  // P2.4 to P2.7
#define COL_MASK (0x0F)  // P2.0 to P2.3

const char keymap[4][4] = {
    {'1','2','3','A'},
    {'4','5','6','B'},
    {'7','8','9','C'},
    {'*','0','#','D'}
};

void keypad_init(void) {
    LPC_GPIO2->FIODIR |= ROW_MASK;   // Rows as output
    LPC_GPIO2->FIODIR &= ~COL_MASK;  // Columns as input
}
char keypad_scan(void) {
    int row, col;

    for (row = 0; row < 4; row++) {

        LPC_GPIO2->FIOSET = ROW_MASK;          // All rows HIGH
        LPC_GPIO2->FIOCLR = (1 << (row + 4));  // One row LOW

        for (col = 0; col < 4; col++) {

            if (!(LPC_GPIO2->FIOPIN & (1 << col))) {

                delay_lcd(500);  // Debounce

                if (!(LPC_GPIO2->FIOPIN & (1 << col))) {

                    while (!(LPC_GPIO2->FIOPIN & (1 << col))); // Wait release
                    return keymap[row][col];
                }
            }
        }
    }
    return 0;
}
#include <LPC17xx.h>
#include "lcd_header.h"

#define RS (1<<10)
#define EN (1<<11)
#define DT_MASK (0xFF<<15)

void delay_lcd(unsigned int t) {
    int i, j;
    for(i = 0; i < t; i++)
        for(j = 0; j < 100; j++);
}

void lcd_pulse(void) {
    LPC_GPIO0->FIOSET = EN;
    delay_lcd(10);
    LPC_GPIO0->FIOCLR = EN;
	delay_lcd(10);
}

void lcd_cmd_write(unsigned char cmd) {
    LPC_GPIO0->FIOCLR = RS;
    LPC_GPIO0->FIOCLR = DT_MASK;
    LPC_GPIO0->FIOSET = (cmd <<15);
    lcd_pulse();
}

void lcd_data_write(unsigned char data) {
    LPC_GPIO0->FIOSET = RS;
    LPC_GPIO0->FIOCLR = DT_MASK;
    LPC_GPIO0->FIOSET = (data<<15);
    lcd_pulse();
}

void lcd_str_write(char *str) {
    while(*str!='\0') {
        lcd_data_write(*str);
		str++;
    }
}
#include <lpc17xx.h>
#include <string.h>
#include <stdio.h>

#include "lcd_header.h"
#include "keypad.h"

#define BUZZER (1<<27)

char *questions[] = {
   "Square root 16?",
    "First Prime?",
    "Square of 7?",
    "Next in seq: 2,4?",
    "x-5=0,x=?",

};

char *answers[] = {"4","2","49","8","5"};
int score = 0;

void delay(unsigned int t) {
    int i, j;
    for(i = 0; i < t; i++)
        for(j = 0; j < 1000; j++);
}

void buzzer_init() {
    LPC_GPIO1->FIODIR = BUZZER;
	LPC_GPIO1->FIOCLR = BUZZER;
}

void buzzer_beep_short() {
    LPC_GPIO1->FIOSET = BUZZER;
    delay(200);
    LPC_GPIO1->FIOCLR = BUZZER;
}

void buzzer_beep_long() {
    LPC_GPIO1->FIOSET = BUZZER;
    delay(600);
    LPC_GPIO1->FIOCLR = BUZZER;
}

void keypad_read(char *buf) {
    int idx = 0;
    char ch;
    while(1) {
        ch = keypad_scan();
        if(ch != '\0') {
            if(ch == '#') break;
            buf[idx++] = ch;
            lcd_data_write(ch);
        }
    }
    buf[idx] = '\0';
}

int main(void) {
    int i;
    char input[10];
    char buf[16];

    lcd_config();
    buzzer_init();
    keypad_init();

	lcd_str_write("QUIZ Contest..");
	delay(5000);

    for(i = 0; i < 5; i++) {
        lcd_cmd_write(0x01);
        lcd_str_write(questions[i]);
        lcd_cmd_write(0xC0);
        keypad_read(input);

        if(strcmp(input, answers[i]) == 0) {
            lcd_cmd_write(0x01);
            lcd_str_write("RIGHT");
            buzzer_beep_short();
            score++;
        } else {
            lcd_cmd_write(0x01);
            lcd_str_write("WRONG");
            buzzer_beep_long();
        }
        delay(2000);
    }

    lcd_cmd_write(0x01);
    sprintf(buf, "Score: %d/5", score);
    lcd_str_write(buf);
    buzzer_beep_short();
    buzzer_beep_short();

    while(1);
}

void lcd_config(void) {
    LPC_GPIO0->FIODIR = RS | EN | DT_MASK;
    lcd_cmd_write(0x38);
    lcd_cmd_write(0x0E);
    lcd_cmd_write(0x01);
	lcd_cmd_write(0x80);
}
