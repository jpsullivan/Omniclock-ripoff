In progress.  Source rebased due to hardware malfunction.  Basing this off of the Arduino DCF77 decoder below.  Need to convert the German wording to English...



    /**
     * Where is the DCF receiver connected?
     */
    #define DCF77PIN 2
    /**
     * Turn debugging on or off
     */
    //#define DCF_DEBUG 1
    
    /**
     * Number of milliseconds to elapse before we assume a "1",
     * if we receive a falling flank before - its a 0.
     */
    #define DCF_split_millis 140
    /**
     * There is no signal in second 59 - detect the beginning of 
     * a new minute.
     */
    #define DCF_sync_millis 1200
    /**
     * Definitions for the timer interrupt 2 handler:
     * The Arduino runs at 16 Mhz, we use a prescaler of 64 -> We need to 
     * initialize the counter with 6. This way, we have 1000 interrupts per second.
     * We use tick_counter to count the interrupts.
     */
    #define INIT_TIMER_COUNT 6
    #define RESET_TIMER2 TCNT2 = INIT_TIMER_COUNT
    int tick_counter = 0;
    /**
     * DCF time format struct
     */
    struct DCF77Buffer {
      unsigned long long prefix	:21;
      unsigned long long Min	:7;	// minutes
      unsigned long long P1		:1;	// parity minutes
      unsigned long long Hour	:6;	// hours
      unsigned long long P2		:1;	// parity hours
      unsigned long long Day	:6;	// day
      unsigned long long Weekday	:3;	// day of week
      unsigned long long Month	:5;	// month
      unsigned long long Year	:8;	// year (5 -> 2005)
      unsigned long long P3		:1;	// parity
    };
    struct {
        unsigned char parity_flag	:1;
        unsigned char parity_min	:1;
        unsigned char parity_hour	:1;
        unsigned char parity_date	:1;
    } flags;
    /**
     * Clock variables 
     */
    volatile unsigned char DCFSignalState = 0;  
    unsigned char previousSignalState;
    int previousFlankTime;
    int bufferPosition;
    unsigned long long dcf_rx_buffer;
    /**
     * time vars: the time is stored here!
     */
    volatile unsigned char ss;
    volatile unsigned char mm;
    volatile unsigned char hh;
    volatile unsigned char day;
    volatile unsigned char mon;
    volatile unsigned int year;
    
    
    /**
     * used in main loop: detect a new second...
     */
    unsigned char previousSecond;
    
    /**
     * Initialize the DCF77 routines: initialize the variables,
     * configure the interrupt behaviour.
     */
    void DCF77Init() {
      previousSignalState=0;
      previousFlankTime=0;
      bufferPosition=0;
      dcf_rx_buffer=0;
      ss=mm=hh=day=mon=year=0;
      
    #ifdef DCF_DEBUG 
      Serial.println("Initializing DCF77 routines");
      Serial.print("Using DCF77 pin #");
      Serial.println(DCF77PIN);
      pinMode(DCF77PIN, INPUT);
    #endif
      pinMode(DCF77PIN, INPUT);
    #ifdef DCF_DEBUG
      Serial.println("Initializing timerinterrupt");
    #endif
      //Timer2 Settings: Timer Prescaler /64, 
      TCCR2B |= (1<<CS22);    // turn on CS22 bit
      TCCR2B &= ~((1<<CS21) | (1<<CS20));    // turn off CS21 and CS20 bits   
    
      // Use normal mode
      TCCR2A &= ~((1<<WGM21) | (1<<WGM20));   // turn off WGM21 and WGM20 bits 
      TCCR2B &= ~(1<<WGM22);                  // turn off WGM22
    
      // Use internal clock - external clock not used in Arduino
      ASSR |= (0<<AS2);
      TIMSK2 |= (1<<TOIE2) | (0<<OCIE2A);        //Timer2 Overflow Interrupt Enable  
    
      RESET_TIMER2;
      
    #ifdef DCF_DEBUG
      Serial.println("Initializing DCF77 signal listener interrupt");
    #endif  
    
      attachInterrupt(0, int0handler, CHANGE);
    }
    
    /**
     * Append a signal to the dcf_rx_buffer. Argument can be 1 or 0. An internal
     * counter shifts the writing position within the buffer. If position > 59,
     * a new minute begins -> time to call finalizeBuffer().
     */
    void appendSignal(unsigned char signal) {
    
    #ifdef DCF_DEBUG
      Serial.print(", appending value ");
      Serial.print(signal, DEC);
      Serial.print(" at position ");
      Serial.println(bufferPosition);
    #endif
      dcf_rx_buffer = dcf_rx_buffer | ((unsigned long long) signal << bufferPosition);
      // Update the parity bits. First: Reset when minute, hour or date starts.
      if (bufferPosition ==  21 || bufferPosition ==  29 || bufferPosition ==  36) {
        flags.parity_flag = 0;
      }
      // save the parity when the corresponding segment ends
      if (bufferPosition ==  28) {flags.parity_min = flags.parity_flag;};
      if (bufferPosition ==  35) {flags.parity_hour = flags.parity_flag;};
      if (bufferPosition ==  58) {flags.parity_date = flags.parity_flag;};
      // When we received a 1, toggle the parity flag
      if (signal == 1) {
        flags.parity_flag = flags.parity_flag ^ 1;
      }
      bufferPosition++;
      if (bufferPosition == 59) {
        Serial.print("Nummer is te hooooooog");
        finalizeBuffer();
      }
    }
    
    /**
     * Evaluates the information stored in the buffer. This is where the DCF77
     * signal is decoded and the internal clock is updated.
     */
    void finalizeBuffer(void) {
    
      //if(bufferPosition > 59){ bufferPosition = 59; }
    
      Serial.print("finalizeBuffer: ");
      Serial.println(bufferPosition);
     
      if (bufferPosition == 59) {
    #ifdef DCF_DEBUG
        Serial.println("Finalizing Buffer");
    #endif
        struct DCF77Buffer *rx_buffer;
        rx_buffer = (struct DCF77Buffer *)(unsigned long long)&dcf_rx_buffer;
        if (flags.parity_min == rx_buffer->P1  &&
            flags.parity_hour == rx_buffer->P2  &&
            flags.parity_date == rx_buffer->P3) 
        { 
    #ifdef DCF_DEBUG
          Serial.println("Parity check OK - updating time.");
    #endif
          //convert the received bits from BCD
          mm = rx_buffer->Min-((rx_buffer->Min/16)*6);
          hh = rx_buffer->Hour-((rx_buffer->Hour/16)*6);
          ss = 0;
          day= rx_buffer->Day-((rx_buffer->Day/16)*6); 
          mon= rx_buffer->Month-((rx_buffer->Month/16)*6);
          year= 2000 + rx_buffer->Year-((rx_buffer->Year/16)*6);
          
          displaytime();
        }
    #ifdef DCF_DEBUG
          else {
            Serial.println("Parity check NOK - running on internal clock.");
        }
    #endif
      } 
      // reset stuff
      bufferPosition = 0;
      dcf_rx_buffer=0;
    }
    
    /**
     * Dump the time to the serial line.
     */
    void serialDumpTime(void){
      Serial.print("Time: ");
      Serial.print(hh, DEC);
      Serial.print(":");
      Serial.print(mm, DEC);
      Serial.print(":");
      Serial.print(ss, DEC);
      Serial.print(" Date: ");
      Serial.print(day, DEC);
      Serial.print(".");
      Serial.print(mon, DEC);
      Serial.print(".");
      Serial.println(year, DEC);
    }
    
    
    /**
     * Evaluates the signal as it is received. Decides whether we received
     * a "1" or a "0" based on the 
     */
    void scanSignal(void){
      if (DCFSignalState == 1) {
        /* We detected a raising flank, increase per one second! */
        int thisFlankTime = millis();
    
        if (thisFlankTime - previousFlankTime > DCF_sync_millis) {
          #ifdef DCF_DEBUG
            Serial.println("####");
            Serial.println("#### Begin of new Minute!!!");
            Serial.println("####");
          #endif
          finalizeBuffer();
        }
        else if (thisFlankTime - previousFlankTime < 300) {
          #ifdef DCF_DEBUG
            Serial.println(": Double flank detected");
          #endif
          bufferPosition--;
          if (bufferPosition < 0)
            bufferPosition = 0;
        }
        if (thisFlankTime - previousFlankTime > 300)
          previousFlankTime = thisFlankTime;
    
        #ifdef DCF_DEBUG
          Serial.print(previousFlankTime);
          Serial.print(": DCF77 Signal detected, ");
        #endif
    
      }
      else {
        /* or a falling flank */
        int difference = millis() - previousFlankTime;
        #ifdef DCF_DEBUG
          Serial.print("duration: ");
          Serial.print(difference);
        #endif
        if (difference < DCF_split_millis) {
          appendSignal(0);
        }
        else {
          appendSignal(1);
        }
      }
    }
    
    /**
     * The interrupt routine for counting seconds - increment hh:mm:ss.
     */
    ISR(TIMER2_OVF_vect) {
      RESET_TIMER2;
      tick_counter += 1;
      if (tick_counter == 1000) {
        ss++;
        if (ss==60) {
          ss=0;
          mm++;
          if (mm==60) {
            mm=0;
            hh++;
            if (hh==24) 
              hh=0;
          }
        }
        tick_counter = 0;
      }
    };
    
    /**
     * Interrupthandler for INT0 - called when the signal on Pin 2 changes.
     */
    void int0handler() {
      DCFSignalState = digitalRead(DCF77PIN);
    }
    
    
    
    
    /**
     * Time to words
     */
     /**************************************************************************
    
    Display Logic:
    
     "Het is" wordt altijd weergegeven
     
     "Vijf" wordt weergegeven tussen xx:05 en xx:09, xx:35 en xx:39
     "Tien" wordt weergegeven tussen xx:10 en xx:14, xx:40 en xx:44
     "Kwart" wordt weergegeven tussen xx:15 en xx:19, xx:45 en xx:49
    
     'Over' wordt weergegeven tussen xx:05 en xx:19, xx:35 en xx:44
     'Voor' wordt weergegeven tussen xx:20 en xx:59, xx:45 en xx:59
     
     'Half' wordt weergegeven tussen xx:20 en xx:44
    
     'Uur' wordt weergegeven tussen xx:00 en xx:04
    
      Ã‰en wordt weergegeven tussen 12:20 en 13:19, 00:20 en 01:19
      Twee wordt weergegeven tussen 13:20 en 14:19, 01:20 en 02:19
      Drie wordt weergegeven tussen 14:20 en 15:19, 02:20 en 03:19
      Vier wordt weergegeven tussen 15:20 en 16:19, 03:20 en 04:19
      Vijf wordt weergegeven tussen 16:20 en 17:19, 04:20 en 05:19
      Zes wordt weergegeven tussen 17:20 en 18:19, 05:20 en 06:19
      Zeven wordt weergegeven tussen 18:20 en 19:19, 06:20 en 07:19
      Acht wordt weergegeven tussen 19:20 en 20:19, 07:20 en 08:19
      Negen wordt weergegeven tussen 20:20 en 21:19, 08:20 en 09:19
      Tien wordt weergegeven tussen 21:20 en 22:19, 09:20 en 10:19
      Elf wordt weergegeven tussen 22:20 en 23:19, 10:20 en 11:19
      Twaalf wordt weergegeven tussen 23:20 en 00:19, 11:20 en 12:19
    
      Time: 0:0:19 Date: 0.0.0
      aan:1 - aan:2 - aan:24 -  
      
      Time: 1:25:9 Date: 18.10.2009
      HETIS:1 - HALF:3 - PRE_VIJF:4 - VOOR:11 - TWEE:14 -  
      
      Time: 1:26:5 Date: 18.10.2009
      HETIS:1 - HALF:3 - PRE_VIJF:4 - VOOR:11 - MIN1:7 - TWEE:14 -  
      
      Time: 1:27:51 Date: 18.10.2009
      HETIS:1 - HALF:3 - PRE_VIJF:4 - VOOR:11 - MIN1:7 - MIN2:8 - TWEE:14 - 
    
    */
    
    const int HETIS = 52;
    const int UUR = 34; const int HALF = 48;
    
    const int PRE_VIJF = 46;
    const int PRE_TIEN = 50; 
    const int PRE_KWART = 44;
    
    const int MIN1 = 28; const int MIN2 = 33; const int MIN3 = 32; const int MIN4 = 35;
    
    const int VOOR = 51;
    const int OVER = 40;
    
    const int EEN = 53;
    const int TWEE = 47;
    const int DRIE = 41;
    const int VIER = 43;
    const int VIJF = 49;
    const int ZES = 37;
    const int ZEVEN = 31;
    const int ACHT = 42;
    const int NEGEN = 38;
    const int TIEN = 45;
    const int ELF = 36;
    const int TWAALF = 39;
    
    void LEDsInit(){
    
      pinMode(HETIS, OUTPUT);
      
      pinMode(UUR, OUTPUT); pinMode(HALF, OUTPUT);
    
      pinMode(PRE_VIJF, OUTPUT); pinMode(PRE_TIEN, OUTPUT); pinMode(PRE_KWART, OUTPUT);
      pinMode(MIN1, OUTPUT); pinMode(MIN2, OUTPUT); pinMode(MIN3, OUTPUT); pinMode(MIN4, OUTPUT);
      
      pinMode(VOOR, OUTPUT); pinMode(OVER, OUTPUT);
      
      pinMode(EEN, OUTPUT); pinMode(TWEE, OUTPUT); pinMode(DRIE, OUTPUT); pinMode(VIER, OUTPUT); pinMode(VIJF, OUTPUT); pinMode(ZES, OUTPUT);
      pinMode(ZEVEN, OUTPUT); pinMode(ACHT, OUTPUT); pinMode(NEGEN, OUTPUT); pinMode(TIEN, OUTPUT); pinMode(ELF, OUTPUT); pinMode(TWAALF, OUTPUT);
      
      ledsoff();
    
    }
    
    void on(unsigned int pin){
      digitalWrite(pin, HIGH);
    }
    
    void off(unsigned int pin){
      digitalWrite(pin, LOW);
    }
    
    void displaytime(void){
      serialDumpTime();
    
       //alles uitzetten
      ledsoff();
    
      //HET IS altijd aan
      on(HETIS);
    
      //UUR en HALF
      if(mm < 5)
        on(UUR);
      else if(mm >= 20 && mm < 45)
        on(HALF);
      
      //PRE VIJF TIEN en KWART
      if((mm >= 5 && mm < 10) || (mm >= 55) || (mm >= 25 && mm < 30) || (mm >= 35 && mm < 40))
        on(PRE_VIJF);
      else if((mm >= 10 && mm < 15) || (mm >= 50) || (mm >= 20 && mm < 25) || (mm >= 40 && mm < 45))
        on(PRE_TIEN);
      else if((mm >= 15 && mm < 20) || (mm >= 45 && mm < 50))
        on(PRE_KWART);
    
      //VOOR en OVER
      if((mm < 20 && mm >= 5) || (mm >= 35 && mm < 45))
        on(OVER); 
      else if((mm >= 20 && mm < 30) || (mm >= 45))
        on(VOOR);
        
      //extra minuten
      switch (mm) {
        case 1: case 6: case 11: case 16: case 21: case 26: case 31: case 36: case 41: case 46: case 51: case 56: on(MIN1); break;
        case 2: case 7: case 12: case 17: case 22: case 27: case 32: case 37: case 42: case 47: case 52: case 57: on(MIN1); on(MIN2); break;
        case 3: case 8: case 13: case 18: case 23: case 28: case 33: case 38: case 43: case 48: case 53: case 58: on(MIN1); on(MIN2); on(MIN3); break;
        case 4: case 9: case 14: case 19: case 24: case 29: case 34: case 39: case 44: case 49: case 54: case 59: on(MIN1); on(MIN2); on(MIN3); on(MIN4); break;
      }
    
      if ((mm <= 19)){
        switch (hh) {
          case 1: case 13: on(EEN); break;
          case 2: case 14: on(TWEE); break;
          case 3: case 15: on(DRIE); break;
          case 4: case 16: on(VIER); break;
          case 5: case 17: on(VIJF); break;
          case 6: case 18: on(ZES); break;
          case 7: case 19: on(ZEVEN); break;
          case 8: case 20: on(ACHT); break;
          case 9: case 21: on(NEGEN); break;
          case 10: case 22: on(TIEN); break;
          case 11: case 23: on(ELF); break;
          case 12: case 00: on(TWAALF); break;
        }
      }
      else {
        switch (hh) {
          case 1: case 13: on(TWEE); break;
          case 2: case 14: on(DRIE); break;
          case 3: case 15: on(VIER); break;
          case 4: case 16: on(VIJF); break;
          case 5: case 17: on(ZES); break;
          case 6: case 18: on(ZEVEN); break;
          case 7: case 19: on(ACHT); break;
          case 8: case 20: on(NEGEN); break;
          case 9: case 21: on(TIEN); break;
          case 10: case 22: on(ELF); break;
          case 11: case 23: on(TWAALF); break;
          case 12: case 00: on(EEN); break;
        }
      }
    }
    
    void ledsoff(){
      
      off(HETIS);
    
      off(UUR); off(HALF);
      
      off(PRE_VIJF); off(PRE_TIEN); off(PRE_KWART);
      off(MIN1); off(MIN2); off(MIN3); off(MIN4);
      
      off(VOOR); off(OVER);
      
      off(EEN); off(TWEE); off(DRIE); off(VIER); off(VIJF); off(ZES);
      off(ZEVEN); off(ACHT); off(NEGEN); off(TIEN); off(ELF); off(TWAALF);
    
    }
    
    void ledson(){
      
      on(HETIS);
    
      on(UUR); on(HALF);
      
      on(PRE_VIJF); on(PRE_TIEN); on(PRE_KWART);
      on(MIN1); on(MIN2); on(MIN3); on(MIN4);
      
      on(VOOR); on(OVER);
      
      on(EEN); on(TWEE); on(DRIE); on(VIER); on(VIJF); on(ZES);
      on(ZEVEN); on(ACHT); on(NEGEN); on(TIEN); on(ELF); on(TWAALF);
      
    
    }
        
    
    /**
     * Standard Arduino methods below.
     */
    void setup(void) {
      Serial.begin(9600);
      LEDsInit();
      DCF77Init();
    }
    
    char ledsare = 1;  
    void loop(void) {
      if (ss != previousSecond) {
        previousSecond = ss;
      
        if(ss%5 == 0){
           displaytime();
            //ledson();
    /** /
    if(ledsare == 1){
      ledsoff();
      ledsare = 0;
    }
    else {
      ledson();
      ledsare = 1;
    }
    /**/
        }
        //else if(ss%5 == 3){
            //ledsoff();
        //}
      }
      
      if (DCFSignalState != previousSignalState) {
        scanSignal();
        /*
        #ifdef DCF_DEBUG
          if (DCFSignalState) {
          } else {
          }
        #endif
        */
        previousSignalState = DCFSignalState;
      }
    }