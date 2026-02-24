# UART Polling Interrupt
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
This is the simplest method to transmit and receive signals; we can utilize the in-built function provided by STM32 *HAL_UART_Transmit()* and *HAL_UART_Receive()*. Polling is basically receiver constantly asking the system, are you ready every cycle. If the user has input a character, the routine is carried out.
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
To use interrupt in UART, we have to enable NVIC global interrupt in cubeIDE->connectivity->USART2. It is given the topmost preempt priority (priority 0). Instead of the earlier HAL functions, we use the interrupt versions: *HAL_UART_Transmit_IT()* and *HAL_UART_Receive_IT()*.

When a character is sent, the NVIC interrupts the current task and execute the Interrupt Service Routine (ISR). The user-defined *HAL_UART_RxCpltCallback()* function checks the USART called and put the input symbol into currentUserInput variable. It also resets HAL_UART_Receive_IT() so we don’t miss any future signals. 

In the main while loop, we force the system to blink the yellow LED using *HAL_Delay()*, which is a blocking function, which means, the processor is forced to wait for the LED to complete a cycle before servicing the ISR. 
## Logic Analyzer
Using a logic analyzer, we can observe the green LED (PA5) and yellow LED (PA6). According to the capture software, it shows that channel 1 (green LED) will only toggle when channel 2 (yellow LED) finishes the cycle. The button 1 is constantly being pressed on the keyboard without delay. However, the green LED remained pulsing in sync with the LED, ignoring the frequency of the keypresses.

<img width="1430" height="748" alt="image4" src="https://github.com/user-attachments/assets/fe79309b-2c9d-4e83-aa2c-babe67044dfe" />

Figure 4 shows the signals of green LED (channel 1) and yellow LED (channel 2). The sample rate is 50kHz.
We have overcome the problems of receiver missing the signal; however, the system is still blocked due to *HAL_Delay()*. To overcome this problem, we should use a non-blocking algorithm to make green and yellow LEDs fully asynchronous.
## Experiment
In the real experiment, the yellow and green LED flashes in sync, the green LED only toggles at when the yellow LED begins to light up. The same is true for the released signal shown on the terminal PuTTY. This is in agreement with what we observe with the logic analyzer.

## Part 3 UART Non-Blocking Interrupt Mode
With minimal code, we can use prevTime and *HAL_GetTick()* to keep track of time and tell the processor when to blink the LED. We can turn this into a state machine. The yellow LED blinks at its frequency and green LED toggles every time number 1 is pressed on the keyboard.
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

<img width="1122" height="140" alt="Screenshot 2026-02-24 at 5 44 28 PM" src="https://github.com/user-attachments/assets/5a4265fe-d0e8-4178-ac52-de8f5b897b3f" />

We first transmit a welcome message through *HAL_UART_Transmit_DMA()*, the user can now initiate the keypresses, when three characters are received, the green LED (PA5) will flash quickly and “Message Received\r\n” will be sent to the terminal (PuTTY). In this experiment, we also configure DMA to be in circular mode instead of normal mode so we don’t have to manually reenable *HAL_UART_Receive_DMA()* when three keypresses are received; we set the system to be in an infinite loop.

<img width="1430" height="767" alt="Screenshot 2026-02-24 at 6 08 56 PM" src="https://github.com/user-attachments/assets/140b7201-f4ac-4248-ba89-af2d0b6a51ea" />


Figure 6 shows 4 signals measured by Logic Analyzer: Ch1- USART_Rx, Ch2-USART_Tx, Ch3-GreenLED (PA5), Ch4-YellowLED (PA6).

According to the logic analyzer, we can see three negative edge signals (in red), these are the signals emitted by USART_Rx, we can see how far apart they are despite being pressed as fast as the author could (they are 1, 2, and 3 on the keyboard). The green signals are decoded into “Message Received\r\n”. We can also see channel 3 (Green LED) have a rising edge at the beginning of the signal transmission (due to *HAL_UART_RxCpltCallback()*) and a falling edge at the end of the signal (due to *HAL_UART_TxCpltCallback()*). 

<img width="1432" height="531" alt="Screenshot 2026-02-24 at 6 09 21 PM" src="https://github.com/user-attachments/assets/706df2fb-d3be-4708-8d67-c76d032268f8" />


Figure 7 shows the UART signals being decoded using PulseView.

Besides that, we can also see that there is a small time delay between the arrival of the signal 3 and the beginning of the signal (shown in yellow arrows). The time delay is about 23μs. Within this time, the DMA had been counting for three letters to arrive, once the third signal has arrived. DMA initiates an interrupt to the processor and the processor runs *HAL_UART_RxCpltCallback()* which involves lighting the green LED and transmitting the signal.
## How the DMA Liberates the CPU from Simple Movement of Data
One of the best ways to observe the limited resources of a microcontroller is to force it to run at high frequency and see how it fumbles and misses a few beats with a logic analyzer.
	
DMA and interrupts both work on similar principles, they interrupt the processor when it is warranted. Certain tasks such as memory-to-memory transfers can be done using code alone and can be performed faster than DMA transfers. Here, the DMA acts as a butler to perform these mundane tasks, thus freeing up precious processors time so that it can perform other more critical tasks.

To achieve this, we configure in the while loop the task of switching the LED on and off at high speeds. Using the BSRR register, instead of the usual ODR, which modifies all the bits and slows down the switching speed. BSRR is a real single bit modification register.
GPIOA->BSRR = (1 << 6); and GPIOA->BSRR = (1 << (6 + 16)) within the while loop.

As shown in Figure 7, the PA6 pin of yellow LED shows the oscillation of the square wave. If there are any gaps in the square wave, there are two possibilities, either the logic analyzer fails to capture the signal due to noise, or the processor could not switch it because it has to do something else more important.

<img width="1250" height="439" alt="Screenshot 2026-02-24 at 6 10 14 PM" src="https://github.com/user-attachments/assets/d653fef3-9039-4a92-9839-c00dea1d583b" />


Figure 8 shows a similar code, but this time, a normal interrupt is used instead of DMA

By inspecting the message sent, we can see that there 17 regularly-spaced interrupts (red arrows) in the yellow LED data and they fit perfectly with the gaps between chars (shown in blue arrow). Counting the number of characters “Message Received\r\n” has, we get 18. It fits almost perfectly with the 17 gaps from yellow LED. 

The 17 gaps correspond to the interrupt made to the processor. When it has to send out a new character, it has to pause momentarily so that the message would be sent out successfully (the order depends on their priorities decided by NVIC). This is the weakness of interrupts; the CPU has to be interrupted constantly just to send the most basic of message. With the DMA, the CPU asks the DMA to listen for 18 chars before taking over. (two interrupts instead of 17). The simple task of moving the data to UART is delegated to DMA.

The final gap (green arrow) is suspected to be due to the double buffer nature of the microprocessor, \r is loaded into the shift register and \n is waiting in the holding register. Since the processor has been woken up, it does not need to been interrupted again.

The final proof that these gaps are actually interrupts to the CPU is the timing between the gaps. As shown by the mouse cursor in Figure 8, the gap between the interrupt is exactly 86μs. What is the significance of 86μs you ask? Well, it is almost 1/115200, which is the baud rate of our UART communication! 

Every time the CPU communicates with UART a single character, it is interrupted. If we were to use DMA, that number reduces to 2 (Figure 7). The gaps at the beginning and the end of the message are due to CPU having to toggle the green LED and tell DMA how long to listen for the message.

We can observe how the CPU is really single-cored, it can only process one task at a time. This leads to a race and a fight of resources to see who gets processed first. The DMA and CPU also share the same bus lane to SRAM, so they are also limited to a single highway channel. Furthermore, the SRAM can only be read by a single entity at a time, so CPU and DMA also have to fight for who gets to read the same data first.

# Conclusion
This concludes our experiments, we have shown that UART communication can be done in various ways: polling, interrupts and DMA. With polling, the receiver has to ask the transmitter constantly for message. With interrupts, the receiver only has to listen when a message arrives, but it suffers from interrupting the CPU for every messages. With the DMA interrupt method, we can delegate the mundane tasks of moving data using DMA instead. With a simple logic analyzer, we can probe the message sent and how the processor was being interrupted during critical moments.


