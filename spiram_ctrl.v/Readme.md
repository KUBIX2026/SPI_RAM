# SPIRAM_CTRL.V

Este espacio está reservado para la especificación de la información relevante del funcionamiento de la memoria RAM mediante protocolo `SPI`. 

![RAM Type Differentiation](../Images/RAM%20type%20description.png)

Con base en esta imagen se puede determinar el tipo de módulo con el que se va a trabajar, y en consecuencia, empezar la creación de su diagrama.
En este sistema la memoria `RAM` va a trabajar como almacenador de información referente a variables de ejecución de algún programa. 

# Cosas a considerar. 

1. La información relevante del protocolo se encuentra en [Protocolo SPI](../spiram_ctrl.v/SPI%20Protocol.md)
2. La memoria `RAM` es un dispositivo externo y no es parte de la `FPGA`, sin embargo, el controlador que se desarrolla sí está en la `FPGA`. Se hace necesaria la aclaración para denotar que existen dos dominios; `BUS LOCAL` (lado `CPU`) que trabaja con el reloj del sistema generado por el oscilador de cristal y `BUS EXTERNO` (lado periféricos) que trabaja con un reloj independiente comandado por el protocolo -`SPI` en nuestro caso- con base en la capacidad del periférico en cuestión.
3. El dispositivo que se va a usar consiste en una `RAM` `QUAD-SPI` (para más información respecto a cómo funciona esto remitirse a /SPI Protocol.md). Consiste en un chip `HCS138` (05EG3/M8K3) y 4 memorias APS6404L-3SQ. Toda la información de estos componentes está disponible en los datasheets presentes en [Datasheets](/datasheets).
