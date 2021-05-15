# SPI-master-reusable

// Code your design here
module spi
  #(parameter reg_size = 8)
  
  (rstn,main_clk,spi_start,d_in,t_size,d_out,miso,mosi,sclk,cs);
  input rstn;
  input main_clk;
  input spi_start;
  input [reg_size-1:0] d_in;
  input [count_width:0] t_size;
  output reg [reg_size-1:0] d_out;
  
  input miso;
  output wire mosi;
  output wire sclk;
  output reg cs;
  parameter count_width = $clog2(reg_size);
  parameter reset = 0, idle = 1, load = 2, transact = 3, unload = 4;
  
  reg [reg_size-1:0] mosi_d;
  reg [reg_size-1:0] miso_d;
  reg [count_width:0] count;
  reg [2:0] state;

	// FSM DATA PATH
	always @(state)
	begin
		case (state)
			reset:
			begin
				d_out <= 0;
				miso_d <= 0;
				mosi_d <= 0;
				count <= 0;
				cs <= 1;
			end
			idle:
			begin
				d_out <= d_out;
				miso_d <= 0;
				mosi_d <= 0;
				count <= 0;
				cs <= 1;
			end
			load:
			begin
				d_out <= d_out;
				miso_d <= 0;
				mosi_d <= d_in;
				count <= t_size;
				cs <= 0;
			end
			transact:
			begin
				cs <= 0;
			end
			unload:
			begin
				d_out <= miso_d;
				miso_d <= 0;
				mosi_d <= 0;
				count <= count;
				cs <= 0;
			end

			default:
				state = reset;
		endcase
	end
	//FSM CONTROLLER PATH
  always @(posedge main_clk)
	begin
		if (!rstn)
			state = reset;
		else
			case (state)
				reset:
                  if (spi_start)
						state = load;
					else
						state = idle;
				idle:
                  if (spi_start)
						state = load;
				load:
					if (count != 0)
						state = transact;
					else
						state = reset;
				transact:
					if (count != 0)
						state = transact;
					else
						state = unload;
				unload:
                  if (spi_start)
						state = load;
					else
						state = idle;
			endcase
	end
	

	//Assigning 

  assign mosi = ( ~cs ) ? mosi_d[reg_size-1] : 1'bz;
  assign sclk = ( state == transact ) ? main_clk : 1'b0;
	
	// Shifting
  always @(posedge sclk)
	begin
		if ( state == transact )
          miso_d <= {miso_d[reg_size-2:0], miso};
	end

  always @(negedge sclk)
	begin
		if ( state == transact )
		begin
          mosi_d <= {mosi_d[reg_size-2:0], 1'b0};
			count <= count-1;
		end
	end
	

endmodule








//spi  test bench


// Code your testbench here
// or browse Examples
module spi_test();

	parameter bits = 8;

	reg main_clk;
	reg spi_start;
	reg [bits-1:0] d_in;
	wire [bits-1:0] d_out;
	reg [$clog2(bits):0] t_size;
	wire cs;
	reg rstn;
	wire sclk;
	wire miso;
	wire mosi;

	spi
	#(
		.reg_size(bits)
	) spi
	(
      .main_clk(main_clk),
      .spi_start(spi_start),
		.d_in(d_in),
		.d_out(d_out),
		.t_size(t_size),
		.cs(cs),
		.rstn(rstn),
		.sclk(sclk),
		.miso(miso),
		.mosi(mosi)
	);

	assign miso = mosi;
	always
		#2 main_clk = ~main_clk;

	initial
	begin
		main_clk = 1;
		spi_start = 0;
		d_in = 0;
		rstn = 0;
		t_size = bits;
		#4;
		rstn = 1;
	end

	initial
	begin
      $dumpfile("dump.vcd");
      $dumpvars(0,spi_test);
	end

	initial begin
      #8 spi_start = 1;
      d_in =8'haf;
      
      
      
      #10 spi_start = 0;
      
      #150$finish;
    end

endmodule
