# bb_spi-tle5010-74hc165
bitband spi for parallel load from 8 tle5010 and 6 shift registers 74hc165

Main idea is to read all connected devices at the same time, so all here we do not use standart SPI peripherals but connect all MOSI lines to uc pins, BB serial clocks and read all data from port at the same time.

Usual SPI read(not IT and DMA) take ~3500cycles, that implementation use 6500-9500cycles to read 9 devices(6 shift registers on one pin and 8 tle5010 each on theyr own pins). So, much faster sensors pooling can be done.



dont forget to properly initialize shift register sample pin as output, and its data pin as input. 


```

void spi_enable(void)
{
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_9, GPIO_PIN_RESET);
}

void spi_disable(void)
{
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_9, GPIO_PIN_SET);
}


void tle_set_input(void) {
    GPIO_InitStruct.Pin = all_tle_data_pins;
    GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
    GPIO_InitStruct.Pull = GPIO_NOPULL;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
    HAL_GPIO_Init(spi_port, &GPIO_InitStruct);
}

void tle_set_output(void) {
    GPIO_InitStruct.Pin = all_tle_data_pins;
    GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
    GPIO_InitStruct.Pull = GPIO_NOPULL;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
    HAL_GPIO_Init(spi_port, &GPIO_InitStruct);
}




while(1)
{

            tle_set_output();

            HAL_GPIO_WritePin(spi_port, all_tle_data_pins, GPIO_PIN_RESET);
            HAL_GPIO_WritePin(spi_port, clk_pin, GPIO_PIN_RESET);

            spi_enable();

            for (uint8_t i = 0; i < 8; i++) {//cmd[0] = 0x00;
                HAL_GPIO_WritePin(spi_port, clk_pin, GPIO_PIN_SET);
                HAL_GPIO_WritePin(spi_port, clk_pin, GPIO_PIN_RESET);
            }



            //sample reg
            HAL_GPIO_WritePin(spi_port, reg_sample_pin, GPIO_PIN_SET);
            HAL_GPIO_WritePin(spi_port, reg_sample_pin, GPIO_PIN_RESET);
            HAL_GPIO_WritePin(spi_port, reg_sample_pin, GPIO_PIN_SET);

            const uint8_t cmd  = 0x8C;
            uint8_t btns0 = 0;
            for (uint8_t i = 0; i < 8; i++) {

                HAL_GPIO_WritePin(spi_port, all_tle_data_pins, ((cmd << i) & 0b10000000) );


                HAL_GPIO_WritePin(spi_port, clk_pin, GPIO_PIN_SET);
                HAL_GPIO_WritePin(spi_port, clk_pin, GPIO_PIN_RESET);

                btns0 |= (HAL_GPIO_ReadPin(spi_port, reg_data_pin) << i);
            }


            tle_set_input();

            uint16_t bits;
            for(uint8_t b = 0; b < 5; b++) {

                axis1[b] = 0 ;
                axis2[b] = 0 ;
                axis3[b] = 0 ;
                axis4[b] = 0 ;
                axis5[b] = 0 ;
                axis6[b] = 0 ;
                axis7[b] = 0 ;
                axis8[b] = 0 ;
                raw_buttons[b] = 0 ;

                for (uint8_t i = 0; i < 8; i++) {

                    HAL_GPIO_WritePin(spi_port, clk_pin, GPIO_PIN_SET);
                    HAL_GPIO_WritePin(spi_port, clk_pin, GPIO_PIN_RESET);

                    bits = spi_port->IDR & (all_tle_data_pins|reg_data_pin);

                    if(bits & tle_data_pins[0]) axis1[b] |= (0b10000000 >> i);
                    if(bits & tle_data_pins[1]) axis2[b] |= (0b10000000 >> i);
                    if(bits & tle_data_pins[2]) axis3[b] |= (0b10000000 >> i);
                    if(bits & tle_data_pins[3]) axis4[b] |= (0b10000000 >> i);
                    if(bits & tle_data_pins[4]) axis5[b] |= (0b10000000 >> i);
                    if(bits & tle_data_pins[5]) axis6[b] |= (0b10000000 >> i);
                    if(bits & tle_data_pins[6]) axis7[b] |= (0b10000000 >> i);
                    if(bits & tle_data_pins[7]) axis8[b] |= (0b10000000 >> i);

                    if(bits & reg_data_pin) raw_buttons[b] |= 1 << i;


                }
            }

            raw_buttons[6] = btns0;


            spi_disable();
}


```



