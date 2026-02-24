# UART_Polling_Interrupt
This is a 4-part UART experiment that shows how UART works in STM32
1) 	UART polling mode, green LED misses most of the input
2)	UART interrupt mode with delay, green LED and yellow LED are in sync
3)	UART non-blocking interrupt, green LED and yellow LED are asynchronous.
4)	UART DMA mode, how DMA and CPU fights for resources.
# Setup
This experiment investigates how UART affects the behavior of LED. The green LED comes with the board, (pin PA5). There is a user button we can use to test to transmit signal. A yellow LED is connected to pin PA6 to see how it influences the blinking of green LED. All three LED are wired similarly; the only difference is the code. We use USART2 in this example, PA2 transmits the signals, while PA3 receives the signals.

<img width="640" height="621" alt="image1" src="https://github.com/user-attachments/assets/62d80178-8777-472a-9a25-c8b0069b124b" />

Figure 1 shows the pinout of STM32 F446RE.
The signals are displayed through usb port and shown through PuTTY. The baud rate is set at 115200, 1.15MHz.
# Part 1 UART Polling Mode
This is the simplest method to transmit and receive signals; we can utilize the in-built function provided by STM32 HAL_UART_Transmit() and HAL_UART_Receive(). Polling is basically receiver constantly asking the system, are you ready every cycle. If the user has input a character, the routine is carried out.
There are a few cons with this method:
1)	The processor has to check the buffer every cycle; this is a very CPU-intensive task and consumes a lot of energy.
2)	If the system is doing some other important tasks before the receiving the signals, but the receiving takes up a few milliseconds (to prevent blocking that important tasks), the receiver may miss the signal entirely.
In this experiment, we set up the yellow LED to blink for 1s bright and 1s dark. We only allow the receiver to listen for 10s. So, for 99% of the time, the system is not listening. It will either miss the signal or have a noticeable lag.
## Logic Analyzer
Using a simple logic analyzer, we can observe the signal transmitted, and by using PulseView (provided by Sigrok).

<img width="1427" height="765" alt="image2" src="https://github.com/user-attachments/assets/73633577-804e-43aa-83a6-66805a8964f3" />



Figure 2 shows the Logic Analyzer software and the initial data transmitted. The sampling rate is 10MHz, which is 10x the transmission frequency between USB and STM32.

<img width="1920" height="1030" alt="image3" src="https://github.com/user-attachments/assets/f21e95fd-bd13-4c4a-b5d9-a1e6497acee2" />

Figure 3 shows the signal being decoded with PulseView.
## Experiment
In a real experiment, we can see that the green LED does not follow the click of 1, it only flashes when it is ready. Pressing 2 will check if the button is pressed or released, it is also very laggy because of the polling mode we are using.

# Part 2 UART Blocking Interrupt Mode
To use interrupt in UART, we have to enable NVIC global interrupt in cubeIDE->connectivity->USART2. It is given the topmost preempt priority (priority 0). Instead of the earlier HAL functions, we use the interrupt versions: HAL_UART_Transmit_IT() and HAL_UART_Receive_IT().
 When a character is sent, the NVIC interrupts the current task and execute the Interrupt Service Routine (ISR). The user-defined HAL_UART_RxCpltCallback() function checks the USART called and put the input symbol into currentUserInput variable. It also resets HAL_UART_Receive_IT() so we don’t miss any future signals. 
In the main while loop, we force the system to blink the yellow LED using HAL_Delay(), which is a blocking function, which means, the processor is forced to wait for the LED to complete a cycle before servicing the ISR. 
## Logic Analyzer
Using a logic analyzer, we can observe the green LED (PA5) and yellow LED (PA6). According to the capture software, it shows that channel 1 (green LED) will only toggle when channel 2 (yellow LED) finishes the cycle. The button 1 is constantly being pressed on the keyboard without delay. However, the green LED remained pulsing in sync with the LED, ignoring the frequency of the keypresses.

<img width="1430" height="748" alt="image4" src="https://github.com/user-attachments/assets/fe79309b-2c9d-4e83-aa2c-babe67044dfe" />

Figure 4 shows the signals of green LED (channel 1) and yellow LED (channel 2). The sample rate is 50kHz.
We have overcome the problems of receiver missing the signal; however, the system is still blocked due to HAL_Delay(). To overcome this problem, we should use a non-blocking algorithm to make green and yellow LEDs fully asynchronous.
## Experiment
In the real experiment, the yellow and green LED flashes in sync, the green LED only toggles at when the yellow LED begins to light up. The same is true for the released signal shown on the terminal PuTTY. This is in agreement with what we observe with the logic analyzer.

## Part 3 UART Non-Blocking Interrupt Mode
With minimal code, we can use prevTime and HAL_GetTick() to keep track of time and tell the processor when to blink the LED. We can turn this into a state machine. The yellow LED blinks at its frequency and green LED toggles every time number 1 is pressed on the keyboard.
## Logic Analyzer
The logic analyzer shows that both green LED (PA5) and yellow LED (PA6) oscillate at their own pace and do not influence each other. They are now fully asynchronous.

<img width="1920" height="1030" alt="image5" src="https://github.com/user-attachments/assets/77efd07d-fd74-45f3-833b-636aaf38f818" />


Figure 5 shows that channel 1 (green LED) flashes as quick as the key 1 is pressed while the yellow LED completes one cycle every 2s. They are now completely independent events.
## Experiment
In a real experiment, the green LED reacts immediately after the key 1 was pressed, unaffected by yellow LED. The “released” messages are also shown on the terminal as soon as the key 2 was pressed.
# Part 4 UART with Direct Memory Access (DMA)
	This section is the main dish of the whole project. It encapsulates all the concepts from interrupts, UART and DMA.
	Although the interrupt method is much better than polling, it still involves the processor (ARM-cortex M4) having to handle all the transmission of data (which is nothing but moving data from point A to point B. The DMA is a controller (with the role of a butler) that facilitates the transmission of data so the processor can be freed to do the complicated logics.
	In this section, we configure pin PA3 to receive 3 letters (char) before transmitting the acknowledgement, he is the flowchart:
