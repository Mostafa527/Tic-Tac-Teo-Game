// SpaceInvaders.c
// Runs on LM4F120/TM4C123
// Jonathan Valvano and Daniel Valvano
// This is a starter project for the edX Lab 15
// In order for other students to play your game
// 1) You must leave the hardware configuration as defined
// 2) You must not add/remove any files from the project
// 3) You must add your code only this this C file
// I.e., if you wish to use code from sprite.c or sound.c, move that code in this file
// 4) It must compile with the 32k limit of the free Keil

// April 10, 2014
// http://www.spaceinvaders.de/
// sounds at http://www.classicgaming.cc/classics/spaceinvaders/sounds.php
// http://www.classicgaming.cc/classics/spaceinvaders/playguide.php
/* This example accompanies the books
   "Embedded Systems: Real Time Interfacing to Arm Cortex M Microcontrollers",
   ISBN: 978-1463590154, Jonathan Valvano, copyright (c) 2013

   "Embedded Systems: Introduction to Arm Cortex M Microcontrollers",
   ISBN: 978-1469998749, Jonathan Valvano, copyright (c) 2013

 Copyright 2014 by Jonathan W. Valvano, valvano@mail.utexas.edu
    You may use, edit, r	un or distribute this file
    as long as the above copyright notice remains
 THIS SOFTWARE IS PROVIDED "AS IS".  NO WARRANTIES, WHETHER EXPRESS, IMPLIED
 OR STATUTORY, INCLUDING, BUT NOT LIMITED TO, IMPLIED WARRANTIES OF
 MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE APPLY TO THIS SOFTWARE.
 VALVANO SHALL NOT, IN ANY CIRCUMSTANCES, BE LIABLE FOR SPECIAL, INCIDENTAL,
 OR CONSEQUENTIAL DAMAGES, FOR ANY REASON WHATSOEVER.
 For more information about my classes, my research, and my books, see
 http://users.ece.utexas.edu/~valvano/
 */
// ******* Required Hardware I/O connections*******************
// Slide pot pin 1 connected to ground
// Slide pot pin 2 connected to PE2/AIN1
// Slide pot pin 3 connected to +3.3V 
// fire button connected to PE0
// special weapon fire button connected to PE1
// 8*R resistor DAC bit 0 on PB0 (least significant bit)
// 4*R resistor DAC bit 1 on PB1
// 2*R resistor DAC bit 2 on PB2
// 1*R resistor DAC bit 3 on PB3 (most significant bit)
// LED on PB4
// LED on PB5

// Blue Nokia 5110
// ---------------
// Signal        (Nokia 5110) LaunchPad pin
// Reset         (RST, pin 1) connected to PA7
// SSI0Fss       (CE,  pin 2) connected to PA3
// Data/Command  (DC,  pin 3) connected to PA6
// SSI0Tx        (Din, pin 4) connected to PA5
// SSI0Clk       (Clk, pin 5) connected to PA2
// 3.3V          (Vcc, pin 6) power
// back light    (BL,  pin 7) not connected, consists of 4 white LEDs which draw ~80mA total
// Ground        (Gnd, pin 8) ground

// Red SparkFun Nokia 5110 (LCD-10168)
// -----------------------------------
// Signal        (Nokia 5110) LaunchPad pin
// 3.3V          (VCC, pin 1) power
// Ground        (GND, pin 2) ground
// SSI0Fss       (SCE, pin 3) connected to PA3
// Reset         (RST, pin 4) connected to PA7
// Data/Command  (D/C, pin 5) connected to PA6
// SSI0Tx        (DN,  pin 6) connected to PA5
// SSI0Clk       (SCLK, pin 7) connected to PA2
// back light    (LED, pin 8) not connected, consists of 4 white LEDs which draw ~80mA total

#include "tm4c123gh6pm.h"
#include "Nokia5110.h"
#include "Random.h"
#include "TExaS.h"




void DisableInterrupts(void); // Disable interrupts
void EnableInterrupts(void);  // Enable interrupts
void Timer2_Init(unsigned long period);
void Delay100ms(unsigned long count); // time delay in 0.1 seconds

//Helping Functions
void Initalization(char TYP);
void Test2(char c);
void Test3(char c);
void DrawingFunc(void);
void CursSetter(void);
void CursRemover(void);
void Lab(void);

char Gamestarter(void);
char GetWinnerPlayer(void); //Function to get Winner player X OR O 
unsigned long TimerCount;
unsigned long Semaphore;
char Place,type,Dispx,Dispy,Step,Gamer,Complete; //Initalization Variables we used while programm
char Arr[3][3];
char _Win_=0,T=1;



 
			//Initalization PortF to use 2 switches while playing to make step next or step back into selected position
	void PortF_Init(void){ volatile unsigned long delay;
  SYSCTL_RCGC2_R |= 0x00000020;     // 1) F clock
  delay = SYSCTL_RCGC2_R;           // delay   
  GPIO_PORTF_LOCK_R = 0x4C4F434B;   // 2) unlock PortF PF0  
  GPIO_PORTF_CR_R = 0x1F;           // allow changes to PF4-0       
  GPIO_PORTF_AMSEL_R = 0x00;        // 3) disable analog function
  GPIO_PORTF_PCTL_R = 0x00000000;   // 4) GPIO clear bit PCTL  
  GPIO_PORTF_DIR_R = 0x0E;          // 5) PF4,PF0 input, PF3,PF2,PF1 output   
  GPIO_PORTF_AFSEL_R = 0x00;        // 6) no alternate function
  GPIO_PORTF_PUR_R = 0x11;          // enable pullup resistors on PF4,PF0       
  GPIO_PORTF_DEN_R = 0x1F;          // 7) enable digital pins PF4-PF0        
}
	//Initalization PortE Registers to control Nokia Screen
	
	void PORTE_INIT(void){
	volatile unsigned long delay;
	SYSCTL_RCGC2_R |= 0x00000010;     // 1) F clock
  delay = SYSCTL_RCGC2_R;           // delay   
  GPIO_PORTE_LOCK_R = 0x4C4F434B;   // 2) unlock PortE
  GPIO_PORTE_CR_R = 0x03;           // allow changes to PE2       
  GPIO_PORTE_AMSEL_R = 0x00;        // 3) disable analog function
  GPIO_PORTE_PCTL_R = 0x00000000;   // 4) GPIO clear bit PCTL  
  GPIO_PORTE_DIR_R = 0x02;          // 5) PE0 input, PE1 output   
  GPIO_PORTE_AFSEL_R = 0x00;        // 6) no alternate function
  GPIO_PORTE_PDR_R= 0x01;          // enable pulldown resistors on PE0       
  GPIO_PORTE_DEN_R = 0x01;          // 7) enable digital pins PE0-PE1
}


void Initalization(char TYP){
	
	char z,k;
	type=TYP;
	Place=0;
	Gamer=(type*type-1);
	Complete=1;
 Step=0; // 0 For X-Player and 1 For O-Player
	if(TYP==3)Dispx=4,Dispy=2;
		
		for( z=0;z<TYP;z++){
		 for( k=0;k<TYP;k++)
		 Arr[z][k]=0;
	 }
}

char Gamestarter(void){
	char  SWITCH3 , ch=3;
	
	Nokia5110_SetCursor(1,2);
	Nokia5110_OutString("Welcome to XO Game !!");

	while(1){
	 SWITCH3 = GPIO_PORTE_DATA_R&0x01;     // Get PE0 inside SWITCH3
	if(SWITCH3) return ch;
	}

	
}

void DrawingFunc(){
	char X,Y,d,w;
	 X=84/type;
	 Y=48/type;
	for( d=0;d<type;d++){
		for( w=0;w<84;w++)Nokia5110_SetPixel(w,Y*d);
		for( w=0;w<48;w++)Nokia5110_SetPixel(X*d,w);
	}
	
	 Nokia5110_SetCursor(1,5);
	 Nokia5110_OutString("X->Role");
	
}

	
 void CursSetter(){
	char x_Position,y_Position;
	 if( Arr[Place/type][Place%type]==0){		       
						Arr[Place/type][Place%type]='#';
						x_Position=(Dispx*(Place%type));
				    y_Position=(Dispy*(Place/type));
						Nokia5110_SetCursor(x_Position,y_Position);
			  	  Nokia5110_OutChar('#');
		        Nokia5110_SetCursor(x_Position,y_Position);
						}
	 }
 
	 void CursRemover(){
		char x_Position,y_Position;
		 if(Arr[Place/type][Place%type]=='#'){
			   
					Arr[Place/type][Place%type]=0;
					x_Position=(Dispx*(Place%type));
				  y_Position=(Dispy*(Place/type));
					Nokia5110_SetCursor(x_Position,y_Position);
			  	Nokia5110_OutChar(' ');
				}
		 
	 }
	 
	 char GetWinnerPlayer(void){
	
			char i,j, RowxCount=0,RowOCount=0,ColXCount=0,ColOCount=0,DiagXCount=0,DiagOCount=0,Diag2XCount=0,Diag2OCount=0,F=1;
		 for( i=0; i<type;i++){
			 RowxCount=0,RowOCount=0,ColXCount=0,ColOCount=0;
			 
			 //Diagonal Check
			 switch(Arr[i][i]){
				 case 'X':
					 DiagXCount++;
						break;
				 case 'O' :
					 DiagOCount++;
						break;
				 
			 }
				//Check Diagonal Two
				switch(Arr[i][type-1-i])
				{
					case 'X' :
					Diag2XCount++;
					break;
					case 'O' :
					Diag2OCount++;
						break;
				}
				
		
		    
			 for( j=0;j<type; j++){
					
				 if(Arr[i][j]==0) F=0;  
					switch(Arr[i][j]){
						case 'X':
						RowxCount++;  // Increament X with Rows
						break;
						case 'O': // 	Incremeant O With Rows
						RowOCount++;
						break;
					}
					switch(Arr[j][i]){
						case'X' :
						ColXCount++;  // Incremeant X with Columns
						break;
						case 'O' :
							ColOCount++;	// Increament O With Columns
						break;
					
					}
		 }
			if(DiagXCount==type||Diag2XCount==type||RowxCount==type||ColXCount==type)return 'X';
		  if(DiagOCount==type||RowOCount==type||Diag2OCount==type||ColOCount==type)return 'O';	
				
	 }
		 if(F) return 'E';
	   return 0;
}

void Clean(void){
	Delay100ms(5);
	Nokia5110_Clear();
	Nokia5110_SetCursor(0,1);
		Nokia5110_OutString("Thank you^_^");
}
void Lab(void){
 TExaS_Init(SSI0_Real_Nokia5110_Scope);  // set system clock to 80 MHz
	 Random_Init(1);
 	 Nokia5110_Init();                  //Initalization
	 Nokia5110_ClearBuffer();         // buffer is cleared
	 Nokia5110_DisplayBuffer();      // buffer is drawn
	
			PortF_Init(); //Call Function that initalize PortF
	 PORTE_INIT();	//Call Function that initalize PortF
	
	Nokia5110_DisplayBuffer();
	
	 Nokia5110_Clear();
	
	 Initalization(Gamestarter()); 
	 Nokia5110_Clear();
	 Delay100ms(1);
	 DrawingFunc();
	 CursSetter();
	 Nokia5110_SetCursor(0,0);

}
	void Test2(char c){
					Nokia5110_Clear();
					Nokia5110_SetCursor(0,1);
					switch(c){
						case 1:
						Nokia5110_OutString("Winner is X Congrats!!");
						break;
						case 2:
						Nokia5110_OutString("Winner is O Congrats !!");
						break;
						default:
						Nokia5110_OutString("No Winner");
						break;
							
					
					}
					Complete=0;
		}
void Test3(char c){
							Place++;
							if(Place>Gamer)
								Place=Gamer;
							Nokia5110_SetCursor(1,5);
							switch(c)
							{
								case 1:
							Nokia5110_OutString("X->Role");
								break;
								case 2 :
							Nokia5110_OutString("O->Role");
								break;
							}
							CursSetter();
							Step=Step^1;
								

}
	int main(void){
		 unsigned long SWITCH1,SWITCH2,SWITCH3;
		Lab();
	while(1){
		
		
			SWITCH3 = GPIO_PORTE_DATA_R&0x01;    
		 SWITCH1 = GPIO_PORTF_DATA_R&0x10;    
     SWITCH2 = GPIO_PORTF_DATA_R&0x01;     
			
			if(!(SWITCH1)){
				
        CursRemover();				
				
				Place++;
				if(Place>Gamer)
					Place=Gamer;
    	   while(!(GPIO_PORTF_DATA_R&0x10));
				
				  CursSetter();	
			}
				
			if(!(SWITCH2))
			{
				CursRemover();
				
				Place--;
				if(Place<0)
				Place=0;
				while(!(GPIO_PORTF_DATA_R&0x01));
				
				  CursSetter();
				
			}
			if((SWITCH3)){
				while(GPIO_PORTE_DATA_R&0x01);
				if(!(Step)){
					if(Arr[Place/type][Place%type]=='#')
					{
				  	Nokia5110_OutChar('X');
				  	Arr[Place/type][Place%type]='X';
						Test3(1);
						
					}
					
				}
					
				else{
					if(Arr[Place/type][Place%type]=='#')
						{
							Nokia5110_OutChar('O');
							Arr[Place/type][Place%type]='O';
							Test3(2);
					}
				}
				_Win_=GetWinnerPlayer();
				
			}
			if(_Win_){
				
				switch(_Win_){
					case 'X' :
					if(Complete){
						Test2(1);
					}
					break;
					case 'O' :
						if(Complete){
						Test2(2);
					}
					break;
					case 'E' :
					if(Complete){
						Test2(3);
					}
					break;
					default :
						break;
				}
				Clean();
			}
		
			}
	
		}
void Delay100ms(unsigned long count){unsigned long volatile time;
  while(count>0){
    time = 727240;  // 0.1sec at 80 MHz
    while(time){
	  	time--;
    }
    count--;
  }
}



