# CRC32

A non-reflected CRC32 implementation in verilog. \
This is a relatively simple project, with only 2 inputs (not including clock and reset) and 2 outputs (one of wich being an echo of one of the inputs) \

## The CRC32 algorithm

### What is CRC32?

CRC stands for cyclic redundancy check, and the 32 means it outputs a 32 bit checksum. \
It is used in things like ZIP, Ethernet and PNG.

### The algorithm

The CRC algorithm divides the input by a binary number derived from a polynomial. \
More specifically the coeficients of the polynomial, so for an example CRC4 algorithm, lets take the arbitrary polynomial `x⁴+x³+x`, \
we have the coeficients `1x⁴+1x³+0x²+1x¹+0x⁰`, giving us the number `11010`. \

Now to calculate our checksum of an arbitrary input, say `1011100`, we:
```
1011100 0000   <-- we pad the input with the polynomial grade number of zeroes
11010          <-- xor by the polynomial number
------------
01101000000
```
next shift over to the next `1` in the result
```
0110100 0000
 11010
-----------   <-- repeat
0000000 0000
```
In this case we got all zeroes, which means the checksum is `0`.

## The very basics of verilog

### Wires

A `wire` is what is called a net, it describes a physical connection. \
It can not store anything, and must be continuosly assigned to something using an `assign` statement.

### Reg

A `reg` is a variable, can hold a value, and is assigned inside an `always` or `initial` block.

## Short overview of source code

```
`default_nettype none // Disables implicit nets
 

module tt_um_crc32 ( // Instanciates the module
	input  wire [7:0] ui_in,    // Dedicated inputs
	output wire [7:0] uo_out,   // Dedicated outputs
	input  wire [7:0] uio_in,   // IOs: Input path
	output wire [7:0] uio_out,  // IOs: Output path
	output wire [7:0] uio_oe,   // IOs: Enable path (active high: 0=input, 1=output)
	input  wire       ena,      // always 1 when the design is powered, so you can ignore it
	input  wire       clk,      // clock
	input  wire       rst_n     // reset_n - low to reset
);

	reg [31:0] crc;
	reg [31:0] next_crc;   // declarations of reg:s and net:s
	wire done;
	assign done = ui_in[1];// continuos assignment of wire, done is always equal to whatever ui_in[1] is at that time
	wire fb;                           // Declaring the feedback bit
	assign fb = crc[31] ^ ui_in[0];    // Continuosly assigning the feedback bit as the first bit in the register xor:ed against the input.
	always @(*) begin
		next_crc = crc;
		if (!done) begin
		next_crc = crc << 1;              // shifting over one bit
		next_crc[26] = fb ^ next_crc[26]; // actually doing the CRC32
		next_crc[23] = fb ^ next_crc[23];
		next_crc[22] = fb ^ next_crc[22];
		next_crc[16] = fb ^ next_crc[16];
		next_crc[12] = fb ^ next_crc[12];
		next_crc[11] = fb ^ next_crc[11];
		next_crc[10] = fb ^ next_crc[10];
		next_crc[8 ] = fb ^ next_crc[8 ];
		next_crc[7 ] = fb ^ next_crc[7 ];
		next_crc[5 ] = fb ^ next_crc[5 ];
		next_crc[4 ] = fb ^ next_crc[4 ];
		next_crc[2 ] = fb ^ next_crc[2 ];
		next_crc[1 ] = fb ^ next_crc[1 ];
		next_crc[0 ] = fb ^ next_crc[0 ];
		end

	end
	always @(posedge clk) begin
		if (!rst_n)  begin 
			crc <= 32'hFFFFFFFF;   // init at reset
		end
		else if (!done) crc <= next_crc;
	end
	wire [31:0] rev;
	genvar i;
	generate 
		for (i = 0; i < 32; i = i + 1) begin     // Reversing the register at output
			assign rev[i] = crc[31-i];
		end
	endgenerate
	wire [31:0] out;
	assign out = ~rev;              // inverting the reversed register at output


	reg sout;
	reg [4:0] count2;
	reg shifting;
	always @(posedge clk) begin
		if (!rst_n) begin
			sout <= 0;
			count2 <= 0;      // initing at reset
			shifting <= 1;
		end
		else if (done && shifting) begin
			sout <= out[count2];        // shifting the register out, one bit per clock on output
		if (count2 == 31) shifting <= 0;
		else count2 <= count2 + 1;      // counting the bits for output
		end
	end

	assign uio_oe = 0;	
	assign uio_out = 0;             // assigning unused IO:s
	assign uo_out[7:2] = 0;
	wire _unused = &{ena, ui_in[7:2], uio_in, 1'b0};
	assign uo_out[0] = sout;
	assign uo_out[1] = done;    // assigning outputs



endmodule
```

I'll now go through the code one `always` block at a time

### Always block one

```
always @(*) begin
    next_crc = crc;
    if (!done) begin
    next_crc = crc << 1;
    next_crc[26] = fb ^ next_crc[26];
    next_crc[23] = fb ^ next_crc[23];
    next_crc[22] = fb ^ next_crc[22];
    next_crc[16] = fb ^ next_crc[16];
    next_crc[12] = fb ^ next_crc[12];
    next_crc[11] = fb ^ next_crc[11];
    next_crc[10] = fb ^ next_crc[10];
    next_crc[8 ] = fb ^ next_crc[8 ];
    next_crc[7 ] = fb ^ next_crc[7 ];
    next_crc[5 ] = fb ^ next_crc[5 ];
    next_crc[4 ] = fb ^ next_crc[4 ];
    next_crc[2 ] = fb ^ next_crc[2 ];
    next_crc[1 ] = fb ^ next_crc[1 ];
    next_crc[0 ] = fb ^ next_crc[0 ];
    end
end
```

The `always @(*)` means that the code inside the block runs anytime anything on the right hand side of an assignment inside the block is updated. \
For example, this is the first line of the block:
```
Left hand side
   ↓
next_crc = crc;
            ↑
     Right hand side
```
So anytime the right hand side (crc) is updated, all of the code in the block is run.

This
```
if (!done) begin
```
means that the code inside the if statement (until the next `end`) will run if `done` is `0`. \
This is because when all the input has been fed, and done goes high, we want to output the register without updating it further.

This line
```
next_crc = crc << 1;
```
shifts the register one bit to the left.

These next couple of lines
```
next_crc[26] = fb ^ next_crc[26];
next_crc[23] = fb ^ next_crc[23];
next_crc[22] = fb ^ next_crc[22];
next_crc[16] = fb ^ next_crc[16];
next_crc[12] = fb ^ next_crc[12];
next_crc[11] = fb ^ next_crc[11];
next_crc[10] = fb ^ next_crc[10];
next_crc[8 ] = fb ^ next_crc[8 ];
next_crc[7 ] = fb ^ next_crc[7 ];
next_crc[5 ] = fb ^ next_crc[5 ];
next_crc[4 ] = fb ^ next_crc[4 ];
next_crc[2 ] = fb ^ next_crc[2 ];
next_crc[1 ] = fb ^ next_crc[1 ];
next_crc[0 ] = fb ^ next_crc[0 ];
```
Implements the xor taps with the feedback bit according to the polynomial `x³²+x²⁶+x²³+x²²+x¹⁶+x¹²+x¹¹+x¹⁰+x⁸+x⁷+x⁵+x⁴+x²+x¹+x⁰`.

### Always block 2

```
always @(posedge clk) begin
	if (!rst_n)  begin 
		crc <= 32'hFFFFFFFF;
	end
	else if (!done) crc <= next_crc;	
end
```

This
```
always @(posedge clk) begin
```
again means that the code inside the block runs anytime `clk` is updated. Specifically on the positive edge, so it updates when `clk` goes from `0` to `1`, but not `1` to `0`.


This,
```
if (!rst_n)  begin 
	crc <= 32'hFFFFFFFF;
end
```
scince reset is active low, means that when reset is active, it initializes crc to the hex value `0xFFFFFFFF`. \
This is so that the register starts as a clean slate whenever a new checksum is to be calculated.

As we saw in the previous `always` block, we set `next_crc = crc;` and update what the *next* clocks register value should be, \
and in this block `crc` is set to the previous clocks `next_crc` value:
```
else if (!done) crc <= next_crc;
```

### Generate block

```
generate
	for (i = 0; i < 32; i = i + 1) begin
		assign rev[i] = crc[31-i];
	end
endgenerate
```

This is a generate for loop. It differs from the always block in the way that, instead of behavioral repetition like in an always block, it "unrolls" the code onto the hardware, meaning it creates 32 copies of the hardware, one for each loop.

All it does is reverse `crc` and put it in `rev`.

### Third always block

```
always @(posedge clk) begin
	if (!rst_n) begin
		sout <= 0;
		count2 <= 0;
		shifting <= 1;
	end
	else if (done && shifting) begin
		sout <= out[count2];
	if (count2 == 31) shifting <= 0;
	else count2 <= count2 + 1;
	end
end
```

The first
```
if (!rst_n)
```
just initializes on reset, nothing too fancy, but the rest is what shifts the register one bit at a time into `sout` for output.

The `shifting` variable tells us if we are still shifting out the reg, or if everything has been shifted already. That's why its counting, to know when its shifted out all 32 bits.
```
else if (done && shifting) begin   <-- checks if we are actively shifting output out
	sout <= out[count2];          <-- shifts the current bit into sout
if (count2 == 31) shifting <= 0;   <-- Sees if all bits are shifted, if so set shifting to 0
else count2 <= count2 + 1;        <-- otherwise count up
end
```

## Waveforms

Now, lets take a look at some of the computed waveforms

![Waveform 1](./images/wav-1.png)

This is an image of the waveforms generated by the program plus a testbench, viewed in GTKwave. I have included some arrows with numbers to make it a bit easier.

Arrow number one is pointing at the `clk` signal. This is what decides the timing of the chip. \
Arrow number two is pointing at a red box from `uo_out[0]`. `uo_out[0]` is the output pin (`sout` in the code), and the red box means it is uninitialized. This is because `sout` isn't initialised until the first clock, and we can se that at the first clock, the red box turns into a `0`.

Arrow number three is pointint at the posedge of `rst_n`. Scince `rst_n` is active low, it going high means it turns inactive. As soon as reset goes inactive the program starts reading input, and as we can see at arrow four, as soon as reset goes inactive, the testbench starts feeding data bits to the input.

Now for the next part of the waveform

![Waveform 2](./images/wav-2.png)

This is about halfway through the simulation.

Arrow one is pointing at the clock. As you can see its still going, and always will be.
Arrow two is pointing at the data pin. It's continuing to feed data for the program to calculate the checksum with.
At arrow number three you can see that both the output pins and the `done` signal are low, this is because the TB dosent drive `done` high until ii's fed all the input bits, and the program dosent output anything until it has gotten all of the input.

![Waveform 3](./images/wav-3.png)

Now this one is a little bit more interesting. This is the end of the simulation.

As you can see at arrow one, the TB feeds it's last input bit, a zero. \
After the last bit has been fed, at arrow two, the TB drives `done` high, indicating that all input bits have been fed. \
At the same time, at arrow three, the programs `uo_out[1]` goes high. This is because that output is simply an echo of the `done` input. \
Then, at arrow four, the program starts driving the output, the first bit being a zero.

