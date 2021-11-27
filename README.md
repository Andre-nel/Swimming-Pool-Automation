//Swimming-Pool-Automation
//Stellenbosch University

//Final Year Project (Scripsie); Mechatronic Engineering
//Author: Christoffel Andre Nel (21784590)

// Import required libraries

#include <WiFi.h>					//websocket, esp_now(station mode)
#include <WebSocketsServer.h>		//websocket

#include <ESPAsyncWebServer.h>		//run webserver, respont to http requests https://github.com/me-no-dev
#include "SPIFFS.h"					

#include <Wire.h>                //enable I2C.
#include <Arduino.h>
#include <ArduinoJson.h>		//Store the last 20 readings in Flash File System, FFS

// OTA includes
// Import required libraries
#include <AsyncTCP.h>
#include <Arduino_JSON.h>
#include <AsyncElegantOTA.h>

//WhatsApp Includes
#include <HTTPClient.h>   //Including HTTPClient.h library is used to post the API with HTTP requests.

//Thingspeak Includes
#include "ThingSpeak.h"

//nRF24L01
#include <SPI.h>
#include <nRF24L01.h>
#include <RF24.h>

RF24 radio(15, 4); // CE, CSN
const byte addresses[][6] = {"00001", "00002"};		//address through which two modules communicate.
//Check write successful
bool writeSuccess;

// Constants						//websocket
const char *ssid = "...-...";   			//Andre 			VodafoneMobileWi.....
const char *password =  ".........";   	//........			........

const int http_port = 888;				//Port 80 dont work on ngrok, used to upload the html file which creates the website
const int ws_port = 1337;			//websocket

const int led_pin = 2;			//Onboard led connected to GPIO pin 2
const int RTD_sleep_pin = 5;    //pH IC Vcc
const int isolators_sleep_pin = 26;    //pin which is connected to isolators enable via N-MOS
const int nRF24L01_sleep_pin = 13;    //nRF24L01 Vcc pin

//Sleep
int NumCyclesPerDay = 3;
int NumberOfMonitoredParameters = 4;
int timerDelay = 60; 						//60 Sec time between measurements of different parameters, 60 Sec, for ThingSpeak
int Delay_DoseInfo_TS = 20; 				//17 Seconds between dosing updates sent to website and ThingSpeak
int BufferTimeBeforeDeepSleep = 120;		//360 Dosing system has seconds to reply to monitoring system to update interfaces before entering deep sleep

int CycleDuration = NumberOfMonitoredParameters*timerDelay+BufferTimeBeforeDeepSleep;

unsigned long DaySleepDuration = 43200/(NumCyclesPerDay-1) - BufferTimeBeforeDeepSleep - CycleDuration;		//21360 (Seconds)Duration between reading cycles in the day
unsigned long DeepSleepDuration = 43200 - BufferTimeBeforeDeepSleep - CycleDuration;		//42960 - BufferTimeBeforeDeepSleep; deep sleep until the next morning, roughly 12 hours

RTC_DATA_ATTR int ReadingCycleCTR = 0;		//stores number of reading cycles, in RTC memory, retained during deepsleep
unsigned long DaySleepStartTime = 0;	
int DaySleepFlag = 0;
int SystemsActive = 1;

RTC_DATA_ATTR int SystemDaysCTR = 0;		//stores number of days system has been running for, in the RTC memory
unsigned long SecondsInADay = 86400;
unsigned long DeepSleepFlagSetTime = 0;	
int DeepSleepFlag = 0;
#define uS_TO_S_FACTOR 1000000ULL  			// Conversion factor for micro seconds to seconds

int NumReadingsPerCycle = 1;	//The system will update The website and Thingspeak this number of times each time it wakes up.


// Timer variables
unsigned long lastTime = 0;
int time_ = 900;                 //used to change the delay needed depending on the command sent to the EZO Class RTD Circuit.


// Globals
AsyncWebServer server(http_port);							//Run a webserver from ESP32
WebSocketsServer webSocket = WebSocketsServer(ws_port);		//tell the server to listen on ws_port, 1337
char msg_buf[40];
int led_state = 0;
uint8_t client_num_store = 0;

int address = 100;                  //EZO RTD Circuit address, set when device configured into I2C mode.
byte code = 0;                   //used to hold the EZO I2C response code.
char i_c_command[20] ="r";         //hold request sent to ezo circuits
char measured_data[20];         //hold incoming data from the EZO circuits.

float CurrentTemp = 25;			//default value
float CurrentPH;
float CurrentORP;

String measured_data_json = "{\"value\":";
String stored_data_json = "{\"value\":";

byte in_char = 0;                //used as a 1 byte buffer to store in bound bytes from the EZO Circuits.
byte i = 0;                      //counter used for RTD_data array.
float float_value;                 //float var used to hold the float value of the RTD
char miscellaneous_buff[20];		//Buffer for random unimportant use


int VoltageTaken = 0;			//Flags for controlling the timming, ThingSpeak requires 15 sec btw uploads for free accounts
int TempTaken = 0;
int pHTaken = 0;
int ORPTaken = 0;


//VARIABLES used for SPIFFS storage
const char* filename = "/data_file.json"; 	//Look in the data folder at the file.json file
const String JsonTempVarNames[] = {"RTD_1", "RTD_2", "RTD_3", "RTD_4", "RTD_5", "RTD_6", "RTD_7", "RTD_8", "RTD_9", "RTD_10", "RTD_11", "RTD_12", "RTD_13", "RTD_14", "RTD_15", "RTD_16", "RTD_17", "RTD_18", "RTD_19", "RTD_20"};
float JsonTempValues[20] = {0};
const String JsonpHVarNames[] = {"pH_1", "pH_2", "pH_3", "pH_4", "pH_5", "pH_6", "pH_7", "pH_8", "pH_9", "pH_10", "pH_11", "pH_12", "pH_13", "pH_14", "pH_15", "pH_16", "pH_17", "pH_18", "pH_19", "pH_20"};
float JsonpHValues[20] = {0};
const String JsonORPVarNames[] = {"ORP_1", "ORP_2", "ORP_3", "ORP_4", "ORP_5", "ORP_6", "ORP_7", "ORP_8", "ORP_9", "ORP_10", "ORP_11", "ORP_12", "ORP_13", "ORP_14", "ORP_15", "ORP_16", "ORP_17", "ORP_18", "ORP_19", "ORP_20"};
float JsonORPValues[20] = {0};
const String JsonVoltageVarNames[] = {"Voltage_1", "Voltage_2", "Voltage_3", "Voltage_4", "Voltage_5", "Voltage_6", "Voltage_7", "Voltage_8", "Voltage_9", "Voltage_10", "Voltage_11", "Voltage_12", "Voltage_13", "Voltage_14", "Voltage_15", "Voltage_16", "Voltage_17", "Voltage_18", "Voltage_19", "Voltage_20"};
float JsonVoltageValues[20] = {0};

const String JsonAcidDoseVarNames[] = {"AD_1", "AD_2", "AD_3", "AD_4", "AD_5", "AD_6", "AD_7", "AD_8", "AD_9", "AD_10", "AD_11", "AD_12", "AD_13", "AD_14", "AD_15", "AD_16", "AD_17", "AD_18", "AD_19", "AD_20"};
float JsonAcidDoseValues[20] = {0};
const String JsonAlkalinityUpDoseVarNames[] = {"AUD_1", "AUD_2", "AUD_3", "AUD_4", "AUD_5", "AUD_6", "AUD_7", "AUD_8", "AUD_9", "AUD_10", "AUD_11", "AUD_12", "AUD_13", "AUD_14", "AUD_15", "AUD_16", "AUD_17", "AUD_18", "AUD_19", "AUD_20"};
float JsonAlkalinityUpDoseValues[20] = {0};
const String JsonChlorineDoseVarNames[] = {"CD_1", "CD_2", "CD_3", "CD_4", "CD_5", "CD_6", "CD_7", "CD_8", "CD_9", "CD_10", "CD_11", "CD_12", "CD_13", "CD_14", "CD_15", "CD_16", "CD_17", "CD_18", "CD_19", "CD_20"};
float JsonChlorineDoseValues[20] = {0};
float buff;

String var_names[141];
int num_elements = 141;
float values[141];

float Reading;

//Battery Reading Variables
#define MAX17043_ADDRESS 0x36  // R/W =~ 0x6D/0x6C
// Global Variables
float batVoltage;
String batVoltageString;
float batPercentage;
int alertStatus = 0;			//Flag set to 1 when the battery is getting low

//WhatsApp Variables
String apiKey = "..............";              //Add your Token number that bot has sent you on WhatsApp messenger
String phone_number = "+27..........7"; //Add your WhatsApp app registered phone number (same number that bot send you in url)
String WhatsApp_url;                    //url String will be used to store the final generated URL

//ThingSpeak variables
WiFiClient  client;

int channelID = ..........;                  //channel number in ThingSpeak
char* writeAPIKey = "................";
char* ReadAPIKey = "................."; 

float VoltageRead;


//Structure to send data		//Must match the receiver structure
typedef struct struct_send_message {
	float verification;
    float pH;
    float ORP;
	float PoolVolume;
} struct_send_message;
//struct_send_message to hold sensor readings
struct_send_message msgTooDosingStation_struct;

//Structure to Receive data		//Must match the Send structure
typedef struct struct_rec_message {
	float verification;
	float AcidCtr;
	float ChlorineCtr;
	float AlkalinityUpCtr;
	float ChlorineLockFlag;
	float DosingKey;
	float pHDoseIndicator;
	float ChlorineDoseIndicator;
} struct_rec_message;
// Create a struct_rec_message to hold incoming messages
struct_rec_message msgFromDosingStation_struct;

  float AcidCtr = 0;
  float ChlorineCtr = 0;
	float AlkalinityUpCtr = 0;
	float ChlorineLockFlag = 0;
	float DosingKey = 0;
	
float ChlorineLockFlag2 = 0;

//Default swimming pool Volume
float PoolVolume = 10000;
int PoolVolumeKey = 0;			//key to indicate data rate will be sent next
const String PoolVolumeName[] = {"PoolVolume"};

float AcidVolumeRatio = 0.0036666;
float AlkalinityUpVolumeRatio = 0.011666;
float ChlorineVolumeRatio = 0.013525;

//Variables hold the respective dosing volumes added by dosing system pumps
float AcidAdded;
float AlkalinityUpAdded;
float ChlorineAdded;

//Hold the reply from the dosing system indicating which pumps are activated, what chemicals added
int pHDoseIndicator;
int ChlorineDoseIndicator;

/****************************************************************************************************************************************************************************************************************
 * Functions
 */
/********************************************************************************************************************************************************************************************************/
 // Functions for more efficient code
 //I2C Transmission
float TakeParameterReading(int address, char* i_c_command){
  Wire.beginTransmission(address);                  //call the circuit by its address.
  Wire.write(i_c_command);                          //transmit the command that was sent through the serial port.
  Wire.endTransmission();                           //end the I2C data transmission.

  delay(time_);                                     //wait the correct amount of time for the circuit to complete its instruction.

  Wire.requestFrom(address, 20, 1);                 //call the circuit and request 20 bytes (this may be more than need)
  code = Wire.read();                               //the first byte is the response code, read this separately.
  switch (code) {                       //switch case based on what the response code is.
    case 1:                             
      //Serial.print("Success");        //means the command was successful.
      //webSocket.sendTXT(client_num_store, "Debug: Successfull sensor reading:)");   //send to website for debug, when running off battery
      break;                            //exits the switch case.

    case 2:                             
      Serial.println("Failed");         //means the command has failed.
      webSocket.sendTXT(client_num_store, "Debug: Command has failed, could be due to the wrong command being sent");   //send to website for debug, when running off battery
      break;                            //exits the switch case.

    case 254:                           
      Serial.println("Pending");        //means the command has not yet been finished calculating.
      webSocket.sendTXT(client_num_store, "Debug: Pending, command has not yet been finished calculating");   //send to website for debug, when running off battery
      break;                            //exits the switch case.

    case 255:                           
      Serial.println("No Data");        //means there is no further data to send.
      webSocket.sendTXT(client_num_store, "Debug: NO Data");    //send to website for debug, when running off battery
      break;                            //exits the switch case.
  }
  
  if(code == 1){
  while (Wire.available()) {          //are there bytes to receive.
      in_char = Wire.read();              //receive a byte.
      measured_data[i] = in_char;         //load this byte into our array.
      i += 1;                             //incur the counter for the array element.
      if (in_char == 0) {                 //been sent a null command.
        i = 0;                            //reset the counter i to 0.
        break;                            //exit the while loop.
      }
    }
    return atof(measured_data);
  }else {return 0;}
}

void SendAcidDosageToWebsite(){
  measured_data_json = "{\"value\":";   //Reset json string
  Serial.printf("\nHTML: AcidDosage = %.2f V\n", AcidAdded);
  measured_data_json += (String)AcidAdded;
  measured_data_json += "}";
  webSocket.sendTXT(client_num_store, "AcidDosage_update");                   //Send Key to website
  webSocket.broadcastTXT(measured_data_json.c_str(), measured_data_json.length());    //Send json formatted reading to website
}

void SendAlkalinityUpDosageToWebsite(){
  measured_data_json = "{\"value\":";   //Reset json string
  Serial.printf("HTML: AlkalinityUpDosage = %.2f (ml)\n", AlkalinityUpAdded);
  measured_data_json += (String)AlkalinityUpAdded;
  measured_data_json += "}";
  webSocket.sendTXT(client_num_store, "AlkalinityUpDosage_update");                   //Send Key to website
  webSocket.broadcastTXT(measured_data_json.c_str(), measured_data_json.length());    //Send json formatted reading to website
}

void SendChlorineDosageToWebsite(){
  measured_data_json = "{\"value\":";
  Serial.printf("HTML: ChlorineDosage = %.2f (ml)\n", ChlorineAdded);
  measured_data_json += (String)ChlorineAdded;
  measured_data_json += "}";
  webSocket.sendTXT(client_num_store, "ChlorineDosage_update");                   //Send Key to website
  webSocket.broadcastTXT(measured_data_json.c_str(), measured_data_json.length());    //Send json formatted reading to website
}


void SendBatteryVoltageToWebsite(){
  measured_data_json = "{\"value\":";   //Reset json string
  Serial.printf("\nHTML: Voltage = %.2f V\n", batVoltage);
  measured_data_json += (String)batVoltage;
  measured_data_json += "}";
  webSocket.sendTXT(client_num_store, "batVoltage_update");                   //Send Key to website batVoltage_update
  webSocket.broadcastTXT(measured_data_json.c_str(), measured_data_json.length());    //Send json formatted reading to website
}

void SendTemperatureToWebsite(float Temp){
  measured_data_json = "{\"value\":";   //Reset json string
  Serial.printf("HTML: Temperature = %s (C)\n", (String)Temp);
  measured_data_json += (String)Temp;
  measured_data_json += "}";
  webSocket.sendTXT(client_num_store, "Temp_update");                 //Send key to website
  webSocket.broadcastTXT(measured_data_json.c_str(), measured_data_json.length());  //Send json formatted reading to website
} 

void SendPHToWebsite(float pH){
  measured_data_json = "{\"value\":";   //Reset json string
  Serial.printf("HTML: pH = %s\n", (String)pH);
  measured_data_json += (String)pH;
  measured_data_json += "}";
  webSocket.sendTXT(client_num_store, "pH_update");                   //Send key to website
  webSocket.broadcastTXT(measured_data_json.c_str(), measured_data_json.length());    //Send json formatted reading to website
} 

void SendORPToWebsite(float ORP){
  measured_data_json = "{\"value\":";   //Reset json string
  Serial.printf("HTML: ORP = %s (mV)\n", (String)ORP);
  measured_data_json += (String)ORP;
  measured_data_json += "}";
  webSocket.sendTXT(client_num_store, "ORP_update");                    //Send Key to website
  webSocket.broadcastTXT(measured_data_json.c_str(), measured_data_json.length());    //Send json formatted reading to website
} 

float takeAverageReading(int address, char* i_c_command){               //Take Reading 5 valid times, then average it.
  float AverageArray[5];
  for(int i = 0; i<5; i++)  
    {
      float Reading = TakeParameterReading(address, i_c_command);
      if(Reading != 0)
      {
        AverageArray[i] = Reading;
      }else i -= 1;
    }
  float AverageReading = (AverageArray[0] + AverageArray[1] + AverageArray[2] + AverageArray[3] + AverageArray[4])/5;
  return AverageReading;
}

float takeAverageTempReading(){
  return takeAverageReading(100, "r");
}

float takeAveragePHReading(float Temp){
  char i_c_comm[20];
  sprintf(i_c_comm, "rt,%.2f", Temp);
  float ans = takeAverageReading(101, i_c_comm);
  return ans;
}

float takeAverageORPReading(){
  return takeAverageReading(102, "r");
}

/********************************************************************************************************************************************************************************************************/
 // RF functions

void ProcessReadingsReceived(){
	Display_Readings_Received();
	if(msgFromDosingStation_struct.verification == 1){
		Serial.println("\nData received from Dosing system uploaded into variables");
		AcidCtr = msgFromDosingStation_struct.AcidCtr;
		ChlorineCtr = msgFromDosingStation_struct.ChlorineCtr;
		AlkalinityUpCtr = msgFromDosingStation_struct.AlkalinityUpCtr;
		ChlorineLockFlag = msgFromDosingStation_struct.ChlorineLockFlag;
		DosingKey = msgFromDosingStation_struct.DosingKey;								//Indicates what triggered the message
		pHDoseIndicator = (int)msgFromDosingStation_struct.pHDoseIndicator;
		ChlorineDoseIndicator = (int)msgFromDosingStation_struct.ChlorineDoseIndicator;
	}
}

void ReactOnReadingsReceived(){
	if(DosingKey == 5){
		switch(pHDoseIndicator) {
			case 1:
     delay(Delay_DoseInfo_TS*1000);
				Serial.println("Dosing system Adding Acid, Update ThingSpeak and Webpage");
				//Update Webpage
				SendAcidDosageToWebsite();
				//Update ThingSpeak
				writeTSData(channelID, 5, AcidAdded, writeAPIKey);
				//Write Data to array which will be transfered to json file:
				UpdateArray(JsonAcidDoseValues, AcidAdded);
				break;
			case 2:
     delay(Delay_DoseInfo_TS*1000);
				Serial.println("Dosing system Adding Alkalinity Up, Update ThingSpeak and Webpage");
				//Update Webpage
				SendAlkalinityUpDosageToWebsite();
				//Update ThingSpeak
				writeTSData(channelID, 6, AlkalinityUpAdded, writeAPIKey);
				//Write Data to array which will be transfered to json file:
				UpdateArray(JsonAlkalinityUpDoseValues, AlkalinityUpAdded);
				break;
			default:
				break;
		}
		switch(ChlorineDoseIndicator) {
			case 1:
     delay(Delay_DoseInfo_TS*1000);
				Serial.println("Dosing system Adding Chlorine, Update ThingSpeak and Webpage");
				//Update Webpage
				SendChlorineDosageToWebsite();
				//Update ThingSpeak
				writeTSData(channelID, 7, ChlorineAdded, writeAPIKey);
				//Write Data to array which will be transfered to json file:
				UpdateArray(JsonChlorineDoseValues, ChlorineAdded);
				break;
			default:
				break;
		}
		storeReadingsInFFs();	//Save Dosing Info
	}
	if(DosingKey == 4){
		if(ChlorineLockFlag){
			Serial.println("The swimming pool has entered Chlorine Lock!,\n After "+ String(ChlorineCtr) + " Chlorine Doses were made with no effect");
			Serial.println("No Chlorine will be added until the exits chlorine Lock\n");
			ChlorineLockFlag2 = 1;		//Send WhatsApp message at the end of the day
		}else{
			Serial.println("The swimming pool has escaped Chlorine Lock :)\n");
			ChlorineLockFlag2 = 2;		//Send WhatsApp message at the end of the day
		}
	}

 
}
 
void Display_Readings_sent(){
	Serial.printf("pH sent = %f\n", msgTooDosingStation_struct.pH);
	Serial.printf("ORP sent = %f\n", msgTooDosingStation_struct.ORP);
	Serial.printf("PoolVolume sent = %f\n\n", msgTooDosingStation_struct.PoolVolume);
}

void Display_Readings_Received(){
	Serial.printf("\nverification received = %f\n", msgFromDosingStation_struct.verification);
	Serial.printf("AcidCtr received = %f\n", msgFromDosingStation_struct.AcidCtr);
	Serial.printf("ChlorineCtr received = %f\n", msgFromDosingStation_struct.ChlorineCtr);
	Serial.printf("AlkalinityUpCtr received = %f\n", msgFromDosingStation_struct.AlkalinityUpCtr);
	Serial.printf("ChlorineLockFlag received = %f\n", msgFromDosingStation_struct.ChlorineLockFlag);
	Serial.printf("DosingKey received = %f\n", msgFromDosingStation_struct.DosingKey);
	Serial.printf("pHDoseIndicator received = %f\n", msgFromDosingStation_struct.pHDoseIndicator);
	Serial.printf("ChlorineDoseIndicator received = %f\n", msgFromDosingStation_struct.ChlorineDoseIndicator);
}
 
 //************************************************************************************************************************************************************************************************
 //WhatApp Functions
 /*
  Date: 25-06-21
  Code adjusted from code written by: Dharmik
  Find more on www.TechTOnions.com
*/ 
 void  message_to_whatsapp(String message)       //function to send meassage to WhatsApp
{
  //adding all number, your api key, your message into one complete url
  WhatsApp_url = "https://api.callmebot.com/whatsapp.php?phone=" + phone_number + "&apikey=" + apiKey + "&text=" + WhatsApp_url_encode(message);

  postData(); // calling postData to run the above-generated url once so that you will receive a message.
}

String WhatsApp_url_encode(String str)  // Function used for encoding the WhatsApp_url
{
    String encodedString="";
    char c;
    char code0;
    char code1;
    char code2;
    for (int i =0; i < str.length(); i++){
      c=str.charAt(i);
      if (c == ' '){
        encodedString+= '+';
      } else if (isalnum(c)){
        encodedString+=c;
      } else{
			code1=(c & 0xf)+'0';
			if ((c & 0xf) >9){
				code1=(c & 0xf) - 10 + 'A';
			}
			c=(c>>4)&0xf;
			code0=c+'0';
			if (c > 9){
				code0=c - 10 + 'A';
			}
			code2='\0';
			encodedString+='%';
			encodedString+=code0;
			encodedString+=code1;
			//encodedString+=code2;
		  }
      yield();
    }
    return encodedString;
}

void postData()     //userDefine function used to call api(POST data)
{
  int httpCode;     // variable used to get the responce http code after calling api
  HTTPClient WhatsApp_http;  // Declare object of class HTTPClient
  WhatsApp_http.begin(WhatsApp_url);  // begin the HTTPClient object with generated url
  
  httpCode = WhatsApp_http.POST(WhatsApp_url); // Finaly Post the URL with this function and it will store the http code
  if (httpCode == 200)      // Check if the responce http code is 200
  {
    Serial.println("WatsApp Sent ok."); // print message sent ok message
  }
  else                      // if response HTTP code is not 200 it means there is some error.
  {
    Serial.println("WatsApp Error."); // print error message.
  }
  WhatsApp_http.end();          // After calling API end the HTTP client object.
}
 
 
 
 //**************************************************************************************************************************************************************
//Battery Reading Functions
/*
BatteryVoltage() returns a 12-bit ADC reading of the battery voltage,
as reported by the MAX17043's VCELL register.
This does not return a voltage value. To convert this to a voltage,
multiply by 5 and divide by 4096.
*/
float BatteryVoltage()
{
  float voltage;
  unsigned int vcell;

  vcell = i2cRead16(0x02);
  vcell = vcell >> 4;  // last 4 bits of vcell are nothing
  voltage = (float) vcell * 1/800;
  return voltage;
}

/*
BatteryPercentage() returns a float value of the battery percentage
reported from the SOC register of the MAX17043.
*/
float BatteryPercentage()
{
  unsigned int soc;
  float percent;

  soc = i2cRead16(0x04);  // Read SOC register of MAX17043
  percent = (byte) (soc >> 8);  // High byte of SOC is percentage
  percent += ((float)((byte)soc))/256;  // Low byte is 1/256%

  return percent;
}

/* 
RestartFuelGauge() issues a quick-start command to the MAX17043.
A quick start allows the MAX17043 to restart fuel-gauge calculations
in the same manner as initial power-up of the IC. If an application's
power-up sequence is very noisy, such that excess error is introduced
into the IC's first guess of SOC, the Arduino can issue a quick-start
to reduce the error.
*/
void RestartFuelGauge()
{
  i2cWrite16(0x4000, 0x06);  // Write a 0x4000 to the MODE register
}

/* 
i2cRead16(unsigned char address) reads a 16-bit value beginning
at the 8-bit address, and continuing to the next address. A 16-bit
value is returned.
*/
unsigned int i2cRead16(unsigned char address)
{
  int data = 0;

  Wire.beginTransmission(MAX17043_ADDRESS);
  Wire.write(address);
  Wire.endTransmission();

  Wire.requestFrom(MAX17043_ADDRESS, 2);
  while (Wire.available() < 2)
    ;
  data = ((int) Wire.read()) << 8;
  data |= Wire.read();

  return data;
}

/*
i2cWrite16(unsigned int data, unsigned char address) writes 16 bits
of data beginning at an 8-bit address, and continuing to the next.
*/
void i2cWrite16(unsigned int data, unsigned char address)
{
  Wire.beginTransmission(MAX17043_ADDRESS);
  Wire.write(address);
  Wire.write((byte)((data >> 8) & 0x00FF));
  Wire.write((byte)(data & 0x00FF));
  Wire.endTransmission();
}

//Take battery readings	function
void takeBatteryReadings(){
	batVoltage = BatteryVoltage();  // vcell reports battery voltage 263.272727
	batVoltageString = (String)batVoltage;
	
	//sprintf(msg_buf, "Battery Voltage = %.2f V", batVoltage);  //Combines text and variables and stores it in the buffer, 1st parameter
	
	//Serial.printf("%s\n", msg_buf);	//abcd
}		

 //*************************************************************************************************************************************************************************************************************************
//Sleep function
void goToDeepSleep(){
	esp_sleep_enable_timer_wakeup(DeepSleepDuration * uS_TO_S_FACTOR);	//timer expects one argument, represents sleep time in micro seconds
	Serial.println("ESP32 enters deep sleep for " + String(DeepSleepDuration) + " Seconds");
	Serial.flush();
	esp_deep_sleep_start();
}	
 
 //*****************************************************************************************************************************************************************************************************************************
//Websocket
//create a callback function which is created inside the websockets library
//the libary is looking for a function, any name, with specific parameters
// Callback: receiving any WebSocket message
void onWebSocketEvent(uint8_t client_num,	//client number, think it can handle up to 5 clients
                      WStype_t type,		//type of message received
                      uint8_t* payload,		//data received in bytes
                      size_t length) {		//number of bytes in payload
						  
	client_num_store = client_num;	//update to whichever client active most recently

  // Figure out the type of WebSocket event
	switch(type) {

		// Client has disconnected
		case WStype_DISCONNECTED:
		  Serial.printf("[%u] Disconnected!\n", client_num);
		  break;

		// New client has connected
		case WStype_CONNECTED:
		  {
			IPAddress ip = webSocket.remoteIP(client_num);
			Serial.printf("[%u] Connection from ", client_num);
			Serial.println(ip.toString());
		  }
		  break;

		// Handle text messages from client
		case WStype_TEXT:

			// Print out raw message
			Serial.printf("\n[%u] Received text: %s\n", client_num, payload);

			// Toggle LED
			if ( strcmp((char *)payload, "toggleLED") == 0 ) {
				led_state = !led_state;
				Serial.printf("Toggling LED to %u\n", led_state);
				digitalWrite(led_pin, led_state);

			  // Report the state of the LED
			} else if ( strcmp((char *)payload, "getLEDState") == 0 ) {
				sprintf(msg_buf, "%d", led_state);  //Combines text and variables and stores it in the buffer, 1st parameter
				Serial.printf("Sending to [%u]: %s\n", client_num, msg_buf);
				webSocket.sendTXT(client_num, "LED_update");
				webSocket.sendTXT(client_num, msg_buf);

			//PoolVolumeKey received
			} else if( strcmp((char *)payload, "PoolVolume") == 0 ) {
				PoolVolumeKey = 1;										//Set PoolVolumeKey flag
				Serial.printf("Set PoolVolumeKey to %d\n", PoolVolumeKey);

			//PoolVolume
			}else if((PoolVolumeKey == 1) && (strcmp((char *)payload, "PoolVolume") != 0)){
				PoolVolume = (float) atof((const char *)&payload[0]);
				Serial.printf("Changed the stored Pool Volume to %f\n", PoolVolume);
				PoolVolumeKey = 0;										//Reset PoolVolumeKey flag
				storeReadingsInFFs();									//Save pool volume
				AssignDosingVolumes();									//Assign new dosing volumes

			//Send the stored json data in FFS, after new website connection, when received "SendJsonFFS"
			}else if( strcmp((char *)payload, "SendJsonFFS") == 0 ){
				Serial.println("FFS json data Requested");
				retrieveAndSendReadingsFromFFs();												//Retrieve and send latest readings to website
				
			//Message not recognized
			}else {
				Serial.println("[%u] Message not recognized");
			}
			break;

		// For everything else: do nothing
		case WStype_BIN:
		case WStype_ERROR:
		case WStype_FRAGMENT_TEXT_START:
		case WStype_FRAGMENT_BIN_START:
		case WStype_FRAGMENT:
		case WStype_FRAGMENT_FIN:
		default:
			break;
	}
}

//***********************************************************************************************************************************************************************************************************************************
//Website callback functions
// Callback: send homepage
void onIndexRequest(AsyncWebServerRequest *request) {
  IPAddress remote_ip = request->client()->remoteIP();
  Serial.println("[" + remote_ip.toString() +
                  "] HTTP GET request of " + request->url());
  request->send(SPIFFS, "/index.html", "text/html");
}

// Callback: send style sheet
void onCSSRequest(AsyncWebServerRequest *request) {
  IPAddress remote_ip = request->client()->remoteIP();
  Serial.println("[" + remote_ip.toString() +
                  "] HTTP GET request of " + request->url());
  request->send(SPIFFS, "/style.css", "text/css");
}

// Callback: send 404 if requested file does not exist
void onPageNotFound(AsyncWebServerRequest *request) {
  IPAddress remote_ip = request->client()->remoteIP();
  Serial.println("[" + remote_ip.toString() +
                  "] HTTP GET request of " + request->url());
  request->send(404, "text/plain", "Not found");
}

//***********************************************************************************************************************************************************************************************************************************
//SPIFFS
//List all files in SPI flash file system (don't read data)
void listAllFiles(){
	Serial.println("Called: listAllFiles()");
  File root = SPIFFS.open("/"); //open acces to files
  File file = root.openNextFile();
  Serial.println("Current files in FFS:");
  while(file){
      Serial.print("File: ");
      Serial.println(file.name());
      file = root.openNextFile();
    }
  root.close();   //Close access to the files
  file.close();
}

//Read JSON data from specific var in a file, "file.json":
float readDataFromFile(const char* filename, const String* var_name){
  File file = SPIFFS.open(filename);
  if(file){
    //Deserialize data:
    StaticJsonDocument<5000> doc;
    DeserializationError error = deserializeJson(doc,file);
    float var_read = doc[*var_name]; //Read from variable
    //Serial.printf("Read %s = %f \n", var_name, var_read, filename);
  return var_read;
  }
  file.close();
}

//open and rewrite file, pre-existing contents are lost
void rewriteFile(const char* filename, const String* var_names, float* var_values, int num_elements){
//  Serial.println("Called: rewriteFile()");
  File outfile = SPIFFS.open(filename, "w"); 						//open file for write
  StaticJsonDocument<5000> doc;  									//create static json doc
  for(int i=0; i<num_elements; i++){
    doc[var_names[i]] = var_values[i];                         		//write the values to their corresponding variables
	//Serial.printf("wrote %s = %f\n", var_names[i], var_values[i]);	//print to serial
  }
  if(serializeJson(doc, outfile)==0){                 				//Store StaticJsonDocument "Doc" in filename
    Serial.println("Failed to write to SPIFFS file");
  } 
  outfile.close();
}

//read whole file contents onto serial monitor
void  readFile(const char* filename){
	Serial.println("Called: readFile()");
  File fileToRead = SPIFFS.open(filename);
  if(!fileToRead){
      Serial.println("Failed to open file for reading");
      return;}
  Serial.println("File Content:");
  while(fileToRead.available()){
      Serial.write(fileToRead.read());
  }
  fileToRead.close();
}

//Write Data to json file:
void storeReadingsInFFs(){
//	Serial.println("Called: storeReadingsInFFs()");
	//Nested for loop
	for(int m = 0; m < 7; m++){		//For loop for going through: Temp, PH, ORP, BatteryVoltage, AcidDose, AlkalinityUpDose, ChlorineDose
		for(int n = 0; n < 20; n++){		//For loop for going through the 20 stored readings of each Type reading
			if(m == 0){			//Temperature
				values[(m*20) + n] = JsonTempValues[n];
				
			}else if(m == 1){	//pH
				values[(m*20) + n] = JsonpHValues[n];
				
			}else if(m == 2){	//ORP
				values[(m*20) + n] = JsonORPValues[n];
				
			}else if(m == 3){	//Voltage
				values[(m*20) + n] = JsonVoltageValues[n];
				
			}else if(m == 4){	//AcidDose
				values[(m*20) + n] = JsonAcidDoseValues[n];
				
			}else if(m == 5){	//AlkalinityUpDose
				values[(m*20) + n] = JsonAlkalinityUpDoseValues[n];
				
			}else if(m == 6){	//ChlorineDose
				values[(m*20) + n] = JsonChlorineDoseValues[n];
			}
		}
	}
	values[140] = PoolVolume;
	var_names[140] = "PoolVolume";
	
	rewriteFile(filename, var_names, values, num_elements);
}

//Read Data from json file and send it to website:
void retrieveAndSendReadingsFromFFs(){
	Serial.println("Called: retrieveAndSendReadingsFromFFs()");
	//Nested for loop
	for(int m = 0; m < 7; m++){		//For loop for going through: Temp, PH, ORP, Battery Voltage, AcidDose, AlkalinityUpDose, ChlorineDose
		for(int n = 0; n < 20; n++){		//For loop for going through the 20 stored readings of each Type reading
			if(m == 0){
				delay(10);
				buff = readDataFromFile(filename, &JsonTempVarNames[n]);	//read Temp var from json file
				stored_data_json = "{\"value\":";		//Reset json string
				stored_data_json += (String)buff;
				stored_data_json += "}";
				webSocket.sendTXT(client_num_store, "Temp_update");										//Send Key to website
				webSocket.broadcastTXT(stored_data_json.c_str(), stored_data_json.length());		//Send json formatted reading to website
				
				//Serial.print("Sending stored temp: ");
				//Serial.println(stored_data_json);
				
			}else if(m == 1){
				delay(10);
				buff = readDataFromFile(filename, &JsonpHVarNames[n]);	//read pH var from json file
				stored_data_json = "{\"value\":";		//Reset json string
				stored_data_json += (String)buff;
				stored_data_json += "}";
				webSocket.sendTXT(client_num_store, "pH_update");										//Send Key to website
				webSocket.broadcastTXT(stored_data_json.c_str(), stored_data_json.length());		//Send json formatted reading to website
				
			}else if(m == 2){
				delay(10);
				buff = readDataFromFile(filename, &JsonORPVarNames[n]);		//read ORP var from json file
				stored_data_json = "{\"value\":";		//Reset json string
				stored_data_json += (String)buff;
				stored_data_json += "}";
				webSocket.sendTXT(client_num_store, "ORP_update");			//Send Key to website
				webSocket.broadcastTXT(stored_data_json.c_str(), stored_data_json.length());		//Send stored json formatted reading to website
				
			}else if(m == 3){
				delay(10);
				buff = readDataFromFile(filename, &JsonVoltageVarNames[n]);		//read Voltage var from json file
				stored_data_json = "{\"value\":";		//Reset json string
				stored_data_json += (String)buff;
				stored_data_json += "}";
				webSocket.sendTXT(client_num_store, "batVoltage_update");			//Send Key to website //batVoltage_update
				webSocket.broadcastTXT(stored_data_json.c_str(), stored_data_json.length());		//Send stored json formatted reading to website

			}else if(m == 4){
				delay(10);
				buff = readDataFromFile(filename, &JsonAcidDoseVarNames[n]);		//read AcidDose var from json file
				stored_data_json = "{\"value\":";		//Reset json string
				stored_data_json += (String)buff;
				stored_data_json += "}";
				webSocket.sendTXT(client_num_store, "AcidDosage_update");			//Send Key to website 
				webSocket.broadcastTXT(stored_data_json.c_str(), stored_data_json.length());		//Send stored json formatted reading to website

			}else if(m == 5){
				delay(10);
				buff = readDataFromFile(filename, &JsonAlkalinityUpDoseVarNames[n]);		//read AlkalinityUpDose var from json file
				stored_data_json = "{\"value\":";		//Reset json string
				stored_data_json += (String)buff;
				stored_data_json += "}";
				webSocket.sendTXT(client_num_store, "AlkalinityUpDosage_update");			//Send Key to website 
				webSocket.broadcastTXT(stored_data_json.c_str(), stored_data_json.length());		//Send stored json formatted reading to website
				
			}else if(m == 6){
				delay(10);
				buff = readDataFromFile(filename, &JsonChlorineDoseVarNames[n]);		//read ChlorineDose var from json file
				stored_data_json = "{\"value\":";		//Reset json string
				stored_data_json += (String)buff;
				stored_data_json += "}";
				webSocket.sendTXT(client_num_store, "ChlorineDosage_update");			//Send Key to website
				webSocket.broadcastTXT(stored_data_json.c_str(), stored_data_json.length());		//Send stored json formatted reading to website
			}
		}
	}
	buff = readDataFromFile(filename, &PoolVolumeName[0]);		//read "PoolVolume" var from json file
	stored_data_json = "{\"value\":";		//Reset json string
	stored_data_json += (String)buff;
	stored_data_json += "}";
	webSocket.sendTXT(client_num_store, "PoolVolume_update");			//Send Key to website 
	webSocket.broadcastTXT(stored_data_json.c_str(), stored_data_json.length());		//Send stored json formatted reading to website
}

//Read Data from json file and store it in arrays
void retrieveFFSDataAndStoreInArrays(){
//	Serial.println("Called: retrieveFFSDataAndStoreInArrays()");
	//Nested for loop
	for(int m = 0; m < 7; m++){		//For loop for going through: Temp, PH, ORP, voltage, AcidDose, AlkalinityUpDose, ChlorineDose
		for(int n = 0; n < 20; n++){		//For loop for going through the 20 stored readings of each Type reading
			if(m == 0){
				JsonTempValues[n] = readDataFromFile(filename, &JsonTempVarNames[n]);	//read Temp var from json file
				
			}else if(m == 1){
				JsonpHValues[n] = readDataFromFile(filename, &JsonpHVarNames[n]);		//read pH var from json file
				
			}else if(m == 2){
				JsonORPValues[n] = readDataFromFile(filename, &JsonORPVarNames[n]);		//read ORP var from json file
			
			}else if(m == 3){
				JsonVoltageValues[n] = readDataFromFile(filename, &JsonVoltageVarNames[n]);		//read battery voltage var from json file
				
			}else if(m == 4){
				JsonAcidDoseValues[n] = readDataFromFile(filename, &JsonAcidDoseVarNames[n]);		//read AcidDose var from json file
				
			}else if(m == 5){
				JsonAlkalinityUpDoseValues[n] = readDataFromFile(filename, &JsonAlkalinityUpDoseVarNames[n]);		//read AlkalinityUpDose var from json file
				
			}else if(m == 6){
				JsonChlorineDoseValues[n] = readDataFromFile(filename, &JsonChlorineDoseVarNames[n]);		//read Temp var from json file
				
			}		
		}
	}
  PoolVolume = readDataFromFile(filename, &PoolVolumeName[0]);   //read PoolVolume from json file
}

//Write Data to array which will be transfered to json file:
void UpdateArray(float* valueArray, float value){
	for(int k = 0; k <= 19; k++){
	  if(k < 19){																		 
		 valueArray[k] = valueArray[k+1];				//Shift array values one down
	  }else{valueArray[k] = value;}						//add latest reading to end of array
	}
}
//***********************************************************************************************************************************************************************************************************************************
//ThingSpeak Functions
 
 //Write to a single field
 int writeTSData(long TSChannel, int TSField, float data, char* writeAPIKey){
  int  writeSuccess = ThingSpeak.writeField(TSChannel, TSField, data, writeAPIKey); //write the data to the channel
  if (writeSuccess == 200) {
        Serial.println("Channel update successful.");
      }
      else {
        Serial.println("Problem updating channel. HTTP error code " + String(writeSuccess));
      }
  return writeSuccess;
}

// Write multiple fields simultaneously.
/*
int writeTDData(long TSChannel,unsigned int TSField1,float data1,unsigned int TSField2,data2,char* ReadAPIKey){
  ThingSpeak.setField(TSField1,data1);
  ThingSpeak.setField(TSField1,data2);
   
  writeSuccess = ThingSpeak.writeFields(TSChannel, writeAPIKey);
  return writeSuccess;
}
*/
 
 //Read from a single field
float readTSData(long TSChannel,unsigned int TSField,char* ReadAPIKey){
  float data = 0;
  data = ThingSpeak.readFloatField(TSChannel,TSField,ReadAPIKey);
  Serial.println(" Data read from ThingSpeak "+String(data));
  return data;
}

//Assign Dosing Volumes:
void AssignDosingVolumes(){
	AcidAdded = AcidVolumeRatio*PoolVolume;
	AlkalinityUpAdded = AlkalinityUpVolumeRatio*PoolVolume;
	ChlorineAdded = ChlorineVolumeRatio*PoolVolume;
}

/********************************************************************************************************************************************************************************************************
 * Main
 */
void setup() {	
	
	// Init LED and turn off
	pinMode(led_pin, OUTPUT);
	led_state = 0;
	digitalWrite(led_pin, led_state);

	//Turn on RTD IC
	pinMode(RTD_sleep_pin, OUTPUT);
	digitalWrite(RTD_sleep_pin, HIGH);
	//Turn on nRF24L01
	pinMode(nRF24L01_sleep_pin, OUTPUT);
	digitalWrite(nRF24L01_sleep_pin, HIGH);
	
	//Init sleep pin and pull it low, turning the measuring subsystems on 
	pinMode(isolators_sleep_pin, OUTPUT);
	digitalWrite(isolators_sleep_pin, LOW);

	// Start Serial port
	Serial.begin(115200);
	Wire.begin();                 //enable I2C port.

	//nRF24L01
	radio.begin();
	radio.openReadingPipe(1, addresses[0]); // 00001
	radio.openWritingPipe(addresses[1]); // 00002
//	radio.setPALevel(RF24_PA_MIN);
	
	// Make sure can read the file system, SPIFFS
	if( !SPIFFS.begin()){
		Serial.println("Error mounting SPIFFS");
		while(1);
	}

	// Connect to Wi-Fi
	WiFi.begin(ssid, password);
	while (WiFi.status() != WL_CONNECTED) {
		delay(4000);
		Serial.println("Connecting to WiFi..");
		WiFi.begin(ssid, password);
	}
	
	// Print ESP32 Local IP Address
    Serial.print("Connected :)\n My IP address = ");
	Serial.println(WiFi.localIP());
//	Serial.print("Wi-Fi Channel: ");
//	Serial.println(WiFi.channel());
	
	//Initialize ThingSpeak
	ThingSpeak.begin(client);     

	// On HTTP request for root, provide index.html file
	server.on("/", HTTP_GET, onIndexRequest);

	// On HTTP request for style sheet, provide style.css
	server.on("/style.css", HTTP_GET, onCSSRequest);

	// Handle requests for pages that do not exist
	server.onNotFound(onPageNotFound);
	
	// Start ElegantOTA
    AsyncElegantOTA.begin(&server);

	// Start web server
	server.begin();

	// Start WebSocket server and assign callback
	webSocket.begin();
	webSocket.onEvent(onWebSocketEvent);

  //Serial.println("Read entire file in setup");
  //readFile(filename);	//NotUsed
  
  //var_names array for json string
  for(int m = 0; m < 7; m++){   //For loop for going through: Temp, PH & ORP,  AcidDose, AlkalinityUpDose, ChlorineDose
    for(int n = 0; n < 20; n++){  //Each reading name per catagory
      if(m == 0){     //Temperature
        var_names[(m*20) + n] = JsonTempVarNames[n];
        
      }else if(m == 1){ //pH
        var_names[(m*20) + n] = JsonpHVarNames[n];
        
      }else if(m == 2){ //ORP
        var_names[(m*20) + n] = JsonORPVarNames[n];
		
      }else if(m == 3){ //Voltage
        var_names[(m*20) + n] = JsonVoltageVarNames[n];
		
	  }else if(m == 4){ //Voltage
        var_names[(m*20) + n] = JsonAcidDoseVarNames[n];
		
	  }else if(m == 5){ //Voltage
        var_names[(m*20) + n] = JsonAlkalinityUpDoseVarNames[n];
		
	  }else if(m == 6){ //Voltage
        var_names[(m*20) + n] = JsonChlorineDoseVarNames[n];
	  }
    }
  }
  var_names[140] = "PoolVolume";

  //Ensure stored data not lost
  retrieveFFSDataAndStoreInArrays();  
  
  //Current Dosing Volumes:
  AssignDosingVolumes();
  
  // RestartFuelGauge();  // restart fuel-gauge calculations
  // Serial.println("after fuel gauge");
  
  lastTime = millis();
  radio.startListening();
  
  //after awake from deep sleep -> take readings
  DaySleepFlag = 0;
  SystemsActive = 1;
  
  //Battery Check
  takeBatteryReadings();
  float BatteryTest = ((int)(batVoltage * 100))/100.0;
  if(BatteryTest < 2.8){
	  Serial.println("Battery Voltage Low: " + String(BatteryTest));
	  // Go to deep sleep for a day
	esp_sleep_enable_timer_wakeup(SecondsInADay * uS_TO_S_FACTOR);	//timer expects one argument, represents sleep time in micro seconds
	Serial.println("ESP32 enters deep sleep for " + String(SecondsInADay) + " Seconds");
	Serial.flush();
	esp_deep_sleep_start();
  }
  
  //Print the day to serial monitor
  SystemDaysCTR++;	
  Serial.println("\nMonitoring System day: " + String(SystemDaysCTR));
  
  //Delete
  //readFile(filename);
}

/********************************************************************************************************************************************************************************************************
 * loop
 */
void loop() {
	
	//OTA
	AsyncElegantOTA.loop();
	
	// Look for and handle WebSocket data
	webSocket.loop();

	AsyncElegantOTA.loop();
	
	// Connect or reconnect to WiFi
	if (WiFi.status() != WL_CONNECTED) {
	  Serial.println("Attempting to connect :(");
	  while (WiFi.status() != WL_CONNECTED) {
		WiFi.begin(ssid, password);
		delay(5000);
	  }
	  Serial.print("Connected :)");
	}

	AsyncElegantOTA.loop();
	
	//nRF24L01
	//Receive message from Dosing system Read the data if available in buffer
	if (radio.available())
	{
		AsyncElegantOTA.loop();
		radio.read(&msgFromDosingStation_struct, sizeof(msgFromDosingStation_struct));
		ProcessReadingsReceived();
		ReactOnReadingsReceived();
	}

  AsyncElegantOTA.loop();

	if(((millis() - DaySleepStartTime) > (DaySleepDuration*1000)) && (!SystemsActive)){		//Day sleep has ended
		Serial.println("Day Sleep End");
		
		digitalWrite(led_pin, HIGH);								//Turn ON LED
		digitalWrite(RTD_sleep_pin, HIGH);            				//Turn ON RTD IC
		digitalWrite(nRF24L01_sleep_pin, HIGH); 					//Turn ON nRF24L01 module
		digitalWrite(isolators_sleep_pin, LOW);              		//Turn ON Isolators
		
		//Reset the last time for pause between parameter readings
		lastTime = millis();
		
		VoltageTaken = 0;			//Reset timming flags
		TempTaken = 0;
		pHTaken = 0;
		ORPTaken = 0;
		
		DaySleepFlag = 0;
		SystemsActive = 1;
	}
  
  if(DaySleepFlag && SystemsActive && ((millis() - DeepSleepFlagSetTime) > (BufferTimeBeforeDeepSleep*1000))){	//Start day sleep
	Serial.println("ESP32 enters Day Sleep for " + String(DaySleepDuration) + " Seconds");
	digitalWrite(led_pin, LOW);									//Turn off LED
	digitalWrite(RTD_sleep_pin, LOW);            				//Turn off RTD IC
	digitalWrite(nRF24L01_sleep_pin, LOW); 						//Turn off nRF24L01 module
	digitalWrite(isolators_sleep_pin, HIGH);              		//Turn off Isolators
	
	DaySleepStartTime = millis();								//Set the start time for Day Sleep
	SystemsActive = 0;
  }
  
  if(!DaySleepFlag && !DeepSleepFlag){	//Monitoring system Measurement Units Active
  
			if (((millis() - lastTime) > (timerDelay*1000)) && (VoltageTaken == 0)) {
			AsyncElegantOTA.loop();
				//Take battery readings	function
				takeBatteryReadings();
				
				//Send Voltage Data to ThingSpeak
				writeTSData(channelID, 4, batVoltage, writeAPIKey);
				
				//Check if Battery Low
				 if(batVoltage < 2.8){
					Serial.println("Battery Voltage Low: " + String(batVoltage));
					// Go to deep sleep for a day
					esp_sleep_enable_timer_wakeup(SecondsInADay * uS_TO_S_FACTOR);	//timer expects one argument, represents sleep time in micro seconds
					Serial.println("ESP32 enters deep sleep for " + String(SecondsInADay) + " Seconds");
					Serial.flush();
					esp_deep_sleep_start();
				}
				
				//Write Data to array which will be transfered to json file:
				UpdateArray(JsonVoltageValues, batVoltage);
				
				//Send Voltage Data to website
				SendBatteryVoltageToWebsite();
				
				VoltageTaken = 1;	
				
			}

		  AsyncElegantOTA.loop();
			
			if (((millis() - lastTime) > (timerDelay*1000*2)) && (TempTaken == 0)) {
			AsyncElegantOTA.loop();
				//Take Temperature Reading	
				CurrentTemp = takeAverageTempReading(); 
				
				//Write Data to array which will be transfered to json file:
				UpdateArray(JsonTempValues, CurrentTemp);
				//Send to HTML to update website
				SendTemperatureToWebsite(CurrentTemp);
				//Send Temperature Data to ThingSpeak
				writeTSData(channelID, 1, CurrentTemp, writeAPIKey);
				
				TempTaken = 1;
			}
			
		  AsyncElegantOTA.loop();
		  
			if (((millis() - lastTime) > (timerDelay*1000*3)) && (pHTaken == 0)) {
			AsyncElegantOTA.loop();
				//Take pH Reading
				CurrentPH = takeAveragePHReading(CurrentTemp);
				//Send to HTML to update website
				SendPHToWebsite(CurrentPH);
				//Send pH Data to ThingSpeak
				writeTSData(channelID, 2, CurrentPH, writeAPIKey);
				//Write Data to array which will be transfered to json file:
				UpdateArray(JsonpHValues, CurrentPH);
				
				pHTaken = 1;
			}

		  AsyncElegantOTA.loop();

			if (((millis() - lastTime) > (timerDelay*1000*4)) && (ORPTaken == 0)) {
			AsyncElegantOTA.loop();
				//Take ORP Reading	
				CurrentORP = takeAverageORPReading();
				//Send to HTML to update website
				SendORPToWebsite(CurrentORP);
				//Send ORP Data to ThingSpeak
				writeTSData(channelID, 3, CurrentORP, writeAPIKey);
				//Write Data to array which will be transfered to json file:
				UpdateArray(JsonORPValues, CurrentORP);
				
				ORPTaken = 1;
			}

		  AsyncElegantOTA.loop();

			if(VoltageTaken && TempTaken && pHTaken && ORPTaken){
			AsyncElegantOTA.loop();
				
				storeReadingsInFFs();	//Store readings after each read cycle
				
				//Update Dosing System
				radio.stopListening();

				msgTooDosingStation_struct.verification = 1;
				msgTooDosingStation_struct.pH = CurrentPH;         //Set the averaged readings to the corresponding variables in the struct
				msgTooDosingStation_struct.ORP = CurrentORP;       //to be sent to the Dosing System
				msgTooDosingStation_struct.PoolVolume = PoolVolume;       //to be sent to the Dosing System
				
				//Send message to Dosing system
				int writeSuccess = radio.write(&msgTooDosingStation_struct, sizeof(msgTooDosingStation_struct));
					
				if(writeSuccess){
					Serial.println("successfull nRF24L01 write");
					//Display_Readings_sent();
				}else{
					Serial.println("nRF24L01 write failed");
					}
				radio.startListening();
				
				ReadingCycleCTR++;		//Another reading cycle has taken place
				Serial.println("Monitoring System Cycle: " + String(ReadingCycleCTR) + " complete.");
				
				if((ReadingCycleCTR%3) != 0){
					DaySleepFlag = 1;			//Morning or midday
					DeepSleepFlagSetTime = millis();
				}else{
					DeepSleepFlag = 1;						//Deep sleep during night time
					DeepSleepFlagSetTime = millis();		//Set the time Deep Sleep flag set					
				}
				
			}

		  AsyncElegantOTA.loop();
  }
  
	//Sleep	
	if (((millis() - DeepSleepFlagSetTime) > (BufferTimeBeforeDeepSleep*1000)) && (DeepSleepFlag)) 				//Give dosing system time to respond
	{
		if(SystemDaysCTR%3 == 0){
			//Send WhatsApp Notification every 3rd day.
			sprintf(msg_buf, "Hello from your swimming pool Monitor :)\nWater parameters:\n   Battery Voltage = %.2f (V)\n   Temperature = %.2f (C)\n   pH = %.2f\n   ORP = %.2f (mV)", batVoltage, JsonTempValues[19], JsonpHValues[19], JsonORPValues[19]);
			delay(5000);					//The delays are necessary otherwise program crashes
			message_to_whatsapp(msg_buf);  // message
			delay(5000);
		}
		if(ChlorineLockFlag2 == 1){
			//Send WhatsApp Notification
			sprintf(msg_buf, "The swimming pool has entered Chlorine Lock!\n   %.2f Chlorine Doses were made with no effect", ChlorineCtr);
			delay(5000);					//The delays are necessary otherwise program crashes
			message_to_whatsapp(msg_buf);  // message
			delay(5000);
			ChlorineLockFlag2 = 0;
		}
		if(ChlorineLockFlag2 == 2){
			//Send WhatsApp Notification
			sprintf(msg_buf, "The swimming pool has escaped Chlorine Lock:)");
			delay(5000);					//The delays are necessary otherwise program crashes
			message_to_whatsapp(msg_buf);  // message
			delay(5000);
			ChlorineLockFlag2 = 0;
		}
		
		goToDeepSleep();	//enter deep sleep at the end of the day  
	}  
 
}	
