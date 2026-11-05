# CRC32

A non-reflected CRC32 implementation in verilog. \
This is a relatively simple project, with only 2 inputs (not including clock and reset) and 2 outputs (one of wich being an echo of one of the inputs) \

## The CRC32 algorithm

### What is CRC32?

CRC stands for cyclic redundancy check, and the 32 means it outputs a 32 bit checksum. \
It is used in things like ZIP, Ethernet and PNG.

### The algorithm



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

	reg [31:0] crc;        // _
	reg [31:0] next_crc;   // |-- 
	wire done;             // -
	assign done = ui_in[1];
	wire fb;
	assign fb = crc[31] ^ ui_in[0];
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
	always @(posedge clk) begin
		if (!rst_n)  begin 
			crc <= 32'hFFFFFFFF;
		end
		else if (!done) crc <= next_crc;	
	end
	wire [31:0] rev;
	genvar i;
	generate 
		for (i = 0; i < 32; i = i + 1) begin
			assign rev[i] = crc[31-i];
		end
	endgenerate
	wire [31:0] out;
	assign out = ~rev;


	reg sout;
	reg [4:0] count2;
	reg shifting;
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

	assign uio_oe = 0;	
	assign uio_out = 0;
	assign uo_out[7:2] = 0;
	wire _unused = &{ena, ui_in[7:2], uio_in, 1'b0};
	assign uo_out[0] = sout;
	assign uo_out[1] = done;



endmodule
```
