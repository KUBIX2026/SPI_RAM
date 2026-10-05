# SPIRAM_CTRL.V

This space is set for specifications on assigned SPIRAM control module within console logic functioning. 

![RAM Type Differentiation](../Images/RAM%20type%20description.png)

Based on this image we can figure out what kind of module we are working with, so by committing to this description we can start creating the logic flowchart for this device. 
Please note that this works through **SPI protocol**.

There are 2 types of RAM memories that will be used. One through SPI protocol called SPI_RAM and the other one that works as b_ram which is parallel to the processor, thus, high speed which has no protocol. During the functioning of the system there are 2 types of data that will be requested from these memories, which one will depend on the scenario in question. 
