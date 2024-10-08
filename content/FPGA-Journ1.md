---
Title: FPGAs - Journey to RISCV P1
Date: 2024-10-08
Category: FPGA
---

FPGAs are pretty cool - you can do some amazing stuff with them. Although I call this series journey to risc v, it's mostly just a scrambled blog of me going through the entire journey, trying out interesting stuff along the way. Hope it's enjoyable for you guys to follow along.  

First things first, I am using a CYC1000 for most of these articles. I got it a few years ago for a pretty cheap price. Some of the instructions here are specific to the Quartus suite, but most should be "multi-platform".  
We need some way of communicating the status of our system on the FPGA to our PC. Keeping with tradition, I choose UART as my protocol of choice to send data to/from my PC.
We will implement a very simple UART module, both for TX and RX. We will look into TX
in this post.
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
module uart_tx#(parameter clock=12000000, parameter baudrate=115200)(output tx_pin,
						input clk, input[32:0] baud_ctr_top, input n_reset,
						input start_write, output reg write_avl, input [7:0] write_data);
localparam baudgen_top_val = clock/baudrate;
reg[32:0] baudgen_ctr;
reg[1:0] write_state; // writer state
reg[9:0] write_reg; // write shift register
reg[3:0] shifted_counter; // used to keep track of how many we have shifted out
reg n_out_en = 0; // used to enable the output. set to 0 to enable the output
wire baud_clock = (baudgen_top_val == baudgen_ctr);
localparam idle_state = 2'd0;
localparam write_busy = 2'd1; // writing is in progress

assign tx_pin = write_reg[0] | n_out_en; // if write state is not write_busy, tx_pin should always be high
always @(posedge clk) begin
	if(!n_reset) begin
		baudgen_ctr <= 0;
		shifted_counter <= 0;
		write_reg <= 0;
		write_avl <= 1;
		write_state <= idle_state;
		n_out_en <= 1; // disable the output
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
			// do nothing
			end
		endcase
	end
end					
endmodule
```
That's a lotta code, but it should be simple enough once we understand how HDLs
generally work. Unlike other programming languages, HDLs are used mainly to implement
state machines.
#### Baurate generation
Let's first start with clock generation. We need something that ticks
at our desired baud rates. The baud generator in actual MCUs tend to be complicated.
Now, I am not sure why, but I think it is done to provide a decently accurate
baud generator across multiple frequencies. Of course, since our hardware is 
configurable, we do not have such simple limits. We will make one with a simple
counter. When the counter equals `baudgen_top_val`, we get a tick. The value is simple
integer divison. I use parameters extensively in code. This allows me to repurpose
code more easily. More importantly, it keeps the complexity of HDL code to a minimum.
We don't need unnecessary mechanisms. The flexibility of an FPGA covers those cases.
Simple is quicker, more often than not. Now, the more inquisitve readers will ask why not use this as the clock for the rest of the circuit. It's a great question. We do this because FPGAs have
specific routing networks for clocks, and different ones for data. Our
baudgen will end up being much less performant. Now, for the purposes of low speed
UART this may not cause issues. As we increase the speed though, it will cause issues.
In general, we want to use our actual clock to sample the tick,
and if the tick is high, we perform the action.
##### The code
```
module uart_tx#(parameter CLOCK=12000000, parameter BAUDRATE=115200)(...);
localparam baudgen_top_val = CLOCK/BAUDRATE;
reg[32:0] baudgen_ctr;
...
wire baud_clock = (baudgen_top_val == baudgen_ctr);
...
always @(posedge clk) begin
	if(!n_reset) begin
        // Reset bit, not important
	end else begin
		baudgen_ctr <= baudgen_ctr + 1;
		if(baudgen_ctr == baudgen_top_val)
			baudgen_ctr <= 0;
        ...
	end
end					
endmodule
```
If you are new to verilog, much of this will be difficult to parse. Keep this in mind  

* parameters are constants
* wires are used to connect bits of data together and perform combinational logic
* always blocks are used to perform sequential logic (It is capable of more, but lets keep things simple for now)
* posedge means the rising edge
* `<=` performs non-blocking assignments. All this means is that all the assignments are done in one go towards the end, so the value does not change after the assignment. It can be a bit difficult to wrap your head around this if you are not a hardware person.
The above code is a simple up counter. It counts up every clock, and resets when it
reaches a particular value. The wire `baudgen_clock` is one when the counter equals
the top value. It can be thought of as pulsing at the baudrate frequency.
#### Getting data ready
Now that we have our clock, we need to get our data ready. Once the device is reset,
we are in the idle state. Like the name suggests, we do nothing here. We wait for the
start condition, trigerred by a HIGH on our input `start_write`. Once we detect this
condition, we ready the shift register, reset bookkeeping variables, and switch to the
state `write_busy`. The way we ready the shift register needs some explanation. Since
our output is essentially just the last bit of the register OR'ed with an enable flag,
we can directly inject the start and the stop bit into the shift register. The LSB willbe 0, and the MSB will be 1, signaling a start and a stop condition.
##### The code
```
module uart_tx#(...);
...
reg[1:0] write_state; // writer state
reg[9:0] write_reg; // write shift register
reg[3:0] shifted_counter; // used to keep track of how many we have shifted out
reg n_out_en = 0; // used to enable the output. set to 0 to enable the output
...
localparam idle_state = 2'd0;
localparam write_busy = 2'd1; // writing is in progress

assign tx_pin = write_reg[0] | n_out_en; // if write state is not write_busy, tx_pin should always be high
always @(posedge clk) begin
	if(!n_reset) begin
        ...
        // Reset, ignore
	end else begin
        ...
		
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
                ...
			end
			default: begin
			// do nothing
			end
		endcase
	end
end					
endmodule
```
We need to be careful about bookkeeping with state machines. The nice things about
state machines is that it makes it easier. Unfortunately, it can still be somewhat of
a pain to debug. Here, we give out signals to let the outside world understand
whether we are free to take a transmission request or not. We need to change it from HIGH when free to LOW when not. It's best to handle these in the transitions from one
state to another.
#### Shifting out data
Now, verilog has some very nifty
syntax that allows us to implement a shift register in one line - 
```
write_reg <= {1'b1, write_reg[9:1]};
```
All this does is shift the data to the right, and fill in the MSB with a binary one.
That binary one is used to emit the stop bit. Of course, how do we know when to stop
shifting? We are shifting out the standard 8 bit frame, a start bit and a stop bit.
So 10 bits total. We must shift 10 times to shift the data out entirely. We use a
counter to keep track of it. Once we have shifted the data out, we must disable the
output flag, do some bookkeeping and shift back to the idle state.
##### The code
```
module uart_tx#(...);
...
reg[9:0] write_reg; // write shift register
reg[3:0] shifted_counter; // used to keep track of how many we have shifted out
reg n_out_en = 0; // used to enable the output. set to 0 to enable the output
...
assign tx_pin = write_reg[0] | n_out_en; // if write state is not write_busy, tx_pin should always be high
always @(posedge clk) begin
	if(!n_reset) begin
        ...
	end else begin
        ...
		case(write_state)
			idle_state: begin
                ...
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
			// do nothing
			end
		endcase
	end
end					
endmodule
```
We first enable the output if it is disabled. If it is enabled, we move to shifting.
Everytime we shift, we increment out counter. Keep in mind that we only shift when
baud tick is HIGH. The rest of the book keeping happens at internal clock speeds.
Once we have shifted out 10 bits, we stop, show the world that we can transmit again
and switch back to idle mode.  



That will be it. Next part will be posted soon!
