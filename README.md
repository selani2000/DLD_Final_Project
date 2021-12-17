# DLD_Final_Project
`timescale 1ns / 1ps
//////////////////////////////////////////////////////////////////////////////////
// Company: 
// Engineer: 
// 
// Create Date: 17.12.2021 02:07:09
// Design Name: 
// Module Name: toplevelmodule
// Project Name: 
// Target Devices: 
// Tool Versions: 
// Description: 
// 
// Dependencies: 
// 
// Revision:
// Revision 0.01 - File Created
// Additional Comments:
// 
//////////////////////////////////////////////////////////////////////////////////


module clk_div(clk_d, clk);
  parameter div_value = 1;
  output clk_d;
  input clk;

  reg clk_d;
  reg count;

  initial
    begin
      clk_d = 0;
      count = 0;
    end

  always @ (posedge clk)
    begin
      if (count == div_value)
        count <= 0;
      else
        count <= count + 1;
    end

  always @ (posedge clk)
    begin
      if (count == div_value)
        clk_d <= ~clk_d;
    end
endmodule

module h_counter(h_count, trig_v, clk);
  output [9:0] h_count;
  output trig_v;
  input clk;

  reg [9:0] h_count;
  reg trig_v;

  initial h_count = 0;
  initial trig_v = 0;

  always @ (posedge clk)
    begin
      if (h_count < 799)
        begin
          h_count <= h_count + 1;
          trig_v = 0;
        end
      else
        begin
          h_count <= 0;
          trig_v = 1;
        end
    end
endmodule

module v_counter(v_count, enable_v, clk);
  output [9:0] v_count;
  input enable_v, clk;

  reg [9:0] v_count;

  always @ (posedge clk)
    begin
      if (v_count < 524)
        v_count <= v_count + 1;
      else
        v_count <= 0;
    end
endmodule

`timescale 1ns / 1ps
module vga_sync (
  output h_sync,
  output v_sync,
  output video_on,         // active area
  output [9:0] x_loc,      // current pixel x - location
  output [9:0] y_loc,      // current pixel y - location
  input [9:0] h_count,
  input [9:0] v_count
);

  //horizontal
  localparam HD = 640;      // horizontal display area
  localparam HF = 16;       // horizontal (front porch) right border
  localparam HB = 48;       // horizontal (back porch) left border
  localparam HR = 96;       // horizontal retrace

  //vertical
  localparam VD = 480;      // vertical display area
  localparam VF = 10;       // vertical (front porch) bottom border
  localparam VB = 33;       // vertical (back borch) top border
  localparam VR = 2;        // vertical retrace

  assign x_loc = h_count;
  assign y_loc = v_count;

  assign h_sync = (h_count < (HD + HF)) | (h_count >= (HD + HF + HR));
  assign v_sync = (v_count < (VD + VF)) | (v_count >= (VD + VF + VR));
  assign video_on = (h_count < (HD)) && (v_count < VD);
endmodule                  // vga_sync

`timescale 1ns/1ps
module pixel_gen(
  output reg [3:0] red=0,
  output reg [3:0] green=0,
  output reg [3:0] blue=0,
  input U,
  input D,
  input L,
  input R,
  input clk_d,
  input [9:0] pixel_x,
  input [9:0] pixel_y,
  input video_on
);
  
  reg [9:0] x_snake = 100;
  reg [9:0] y_snake = 200;
//  reg [3:0] direction;
  parameter size_snake = 10;
  parameter speed_snake = 1000000;
  reg [31:0] snake_synchronise; //this value used to compare to speed_snake so that the snake doesnt move too quickly
  
//  always @(posedge clk_d) begin
//  //direction used to implement continuous movement when a button is pressed
//    if (U == 1'b1)
//    begin
//        direction <= 3'b000; //upwards direction
//    end
//    else if (D == 1'b1)
//    begin
//        direction <= 3'b001; //downwards
//    end
//    else if (R == 1'b1)
//    begin
//        direction <= 3'b010; //rightwards
//    end
//    else if (L == 1'b1)
//    begin
//        direction <= 3'b100; //leftwards
//    end
//    else 
//    begin
//        direction <= direction; //retain previous direction   
//    end
// end
  
always @(posedge clk_d)begin  
    if (snake_synchronise == speed_snake)
        snake_synchronise <= 0;
    else
        snake_synchronise = snake_synchronise + 1;
    
    if (snake_synchronise == speed_snake)
    begin
        if ((U == 1'b1) & (y_snake != 0))
        begin
            y_snake = y_snake - 5;
        end
        if ((D == 1'b1) & (y_snake != 470))
        begin
            y_snake = y_snake + 5;
        end
        if ((L == 1'b1) & (x_snake != 0))
        begin
            x_snake = x_snake - 5;
        end
        if ((R == 1'b1) & (x_snake != 630))
        begin
            x_snake = x_snake + 5;
        end
     end 
  end
  
  always @(posedge clk_d) begin
    if ((pixel_x < 5)||(pixel_x > 630)||(pixel_y < 10)||(pixel_y > 470)) begin
      red <= 4'hF;
      green <= 4'hF;
      blue <= 4'hF;
    end
    else begin
      red <= video_on?(( (pixel_y > y_snake) & (pixel_y < y_snake + size_snake) & (pixel_x > x_snake) & (pixel_x < x_snake + size_snake) )? 4'hF:4'h0):(4'h0);
      green <= 4'h0;
      blue <= 4'h0;
    end
  end
endmodule

module toplevelmodule(red,green,blue,h_sync,v_sync,clk, U, D, L, R);
  output h_sync,v_sync;
  output [3:0] red,green,blue;
  input clk;
  input U;
  input D;
  input L;
  input R;
  wire clkd,vtrig,videoon;
  wire [9:0] hcount,vcount,xloc,yloc;

  clk_div cd(clkd, clk);
  h_counter hc(hcount, vtrig, clkd);
  v_counter vc(vcount, clkd, vtrig);
  vga_sync vs(h_sync, v_sync, videoon, xloc, yloc, hcount, vcount);
  pixel_gen pg(red, green, blue, U, D, L, R, clkd, xloc, yloc, videoon);
endmodule
