# ui-hpojp-8500
Reverse engineering of HP OfficeJet Pro 8500 UI module. Information saved for any other poor soul who may want to scavenge and reuse one of those.

## Markings on the unit
**PCB:**     
"Western"     
SB0523-00 A   
Energy     

**Sticker:**     
CB022-60081   
DWE02905DZ

## FFC connector pinout
<pre>
1   - Pin 4 of LCD, 5V    
 2  - Reset (UI module input, keep high during use)    
3   - SPI MISO (UI module output)    
 4  - GND    
5   - SPI CLK (UI module input)    
 6  - GND    
7   - SPI MOSI (UI module input)    
 8  - Keypad interrupt output (UI module output, idle high)    
9   - VCC, 3.3V    
 10 - Power button    
11  - GND    
 12 - GND    
13  - GND    
</pre>
Logic levels 3.3V.    
Note that there is no SPI CS present.    
Interrupt appears to go low ca 20 ms after button press/release. Stays low until event is read out. Interrupt signal is first set after start-up.

The MCU used on the UI module is most likely ST72F321J9T6, but not confirmed.

**SPI format:**     
CPOL=1 (idle high)     
CPHA=1 (data valid on second edge)    
Clock ca 300 kHz    
8 bits    
Least significant bit first (!)

## Protocol
UI indicates itself being busy by keeping the MISO line high. Once it goes low, you are free to send commands, but wait 50 us before doing so.


### General format:
Given as "MOSI - MISO"

\<command code\> - 0x00 (no status yet)    
\<data byte count to follow\> - 0x7F ("OK")    
\<data bytes (if previous byte count was "0", this is skipped)\> - 0x7F ("OK")          
...    
\<CRC\> - 0x7F ("OK")  
0x01 ("respond now") - \<command code (UI repeats the original command from the start)\>    
0x7F ("OK") - \<data byte count to follow\>    
0x7F ("OK") - \<data bytes (if previous byte count was "0", this is skipped)\>    
...    
0x7F ("OK") - \<CRC\>    

CRC is simple bitwise XOR of all the bytes sent by respective party. The initial value is 0 and includes command code, data byte count and all data bytes. 

### Commands:
**ID: 0x01**    
Always sent after start-up. Likely intialization or "whois". Regularly sent during normal run, so may also be some sort of status check, but sending once is sufficient.    
Host length: 0    
Host data: none    
UI length: 6    
UI data: 0x75 0x07 0x44 0x06 0x28 0x00 (always same)    

**ID: 0x02**    
Interrupt reason.     
Host length: 0    
Host data: none    
UI length: 1    
UI data: single data byte, bitmask that corresponds to the reason.    
&nbsp;&nbsp; 0x08: start-up (reading this clears the interrupt signal)    
&nbsp;&nbsp; 0x01: key event (does not immediately clear interrupt signal, must read key event via command 0x31)  

**ID: 0x04**    
Unknown. If present, then sent once after first 0x01 and before first 0x30.    
Host length: 3    
Host data: 0x02 0x49 0xf0 (always same)    
UI length: 2    
UI data: 0x3f 0x01 (always same)    

**ID: 0x10**    
Set LEDs.     
Host length: 4    
Host data:     
 &nbsp;&nbsp; byte 0: blinking parameters, where enabled (0x00 may keep all off, anything above 0x81 also, range between ca 0x10...0x80 changes blinking rate/mode. 0x2a is safe starting value)    
&nbsp;&nbsp; byte 1: first set of LED enable/blink bitmasks (see below)    
&nbsp;&nbsp; byte 2: second set of LED enable/blink bitmasks (see below)    
&nbsp;&nbsp; byte 3: unused, can be set to 0x00 or 0xff  
UI length: 0    
UI data: none    

**ID: 0x20**    
Unknown, but essential. Sent once, after first 0x30 (if present), but before first 0x02. Sends UI module to "busy" state for 5 ms.    
Host length: 1    
Host data: 0x09    
UI length: 0    
UI data: none    

**ID: 0x23**    
Write to display first line of text. There is ca 12 ms busy state after sending the characters, before host may request "respond now".  
Host length: 17  
Host data: 0x01, followed by 16 ASCII character values (e.g. 0x65 for "e")  
UI length: 0  
UI data: none  

**ID: 0x24**
Write to display first line of text. There is ca 12 ms busy state after sending the characters, before host may request "respond now".     
Host length: 17    
Host data: 0x01, followed by 16 ASCII character values (e.g. 0x65 for "e")    
UI length: 0    
UI data: none    

**ID: 0x25**    
Set custom characters. If present, then sent after 0x20. The characters are available for printing to screen as 0x90-0x97. Not tested whether these can be changed during runtime.    
Host length: 56    
Host data: Lowest 5 bits used in each byte, 7 bytes per character, 8 characters total.    
UI length: 0    
UI data: none    

**ID: 0x30**    
Unknown. Sent after first 0x04 (if present, otherwise follows 0x01). If sent during runtime, appears to be always preceded by 0x01.    
Host length: 0    
Host data: none    
UI length: 8    
UI data: 8 times 0x7f    

**ID: 0x31**    
Keypad interrupt reason. Always preceded by 0x02. Also clears interrupt signal if command 0x02 indicates reason "0x01".    
Host length: 0    
Host data: none    
UI length: 8 per unread button event (up to 6x8=48)    
UI data: if all buttons released, 8 times 0x7f. Held buttons have respective bit set to 0. Byte/Bitmask below.    


### LED bitmasks:
Each LED has an "enabled" and a "flash" bit. Flashing happens only if both bits are set. With only "enabled" bit set the LED is on solid. Describing second and third byte of the command ("Byte 1" and "Byte 2"). Bitmask given as list (0x80; 0x40; 0x20....)    
Byte 1: Quality2StarFlash; Quality2StarEnable; Quality3StarFlash; Quality3StarEnable; ExclamationFlash; ExclamationEnable;  AutoAnswerFlash; AutoAnswerEnable    
Byte 2: -; -; BacklightFlash; BacklightEnable; PowerFlash; PowerEnable;  Quality1StarFlash; Quality1StarEnable    

### Key bitmasks:
For buttons, idle state is high and held keys have their bit cleared. Bitmask given as list (0x80; 0x40; 0x20....)    
Byte 0: -; addrbook; quickdial4; quickdial3; quickdial2; quickdial1; -; -    
Byte 1: -; num1; left; num#; num7; num4; -; -    
Byte 2: -; num2; ok; num0; num8; num5; -; -    
Byte 3: -; num3; right; num*; num9; num6; -; -    
Byte 4: -; fax; quality; reduce/enl.; copy; faxreso; photo; toMemDevice    
Byte 5: -; redial; copyblack; faxcolor; faxblack; autoanswer; startphoto; startscan    
Byte 6: -; -; -; -; -; -; -; -    
Byte 7: -; -; copycolor; scan; redX; wrench; back; quickdial5    

### Command order
Example of command list that get the unit up and running. Only command IDs shown, not full data.

A "good" startup:   
0x01 (init/whois)   
0x04 (unknown)   
0x30 (unknown)   
0x20 (unknown)   
0x25 (set custom characters)   
0x02 (interrupt reason)   
0x02 (interrupt reason)   
0x10 (set LEDs)   
0x23 (LCD Write Line 1)   
0x24 (LCD Write Line 2)   

Recorded when printer started into some "loading screen":    
0x01 (init/whois)    
0x30 (unknown)   
0x20 (unknown)   
0x10 (set LEDs)   
0x01 (init/whois)   
0x30 (unknown)   
0x10 (set LEDs)   
0x24 (LCD Write Line 2)   

Recorded when printer started into displaying an internal fault:   
0x01 (init/whois)   
0x30 (unknown)   
0x10 (set LEDs)   
0x20 (unknown)   
0x23 (LCD Write Line 1)   
0x24 (LCD Write Line 2)   
0x10 (set LEDs)   
