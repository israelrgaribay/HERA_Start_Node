/*  Engineer          Date & Time       Tasks Completed
 *============================================================================================
 *  Israel Garibay    4/27/18           Rewriting all code from Scratch      
 *                                      Including More Documentation
 * 
 * 
 * 
 * 
 */
/*===================================================================================================== 
 * The Following Libraries are need for the peripherals that are used for this program
 * SPI.h                -> Needed for communicating with SD card
 * SD.h                 -> SD card library functions for creating, reading, and writing data
 * Wire.h               -> Arduino I2C library for communicating with RTC module and LCD screen
 * LiquidCrystal_I2C.h  -> library for communicating with I2C LCD
=====================================================================================================*/
/*=============================PROGRAM LIBRARIES=====================================================*/

#include <SPI.h>
#include <SD.h>
#include <Wire.h>
//#include <LiquidCrystal_I2C.h>
#include <String.h>

/*=============================PROGRAM CONSTANTS=====================================================*/

#define DS3231_I2C_ADDRESS 0x68
#define motSen 4      // motion sensor is connected to digital pin 4
#define noiSen 5      // noise sensor is connected to digital pin 5
#define dismountSD 0  // button for safely turning off controller
#define SDpin 6       // Chip Select pin for SD card is digital pin 6

/*=============================FUNCTION PROTOTYPES===================================================*/

//void setBeanName(void);
//void setUpLcdCode(void);
void setUpRTC(void);
void setUpSensors(void);
void setUpSDcard(void);
void sampleSensors(byte *, byte *);
void updateLCD(byte *, byte*);
void checkConnectionState(byte);
void checkDismount(void);
void slidingWindow(byte *);
void readDS3231time(byte*, byte*, byte*, byte*, byte*, byte*, byte* );
void upDateFiles(byte*, byte*, byte);
void upDateScratch(byte);
byte decToBcd(byte);
byte bcdToDec(byte);

/*=============================GLOBAL PARAMETERS===================================================*/

File HERAdata;
//LiquidCrystal_I2C lcd(0x27, 16, 2); // set the LCD address to 0x27 for 16 char and 2 line display

/*=============================SETUP CODE===========================================================*/

void setup() {
  //setBeanName();
  //setUpLcdCode();
  setUpRTC();
  setUpSensors();
  setUpSDcard();
  
  // put your setup code here, to run once:

}

/*=============================MAIN() PROGRAM===================================================*/

void loop() {
  ////lcd.clear(); ////lcd.home();                  // reset LCD display
  byte motSampArr[9] = {0,0,0,0,0,0,0,0,0}; // initializde array to hold nothing but zero
  byte noiSampArr[9] = {0,0,0,0,0,0,0,0,0}; // initializde array to hold nothing but zero
  byte activity = 0x11;
  
  while(1){
    sampleSensors(motSampArr, noiSampArr);  // takes noise and motion samples and stores them in their respective array
    updateLCD(motSampArr, noiSampArr);      // update the LCD screen
    checkConnectionState(activity);// checks if the Bean+ controller is connected
    upDateFiles(motSampArr, noiSampArr, activity);  // update SD card data
    slidingWindow(motSampArr);              // updates LCD motion array values
    slidingWindow(noiSampArr);              // updates LCD noise array values
    checkDismount();                        // checks if SD card dismount button has been pressed
    Bean.setLed(0,0,0);
    Bean.sleep(100);
  }
}

////////////////////////////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////////////////////////////

/*=====================================Functions Go Here!!!============================================*/
/*===================================================================================================== 
 * Function Name: setBeanName
 *  Function Description:
 *    Sets the Bean+ device to have the name "HERA Start"
 *    Saves the name to NVM
 *  Input Parameters
 *    None
 *  Output Parameters
 *    None
 *  Return
 *    None
=====================================================================================================*/

//void setBeanName(void){
//  Bean.enableConfigSave(true);
//  String BeanName = "HERA Start";
//  Bean.setBeanName(BeanName);
//}

/*=================================================================== 
 * Function Name: setUpLcdCode
 *  Function Description:
 *    Starts the LCD screen on Sensor Module
 *  Input Parameters
 *    None
 *  Output Parameters
 *    None
 *  Return
 *    None
=====================================================================*/
/*void setUpLcdCode(void){
  ////lcd.begin();                    // start LCD module
  ////lcd.backlight();                // turn on backlight on LCD module
  ////lcd.print("HERA Start Node");   // print "HERA Start Node" on LCD display
  delay(1500);                    // hold screen display for 1.5 seconds
}*/

/*=================================================================== 
 * Function Name: setUpRTC
 *  Function Description:
 *    This function simply sets the Bean+ controller as a master device
 *    on the I2C bus. 
 *    Sets the I2C baud rate to 9600 bits per second
 *  Input Parameters
 *    None
 *  Output Parameters
 *    None
 *  Return
 *    None
=====================================================================*/

void setUpRTC(void){
  Wire.begin();                     // Initiate the Wire library and join the I2C bus as a master
  Serial.begin(9600);               // Set the I2C baud rate as 9600 bits per second
}

/*=================================================================== 
 *  Function Name: setUpSensors
 *  Function Description:
 *    set the sensor pins for the controller
 *  Input Parameters
 *    None
 *  Output Parameters
 *    None
 *  Return
 *    None
=====================================================================*/

void setUpSensors(void){
  pinMode(motSen, INPUT);             // set motion sensor pin to be an input
  pinMode(noiSen, INPUT);             // set noise sensor pin to be an input
  pinMode(dismountSD, INPUT);         // button for dismounting SD card
  pinMode(SDpin, OUTPUT);             // SS (Slave Select) pin for SD card is set as output
}

/*=================================================================== 
 * Function Name: setUpSDcard
 *  Function Description:
 *    Prepares the SD card to be written to. Creates/Open HSTART.txt
 *    file that will hold all log data
 *  Input Parameters
 *    lcd object
 *    HERAdata object
 *  Output Parameters
 *    lcd object
 *    HERAdata starting files
 *  Return
 *    None
=====================================================================*/
void setUpSDcard(void){
  while(!Serial);                       // wait for serial port to connect
  ////lcd.clear(); ////lcd.home();              // clear and reposition cursor on LCD screen
  ////lcd.print("Starting SD Card");        // let user know the program is going to start preparing the SD card
  SD.begin(SDpin);                      // initialize the SD library and the SD card. return 'true' if initialization worked and 'false' if failed
  delay(1000);                          // wait 1 second
  if (!SD.begin(SDpin)){                // try starting SD card again
    ////lcd.clear(); ////lcd.home();            // clear LCD screen
    ////lcd.print("SD Fail to Init");
    ////lcd.setCursor(0,1);                 // set cursor to write on the left most char of the second row
    ////lcd.print("Check SD Card");         // tell the user to check if SD card is there
    delay(10);
    while(1);                           // halt program and have the user restart the microcontroller
  }
  ////lcd.clear(); ////lcd.home();
  ////lcd.print("SD Good 2 Go!");           // let user know the SD card was successfully initialized
  delay(1000);                          // wait 1 second
  HERAdata = SD.open("Hstart.txt", FILE_WRITE);
  if (!HERAdata){
    ////lcd.clear(); ////lcd.home();
    ////lcd.print("Prob with File");
  }
  HERAdata.print("Raw Hera Start Data");
  HERAdata.println();
  HERAdata.print(" M | N | Time     Date    |");
  HERAdata.println();
  HERAdata.close();                       // close file that was written to
}

/*=================================================================== 
 * Function Name: sampleSensors
 *  Function Description:
 *    Samples the motion and noise sensors connected to the bo
 *    module. 
 *  Input Parameters
 *    byte motion[] :: array of bytes that represents motion data
 *    byte noise[]  :: array of bytes that represetn noise data
 *  Output Parameters
 *    both array's are passed by reference, therefore the data that
 *    is written is passed back to amin
 *  Return
 *    None
=====================================================================*/
void sampleSensors(byte motion[], byte noise[]){
    motion[0] = digitalRead(motSen);
    Bean.setScratchNumber(2, motion[0]);
    if (motion[0]){
      Bean.setLed(50,0,0);
    }
    noise[0] = !digitalRead(noiSen);
    Bean.setScratchNumber(3, noise[0]);
    if (noise[0]){
      Bean.setLed(0,50,0);
    }
}

/*=================================================================== 
 * Function Name: updateLCD
 *  Function Description:
 *    Prepares the SD card to be written to. Creates/Open HSTART.txt
 *    file that will hold all log data
 *  Input Parameters
 *    lcd object
 *    HERAdata object
 *  Output Parameters
 *    lcd object
 *    HERAdata starting files
 *  Return
 *    None
=====================================================================*/
void updateLCD(byte motion[], byte noise[]){
  ////lcd.clear(); //lcd.home();              // reset LCD screen
  //lcd.print("Motion:");                 // "Motion:_________"
                                        // "________________"
  for (int index = 0; index <=9; index++){
    //lcd.print(motion[index]);           // "Motion:n________"
                                        // "________________"
  }
  //lcd.setCursor(0,1);                   // set LCD cursor to botrow, left char position
  //lcd.print("Noise :");                 // "Motion:n________"
                                        // "Noise :_________"
  for (int index = 0; index <=9; index++){
    //lcd.print(noise[index]);            // "Motion:n________"
                                        // "Noise :n________"
  }
}
/*=================================================================== 
 * Function Name: checkConnectionState
 *  Function Description:
 *    Prepares the SD card to be written to. Creates/Open HSTART.txt
 *    file that will hold all log data
 *  Input Parameters
 *    lcd object
 *    HERAdata object
 *  Output Parameters
 *    lcd object
 *    HERAdata starting files
 *  Return
 *    None
=====================================================================*/
void checkConnectionState(byte activity){
  if (Bean.getConnectionState() == true) {
    //lcd.clear(); //lcd.setCursor(0,0);       // set cursor to the top row and leftmost point
    //lcd.print("BEAN CONNECTED");
    upDateScratch(activity);
    while (Bean.getConnectionState() == true){
      // hold program here until disconnection
      Bean.setLed(0,0,50);
      };
    if (Bean.getConnectionState() == false){
      //lcd.clear();
      //lcd.setCursor(0,0);
      //lcd.print("BEAN DETACHED");
      Bean.setLed(10,10,50);
      Bean.sleep(100);
      Bean.setLed(0,0,0);
      Bean.sleep(100);
      Bean.setLed(10,10,50);
      Bean.sleep(100);
      Bean.setLed(0,0,0);
    }
    delay(2000);                          // hold "BEAN DETACHED" for 2 seconds
  }
}
/*=================================================================== 
 * Function Name: setUpSDcard
 *  Function Description:
 *    Prepares the SD card to be written to. Creates/Open HSTART.txt
 *    file that will hold all log data
 *  Input Parameters
 *    lcd object
 *    HERAdata object
 *  Output Parameters
 *    lcd object
 *    HERAdata starting files
 *  Return
 *    None
=====================================================================*/
void checkDismount(){
  if (digitalRead(dismountSD) == 1){
    HERAdata.flush();
    HERAdata.close();
    
    //lcd.clear();
    //lcd.home();
    //lcd.print("SD Remove Safe!");
    while(1){
      Bean.setLed(50,50,0);
      Bean.sleep(100);
      Bean.setLed(0,0,0);
    }
  }
}
/*=================================================================== 
 * Function Name: setUpSDcard
 *  Function Description:
 *    Prepares the SD card to be written to. Creates/Open HSTART.txt
 *    file that will hold all log data
 *  Input Parameters
 *    lcd object
 *    HERAdata object
 *  Output Parameters
 *    lcd object
 *    HERAdata starting files
 *  Return
 *    None
=====================================================================*/
void slidingWindow(byte x[]){
  for(int index = 8; index >= 0; index--){
    x[index+1] = x[index];
  }
}
/*=================================================================== 
 * Function Name: setUpSDcard
 *  Function Description:
 *    Prepares the SD card to be written to. Creates/Open HSTART.txt
 *    file that will hold all log data
 *  Input Parameters
 *    lcd object
 *    HERAdata object
 *  Output Parameters
 *    lcd object
 *    HERAdata starting files
 *  Return
 *    None
=====================================================================*/
byte decToBcd(byte val)
{
  return( (val/10*16) + (val%10) );
}
/*=================================================================== 
 * Function Name: setUpSDcard
 *  Function Description:
 *    Prepares the SD card to be written to. Creates/Open HSTART.txt
 *    file that will hold all log data
 *  Input Parameters
 *    lcd object
 *    HERAdata object
 *  Output Parameters
 *    lcd object
 *    HERAdata starting files
 *  Return
 *    None
=====================================================================*/
byte bcdToDec(byte val)
{
  return( (val/16*10) + (val%16) );
}
/*=================================================================== 
 * Function Name: setUpSDcard
 *  Function Description:
 *    Prepares the SD card to be written to. Creates/Open HSTART.txt
 *    file that will hold all log data
 *  Input Parameters
 *    lcd object
 *    HERAdata object
 *  Output Parameters
 *    lcd object
 *    HERAdata starting files
 *  Return
 *    None
=====================================================================*/
void readDS3231time(byte *second, byte *minute, byte *hour, byte *dayOfWeek, byte *dayOfMonth, byte *month, byte *year){
  Wire.beginTransmission(DS3231_I2C_ADDRESS);
  Wire.write(0); // set DS3231 register pointer to 00h
  Wire.endTransmission();
  Wire.requestFrom(DS3231_I2C_ADDRESS, 7);
  // request seven bytes of data from DS3231 starting from register 00h
  *second = bcdToDec(Wire.read() & 0x7f);
  *minute = bcdToDec(Wire.read());
  *hour = bcdToDec(Wire.read() & 0x3f);
  *dayOfWeek = bcdToDec(Wire.read());
  *dayOfMonth = bcdToDec(Wire.read());
  *month = bcdToDec(Wire.read());
  *year = bcdToDec(Wire.read());
}

/*=================================================================== 
 * Function Name: setUpSDcard
 *  Function Description:
 *    Prepares the SD card to be written to. Creates/Open HSTART.txt
 *    file that will hold all log data
 *  Input Parameters
 *    lcd object
 *    HERAdata object
 *  Output Parameters
 *    lcd object
 *    HERAdata starting files
 *  Return
 *    None
=====================================================================*/
void upDateFiles(byte motion[], byte noise[], byte activity){
  byte second,
       minute, 
       hour, 
       dayOfWeek, 
       dayOfMonth, 
       month, 
       year;
  readDS3231time(&second, &minute, &hour, &dayOfWeek, &dayOfMonth, &month, &year);  //update time
  String Sbuffer;
  HERAdata = SD.open("Hstart.txt", FILE_WRITE);        // open data to start storing data
  Sbuffer = " " + String(motion[0]) + " | " + String(noise[0]) + " | "; // " 0 | 0 | "
  HERAdata.print(Sbuffer);
  Sbuffer = "";
  if (hour<10)
  {
    Sbuffer = "0";
  }
  Sbuffer = Sbuffer + String(hour, DEC) + ":";
  HERAdata.print(Sbuffer);            // " x | x | hh:"
  Sbuffer = "";
  if (minute<10)
  {
    Sbuffer = "0";
  }
  Sbuffer = Sbuffer + String(minute, DEC) + ":";
  HERAdata.print(Sbuffer);            // " x | x | hh:mm:"
  Sbuffer = "";
  if (second<10)
  {
    Sbuffer = "0";
  }
  Sbuffer = Sbuffer + String(second, DEC);
  HERAdata.print(Sbuffer);          // " x | x | hh:mm:ss"
  Sbuffer = "";
  delay(100);
  Sbuffer = " " + String(month, DEC) + "/";// " x | x | hh:mm:ss mm/ " 
  HERAdata.print(Sbuffer);     // " x | x | hh:mm:ss mm"
  Sbuffer = "";
  if (dayOfMonth < 10){
    Sbuffer = "0";
  }
  Sbuffer = Sbuffer + String(dayOfMonth, DEC) + "/" + String(year, DEC);// hh:mm:ss mm/dd/yyyy
  HERAdata.print(Sbuffer);
  Sbuffer = " | " + String(activity, DEC) ;
  HERAdata.print(Sbuffer);
  Sbuffer = "";
  HERAdata.println();
  HERAdata.flush();
  HERAdata.close();
  Bean.setLed(0,0,20);
}

/*=================================================================== 
 * Function Name: setUpSDcard
 *  Function Description:
 *    Prepares the SD card to be written to. Creates/Open HSTART.txt
 *    file that will hold all log data
 *  Input Parameters
 *    lcd object
 *    HERAdata object
 *  Output Parameters
 *    lcd object
 *    HERAdata starting files
 *  Return
 *    None
=====================================================================*/
void upDateScratch(byte activity){
  Bean.setLed(20,0,0);
  Bean.setScratchNumber(1, activity);
  //Bean.setLed(0,0,0);
}
