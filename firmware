/*
	BabyOrb by Orbitronics Firmware 
	Copyright (C) 2016:
		B. Kazemi, ebaykazemi@googlemail.com
	 
This program is free software; you can redistribute it and/or
modify it under the terms of the GNU General Public License
as published by the Free Software Foundation; either version 3
of the License, or (at your option) any later version.
This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.
You should have received a copy of the GNU General Public License
along with this program; if not, write to the Free Software
Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA  02110-1301, USA.

*/

/**********************************************************************************************************
  LIBRARIES
**********************************************************************************************************/
#include <Audio.h>
#include <Wire.h>
#include <SPI.h>

/**********************************************************************************************************
  PIN DEFS / MACROS
**********************************************************************************************************/
#define BLUETOOTH           			Serial1
#define RED_PIN							21
#define GREEN_PIN						20
#define BLUE_PIN						3
#define RED_LED							0
#define GREEN_LED						1
#define BLUE_LED						2
#define MOTOR_PIN  						4
#define MOTOR_PWM_FREQ 					18000
#define LOWEST_MOTOR_PWM 				50
#define LOWEST_MIN_BRIGHTNESS			40
#define LOWEST_PWM_VALUE				50 
#define MODE_STATIC_CONTROL				1
#define	MODE_LAVA_LAMP					2
#define MODE_SLEEP 						3
#define MODE_SLEEP_WITH_LISTENER		4
#define MINIMUM_MAX_PEAK_THRESHOLD 		0.2
#define MINIMUM_MOTOR_PWM 				70

/**********************************************************************************************************
  GLOBALS
**********************************************************************************************************/
uint8_t					mode_select = 			MODE_LAVA_LAMP;
uint8_t 				rgb_led_pins[3] = 		{RED_PIN, GREEN_PIN, BLUE_PIN};
uint8_t 				alertness = 			1; //map max noise to 255, silence to 0, if silent minimise slowly based on multiplier
uint8_t         		previousAlertness =     0;
uint8_t 				led_pwm[3] = 			{0, 0, 0};
uint8_t 				led_target[3] = 		{0, 0, 0};
uint8_t 				max_led_brightness = 	254;
uint8_t 				min_led_brightness = 	10; 
uint8_t 				motor_pwm_value = 		100;
uint16_t 				delayTime = 			100; 
unsigned long 			updateLEDMillis = 		0;
unsigned long 			currentMillis = 		0;
String 					inputString = 			"";         // a string to hold incoming data //for serial event
boolean 				stringComplete = 		false;  	// whether the string is complete //for serial event
boolean 				rgb_state[3] = 			{1, 1, 1};
elapsedMillis 			msec;
elapsedMillis 			peakReset;
elapsedMillis 			sleepTime;
elapsedMillis 			stateSendTime;
boolean 				showPeaks = 			true; // in terminal we change this to determine if we want to see the peaks or not 
float 					maxPeak = 				0.0;
float 					peakReadFloat;
int 					calmScalar = 			40; // this is a ms time per loop that decrements the alertness - change this to affect how baby light decays
AudioInputI2S			i2s1;          
AudioOutputI2S			i2s2;   
AudioAnalyzePeak		peak1;         
AudioConnection			patchCord1(i2s1, peak1);
AudioConnection			patchCord2(i2s1, i2s2);
AudioControlSGTL5000	sgtl5000_1;  


/**********************************************************************************************************
  setup
**********************************************************************************************************/
void setup()  {
	Serial.begin(115200);
	randomSeed(analogRead(17));
	AudioMemory(10);
	sgtl5000_1.enable();
	sgtl5000_1.volume(0.5);
	sgtl5000_1.inputSelect(AUDIO_INPUT_MIC);
	sgtl5000_1.micGain(44);
	inputString.reserve(200);//for serial event
	analogWriteFrequency(MOTOR_PIN, MOTOR_PWM_FREQ);// PWM Freq for motor 
	for (uint8_t i = 0; i < 3 && mode_select != MODE_SLEEP_WITH_LISTENER; i++) 
	{
		if (rgb_state[i])
			checkTarget(i);
	}
	if (mode_select == MODE_SLEEP || mode_select == MODE_SLEEP_WITH_LISTENER)
	{
		min_led_brightness = 0;
		maxPeak = 1.0;
	}
	analogWrite(MOTOR_PIN, motor_pwm_value); // slowest is 60 MIGHT NBEED RESISTOR INFRONT OF MOTOR
	updateLEDMillis = millis();
	currentMillis = millis();
	BLUETOOTH.begin(115200);
	delay(500);
	// BLUETOOTH.println("BabyOrb Babylight by Orbitronics. 2016");
	BLUETOOTH.flush();
} 

/**********************************************************************************************************
  function definitions
**********************************************************************************************************/
// assign new target if reached target
void checkTarget(uint8_t led)
{
	if (led_pwm[led] == led_target[led])
		assignTarget(led);
}

//assign a target
void assignTarget(uint8_t led)
{
	uint8_t randomNum = random(min_led_brightness, max_led_brightness+1);
	if (led == 0 && randomNum > 180) // this is a hack to fix the led fault on R
		randomNum = 180;
	led_target[led] = randomNum;
}

void updateCurrentMillis(void)
{
	currentMillis = millis();
}

void incDecVar(uint16_t *input, char mode, int value)
{
	if (mode == '+')
		(*input)++;
	else if (mode == '-')
		--(*input);
	else
		*input = value;
}

/**********************************************************************************************************
  MAIN LOOP
**********************************************************************************************************/
void loop()
{ 
	BLUETOOTH.flush(); // wait for the current transmission to complete 
	while (BLUETOOTH.available()) 
	{
		char inChar = (char)BLUETOOTH.read();// get the new byte:
		inputString += inChar;// add it to the inputString:
		// if the incoming character is a newline, set a flag so the main loop can do something about it
		if (inChar == ',') 
		{
			stringComplete = true;
		}
	}
	if ((mode_select == MODE_SLEEP_WITH_LISTENER || mode_select == MODE_SLEEP)) 
	{
		if (mode_select == MODE_SLEEP && msec > 24)
		{
			msec = 0;
			//  //BLUETOOTH.println(maxPeak);
			if (peakReset > calmScalar)
			{
				peakReset = 0; 
				if (maxPeak > 0.01)
				{
					maxPeak = maxPeak - 0.01;
				}
			}
			if (maxPeak <= 0.01) // we want this with the 3rd mode, but add the time factor, in the 4th mode add time factor but remove led targer reset
			{
				for (uint8_t i = 0; i < 3; i++)
				{
					led_target[i] = 0;
				}
			}  
		}
		else if (mode_select == MODE_SLEEP_WITH_LISTENER && peak1.available() && msec > 24) 
		{
			msec = 0;
			peakReadFloat = peak1.read();
			if (peakReadFloat >= MINIMUM_MAX_PEAK_THRESHOLD && peakReadFloat > maxPeak)
				maxPeak = peakReadFloat;
			else if (peakReset > calmScalar)
			{
				peakReset = 0; 
				if (maxPeak > 0.01)
				{
					maxPeak = maxPeak - 0.01;
				}
			}
			if (maxPeak <= 0.01) // we want this with the 3rd mode, but add the time factor, in the 4th mode add time factor but remove led targer reset
			{

				for (uint8_t i = 0; i < 3; i++)
				{
					led_target[i] = 0;
				}
			}
			int count = 0;  
			int monoPeak = peakReadFloat * 30.0;
			int maxPeakGfx = maxPeak * 30.0;
			if (showPeaks)
			{
				for (count=0; count < 30 - maxPeakGfx; count++) 
				{
					Serial.print(" ");
				}
				while (count++ < 30) 
				{
					Serial.print("<");
				}
				Serial.print(" | ");
				Serial.print(maxPeak);
				Serial.print("  ");
				Serial.print(alertness);
				Serial.print("  ||  ");
				Serial.print(peakReadFloat);
				Serial.print("  | ");
				for (count=0; count<monoPeak; count++) 
				{
					Serial.print(">");
				}
				Serial.println();
			}
		}
		/*
		* DO THE FOLLOWING FOR BOTH SLEEP MODES 
		*/
		previousAlertness = alertness;
		alertness = map(int(maxPeak * 100), 0, 100, 0, 254);
		if (alertness < 100)
			delayTime = 100 + (100 - int(maxPeak * 100));
		if (previousAlertness < alertness)
		{
			motor_pwm_value = 100;
		}
		else if (alertness > 0)
		{
			motor_pwm_value = map(int(maxPeak * 100), 0, 100, MINIMUM_MOTOR_PWM, 120);
		}
		if (motor_pwm_value == MINIMUM_MOTOR_PWM)
		{
			boolean stillWorking = false;
			for (uint8_t i = 0; i < 3 && stillWorking == false; i++)
			{
				if ( led_pwm[i] > 0) //led_pwm[i] != led_target[i] &&
					stillWorking = true;
			}
			if (!stillWorking)
			{
				//BLUETOOTH.print("sleepTime: ");//BLUETOOTH.println(sleepTime);
				motor_pwm_value = 0;
				delayTime =       100; // reset the delay time 
			}
		}
		analogWrite(MOTOR_PIN, motor_pwm_value);
		max_led_brightness = alertness; 
	}	
	updateCurrentMillis();
	if (currentMillis - updateLEDMillis > delayTime && mode_select != MODE_STATIC_CONTROL) // needs to be ariable that increases as the calmness increases FOR right mode
	{
		updateLEDMillis = millis();
		for (uint8_t i = 0; i < 3; i++)  
		{
			if (rgb_state[i])
			{
				checkTarget(i);
				if (led_pwm[i] < led_target[i])
				{
					led_pwm[i]++;
				}
				else if (led_pwm[i] > 0) 
				{
					led_pwm[i]--;
				}
				analogWrite(rgb_led_pins[i], led_pwm[i]);
			}
		}
	}
	if (stringComplete) 
	{
		char modeSelect = inputString.charAt(0);
		inputString = inputString.substring(1);
		uint8_t led_off_num = 0; 
		/*   
		*    MODE SELECT HERE 
		*/
		switch (modeSelect)
		{


			case 'm':
			switch (inputString.charAt(0))
			{
				case '1':
				mode_select = MODE_STATIC_CONTROL;
				min_led_brightness = 0;
				max_led_brightness = 254;
				//BLUETOOTH.print("\nStatic Control Mode Selected. Continue to enter your desired parameters using the existing API.");
				break;
				case '2':
				mode_select = MODE_LAVA_LAMP;
				delayTime = 100; 
				min_led_brightness = 5;
				max_led_brightness = 254;
				motor_pwm_value = 100;
				analogWrite(MOTOR_PIN, motor_pwm_value);// PWM Freq for motor 
				for (uint8_t i = 0; i < 3 && mode_select != MODE_SLEEP_WITH_LISTENER; i++) 
				{
					if (rgb_state[i])
						checkTarget(i);
				}
				//BLUETOOTH.print("\nLava Lamp Mode Selected. Continue to enter your desired parameters using the existing API.");
				break;
				case '3':  
				mode_select = MODE_SLEEP;
				alertness = 1;
				previousAlertness = 2; 
				case '4':
				min_led_brightness = 0;
				max_led_brightness = 254;
				motor_pwm_value = 100;
				delayTime = 100; 
				sleepTime = 0;
				analogWrite(MOTOR_PIN, motor_pwm_value);// PWM Freq for motor 
				for (uint8_t i = 0; i < 3; i++) 
				{
					if (rgb_state[i])
						checkTarget(i);
				}
				maxPeak = 1.0;
				if (inputString.charAt(0) == '4')
				{
					mode_select = MODE_SLEEP_WITH_LISTENER;
				}
				int timeToSleep = inputString.substring(1, inputString.length()).toInt(); 
				calmScalar = (100 / 12.5) * inputString.substring(1, inputString.length()).toInt(); 
				//BLUETOOTH.print("Seconds til sleep: ");
				//BLUETOOTH.println(timeToSleep);
				//BLUETOOTH.print("calmScalar: ");
				//BLUETOOTH.println(calmScalar);
			}
			break;
			case 'i':
			char buffer[200];
			sprintf(buffer, "\nRGB State: %u%u%u\nAlertness: %u\nRGB PWM: %u, %u, %u\nMotor PWM: %u\nLoop Iteration Time: %u\nRGB Targets: %u, %u, %u\nMaximum Brightness: %u\nMinimum Brightness: %u\nMode: ",\
				rgb_state[0], rgb_state[1], rgb_state[2], alertness, led_pwm[0], led_pwm[1], led_pwm[2], motor_pwm_value, delayTime, led_target[0], led_target[1], led_target[2], max_led_brightness, min_led_brightness);
			Serial.print(buffer);
			switch (mode_select)
			{
				case 1:
				Serial.println("Static Control");
				break;
				case 2:
				Serial.println("Lava Lamp");		
				break;
				case 3:
				Serial.println("Sleep");
				break;
				case 4:
				Serial.println("Sleep w/ Listener");
			}
			break;
			case 'h':		//change motor speed
			Serial.println("Following mode options are a character followed by an int if applicable (ints allow +/- input):");
			Serial.println("\ns: Change motor speed (good values: 80 - 110)");
			Serial.println("t: Loop iteration delay time (good values: 20 - 50 - 250)");
			Serial.println("B: Max LED brightness, must change 't' in accordance to 'B' (good values: 40 - 255)");
			Serial.println("b: Min LED brightness (good value: 10)");
			Serial.println("l: LED IO, 'lnx' where n: r/g/b, and x: 1/0");
			Serial.println("i: Print state information");
			Serial.println("p: show/hide peaks");
			Serial.println("m: Mode select:\n1.	Static Control;\n2.	Lava Lamp;\n3.	Sleep (append number of seconds until lights out);\n4.	Sleep w/ Listener (append number of seconds until lights out).");
			break;			
			case 'l':	
			min_led_brightness = inputString.toInt();
			//BLUETOOTH.print("Turning ");
			uint8_t tempTarget;
			switch (inputString.charAt(0))
			{
				case 'r':
				//BLUETOOTH.print("red ");
				led_off_num = 0;
				break;
				case 'g':
				//BLUETOOTH.print("green ");
				led_off_num = 1;
				break;
				case 'b':
				//BLUETOOTH.print("blue ");
				led_off_num = 2;
			}
			tempTarget = inputString.substring(1, inputString.length()).toInt();
			//BLUETOOTH.print("\ntempTarget: ");
			//BLUETOOTH.println(tempTarget);
			//BLUETOOTH.print("LED ");
				if (inputString.substring(1, inputString.length()).toInt() > 0) // turning LED on  
				{
					//BLUETOOTH.print("on\n");
					if (mode_select == MODE_STATIC_CONTROL)
					{
						led_pwm[led_off_num] = tempTarget;
						analogWrite(rgb_led_pins[led_off_num], led_pwm[led_off_num]);
					}
					else
					{
						led_target[led_off_num] = tempTarget;
					}
					rgb_state[led_off_num] = 1;
				}
				else 									// turning LED off 
				{
					analogWrite(rgb_led_pins[led_off_num], 0);
					rgb_state[led_off_num] = 0;
					led_pwm[led_off_num] = 0;
					led_target[led_off_num] = 0;
					//BLUETOOTH.print("off\n");
				}
				break;
			case 'p': //show/hide peaks
			showPeaks = !showPeaks;
			break;
			case 's':		//change motor speed
			//BLUETOOTH.print("Motor rotation speed: ");
			incDecVar((uint16_t*)&(motor_pwm_value), (char)inputString.charAt(0), (int)inputString.toInt());
			analogWrite(MOTOR_PIN, motor_pwm_value);
			//BLUETOOTH.println(motor_pwm_value);
			break;
			case 't':		//change time
			//BLUETOOTH.print("Loop iteration delay time: ");
			incDecVar(&delayTime, inputString.charAt(0), inputString.toInt());
			//BLUETOOTH.println(delayTime);
			break;
			case 'B':		
			//BLUETOOTH.print("Max LED brightness: ");
			incDecVar((uint16_t*)&max_led_brightness, inputString.charAt(0), inputString.toInt());
			//BLUETOOTH.println(max_led_brightness);
			break;
			case 'b':						
			incDecVar((uint16_t*)&min_led_brightness, inputString.charAt(0), inputString.toInt());
			//BLUETOOTH.print("Min LED brightness: ");
			//BLUETOOTH.println(min_led_brightness);
			break;
			//default:	
			//BLUETOOTH.println("Didn't receive a special character.");
		}
		// clear the string:
		inputString = "";
		stringComplete = false;
	}
	/*
	255 = start
	mode
	red LED value
	green LED Value
	blue LED value
	motor value 
	delayTime
	peak
	*/
	if (stateSendTime > 50)
	{
		for (int i = 0; i < 3; i++)
		{
			if (led_pwm[i] == 255)
				led_pwm[i] = 254; 
		}
		if (motor_pwm_value == 255)
			motor_pwm_value = 254;
		char buffer1[200];
		sprintf(buffer1, "255:%u:%u:%u:%u:%u:%u:%u:",\
		mode_select, led_pwm[0], led_pwm[1], led_pwm[2], motor_pwm_value, delayTime, (int)(100*peakReadFloat));
		BLUETOOTH.print(buffer1);
		// Serial.print(buffer1);
	}
}

 void serialEvent() 
 {
 	while (Serial.available()) 
 	{
 		char inChar = (char)Serial.read();// get the new byte:
 		// Serial.println(inChar);
 		inputString += inChar;// add it to the inputString:
 		// if the incoming character is a newline, set a flag so the main loop can do something about it
 		if (inChar == ',') 
 		{
 			stringComplete = true;
 		}
 	}
 }
