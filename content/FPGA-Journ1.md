---
Title: FPGAs - Journey to RISCV P1
Date: 2024-8-24
Category: FPGA
---

FPGAs are pretty cool - you can do some amazing stuff with them. Although I call this series journey to risc v, it's mostly just a scrambled blog of me fumbling my way through the entire journey, trying out interesting stuff along the way. Hope it's enjoyable for you guys to follow along.  

First things first, I am using a CYC1000 for most of these articles. I got it a few years ago for a pretty cheap price. Some of the instructions here are specific to the Quartus suite, but most should be "multi-platform".  
We need some way of communicating the status of our system on the FPGA to our PC. Keeping with tradition, I choose UART as my protocol of choice to send data to/from my PC.
We will implement a very simple UART module, both for TX and RX.  
### Uart Transmission
We will start with transmission, as that is an easier problem to solve.
![Uart Image](images/uart3.png)
Let us look at the waveform we will have to create. We always begin with a start bit.
We then shift out our data, LSB first, at the baud rate specified. We then signify a
stop condition by staying HIGH. There can be a few variations on this, but we will
go for the most common form. Nice thing about working on an FPGA is that we
can choose to make hardware bits as complicated or as simple as we want. We don't have
to cater for possible use cases, we simply modify the hardware if necessary.
```
module uart_tx#(parameter CLOCK=12000000, parameter BAUDRATE=115200)(output tx_pin,
						input clk, input[32:0] baud_ctr_top, input n_reset,
						input start_write, output reg write_avl, input [7:0] write_data);
localparam baudgen_top_val = CLOCK/BAUDRATE;
reg[32:0] baudgen_ctr;
reg[1:0] write_state; // Writer state
reg[9:0] write_reg; // Write shift register
reg[3:0] shifted_counter; // Used to keep track of how many we have shifted out
reg n_out_en = 0; // Used to enable the output. Set to 0 to enable the output
wire baud_clock = (baudgen_top_val == baudgen_ctr);
localparam idle_state = 2'd0; // Common to both read and write
localparam read_busy = 2'd1; // Command has been given to start reading data
localparam read_fin_state = 2'd2; // Read finished, data available in reg
localparam write_busy = 2'd1; // Writing is in progress

assign tx_pin = write_reg[0] | n_out_en; // If write state is not write_busy, tx_pin should always be high
always @(posedge clk) begin
	if(!n_reset) begin
		baudgen_ctr <= 0;
		shifted_counter <= 0;
		write_reg <= 0;
		write_avl <= 1;
		write_state <= idle_state;
		n_out_en <= 1; // Disable the output
	end else begin
		baudgen_ctr <= baudgen_ctr + 1;
		if(baudgen_ctr == baudgen_top_val)
			baudgen_ctr <= 0;
		
		case(write_state)
			idle_state: begin
			 if(start_write) begin
				write_reg <= {1'b1, write_data, 1'b0};
				write_state <= write_busy;
				write_avl <= 0;
				shifted_counter <= 0;
			 end else begin
				write_avl <= 1;
			end
			end
			write_busy: begin //shift out data
				if(shifted_counter == 4'd10) begin
					write_state <= idle_state;
					write_avl <= 1;
					n_out_en <= 1;
				end
				if(n_out_en == 1'b1) begin
					if(baud_clock) n_out_en <= 0;
				end
				else if(baud_clock) begin
				write_reg <= {1'b1, write_reg[9:1]};
				shifted_counter <= shifted_counter + 1;
				end
			end
			default: begin
			// Do nothing
			end
		endcase
	end
end					
endmodule
```
That's a lotta code, but it should be simple enough once we understand how HDLs
generally work. Unlike other programming languages, HDLs are used mainly to implement
state machines. So we just need to understand the state machine.

