# Project F Display Controller

The Project F display controller makes it easy to add video output to FPGA projects. It's written in Verilog and supports VGA, DVI, and HDMI displays. It includes full configuration for 640x480, 800x600, 1280x720, and 1920x1080, as well as the ability to define custom resolutions. The design aims to be as generic as possible but does make use of Xilinx Series 7 specific features, such as SerDes. See [porting](#porting) for information on adapting the design to other FPGAs. This design and its documentation are licensed under the MIT License.

For tutorials and further information visit [projectf.io](https://projectf.io).

## Contents
- [Display Interface Support](#display-interface-support)
- [Display Resolution Support](#display-resolution-support)
- [Demos](#demos)
- [Architecture](#architecture)
- [Modules](#modules)
- [TMDS Encoder Model](#tmds-encoder-model)
- [Resource Utilization](#resource-utilization)
- [Porting](#porting)

## Display Interface Support
This design supports displays using VGA, DVI, and HDMI.

**VGA** support is straightforward; you can see an example in [`display_demo_vga.v`](hdl/demo/display_demo_vga.v). If you're building your own hardware, then [Retro Ramblings](http://retroramblings.net/?p=190) has a good example of creating a register ladder DAC.

**DVI** & **HDMI** use [transition-minimized differential signalling](https://en.wikipedia.org/wiki/Transition-minimized_differential_signaling) (TMDS) to transmit video over high-speed serial links. HDMI provides extra functionality over DVI, including audio support, but all HDMI displays should accept a standard DVI signal without issue. 

The display controller offers two types of TMDS generation: 

* Direct generation on FPGA
* [Black Mesa Labs DVI Pmod](https://blackmesalabs.wordpress.com/2017/12/15/bml-hdmi-video-for-fpgas-over-pmod/)  

Direct TMDS generation on FPGA requires high-frequency clocks (742.5 MHz for 1280x720) and SerDes but allows full control of the signal including HDMI features. HDMI support is currently via backwards compatibility with DVI: any standard HDMI display should accept DVI signals. However, this display controller lacks support for audio or advanced HDMI features at present.

The Black Mesa Labs (BML) Pmod is based on the Texas Instruments [TFP410](http://www.ti.com/product/TFP410). The Pmod is restricted to standard DVI features but allows even tiny FPGAs to support DVI signalling. As of February 2019, there is an unresolved issue when using a BML DVI Pmod with HDMI displays: some displays report a signal error and show nothing. This issue isn't confined to Project F but has also been reported by Black Mesa Labs themselves.


## Display Resolution Support
The following four display resolutions are tested and included by default (all at 60 Hz refresh rate):
    
     Resolution  Ratio   Clock     
     640 x  480    4:3   25.20 MHz
     800 x  600    4:3   40.00 MHz
    1280 x  720   16:9   74.25 MHz     
    1920 x 1080   16:9  148.50 MHz   

You can easily add timings for other resolutions; see [demos](#demos) for how to do this.

_NB. The canonical clock for 640x480 60Hz is 25.175 MHz, but 25.2 MHz is within VESA spec and easier to generate._


## Demos
The [demo](hdl/demo) directory includes a demo for each supported interface:

* [`display_demo_dvi.v`](hdl/demo/display_demo_dvi.v) - DVI encoded on FPGA
* [`display_demo_dvi_pmod3.v`](hdl/demo/display_demo_dvi_pmod3.v) - DVI generated by BML 3-bit Pmod
* `display_demo_dvi_pmod24.v` - DVI generated by BML 24-bit Pmod (coming soon)
* [`display_demo_vga.v`](hdl/demo/display_demo_vga.v) - analogue VGA with 12-bit output

You can find the list of required modules for each demo in a comment at the top of its file. You'll also need suitable constraints, such as those from the Project F [hardware support](https://github.com/projf/hardware-support) repo.

You can adjust the demo resolution by changing the parameters for `display_clocks`, `display_timings`, and `test_card`. Comments in the demos provide settings for tested [resolutions](#display-resolution-support).


## Architecture
There are two different high-level designs. This section explains the steps used to generate the display signal in both cases.

### Analogue VGA and BML DVI Pmod

1. Display Clocks - synthesizes the pixel clock, for example, 40 MHz for 800x600
2. Display Timings - generates the display sync signals and active pixel position
3. Colour Data - the colour of a pixel, taken from a bitmap, sprite, test card etc.
4. Parallel Colour Output - external hardware converts this to analogue VGA or TMDS DVI as appropriate

### TMDS Encoding on FPGA for DVI or HDMI

1. Display Clocks - synthesizes the pixel and SerDes clocks, for example, 74.25 and 371.25 MHz for 720p
2. Display Timings - generates the display sync signals and active pixel position
3. Colour Data - the colour of a pixel, taken from a bitmap, sprite, test card etc.
4. TMDS Encoder - encodes 8-bit red, green, and blue pixel data into 10-bit TMDS values
5. 10:1 Serializer - converts parallel 10-bit TMDS value into serial form
6. Differential Signal Output - converts the TMDS data into differential form for output via two FPGA pins

## Modules
* **[display_clocks](hdl/display_clocks.v)** - pixel and high-speed clocks for TMDS (includes Xilinx MMCM)
* **[display_timings](hdl/display_timings.v)** - generates display timings, including horizontal and vertical sync
* **[dvi_generator](hdl/dvi_generator.v)** - uses `serializer_10to1` and `tmds_encode_dvi` to generate a DVI signal
* **[serializer_10to1](hdl/serializer_10to1.v)** - serializes the 10-bit TMDS data (includes Xilinx OSERDESE2)
* **[test_card](hdl/test_card.v)** - generates a video test card based on provided resolution
* **[tmds_encoder_dvi](hdl/tmds_encoder_dvi.v)** - encodes 8-bit per colour into 10-bit TMDS values for DVI

You also need a _top_ module to operate the display controller; the project includes [demo](hdl/demo) versions for different display interfaces. When performing TMDS encoding on FPGA, the top module makes use of the Xilinx OBUFDS buffer to generate the differential output.

### Module Parameters

#### Display Clocks
* `MULT_MASTER` - multiplication for both output clocks
* `DIV_MASTER` - division for both output clocks
* `DIV_5X` - division for SerDes clock
* `DIV_1X` - division for pixel clock
* `IN_PERIOD` - period of input clock in nanoseconds (ns)

#### Display Timings
* `H_RES` - active horizontal resolution in pixels 
* `V_RES` - active vertical resolution in lines 
* `H_FP` - horizontal front porch length in pixels
* `H_SYNC` - horizontal sync length in pixels
* `H_BP` - horizontal back porch length in pixels
* `V_FP` - vertical front porch length in lines
* `V_SYNC` - vertical sync length in lines
* `V_BP` - vertical back porch length in lines
* `H_POL` - horizontal sync polarity (0:negative, 1:positive)
* `V_POL` - vertical sync polarity (0:negative, 1:positive)

#### Test Card
* `H_RES` - horizontal test card resolution in pixels 
* `V_RES` - vertical test card resolution in lines 


## TMDS Encoder Model
The display controller includes a simple [Python model](model/tmds.py) to help with TMDS encoder development. 

There are two steps to TMDS encoding: applying XOR or XNOR to the bits to minimise transitions and keeping the overall number of 1s and 0s similar to ensure DC balance. The first step depends only on the current input value, so it is easy to test. However, balancing depends on the previous values, which makes testing harder; this is where the model is particularly useful. 

By default, the Python model encodes all 256 possible 8-bit values in order, but it's easy to change the script to handle other combinations. `A0, A1, B0, or B1` show which of the four balancing options was taken: you can see what they do in the Python or Verilog code.

Sample Python output:

             1s  B   O  76543210    876543210    9876543210
    =======================================================
     30: XNOR(2, 0, A1) 00011110 -> 010100000 -> 1001011111
     31: XNOR(6, 4, B1) 00011111 -> 001011111 -> 1010100000
     32: XOR (3, 0, A0) 00100000 -> 111100000 -> 0111100000

Sample output from Verilog TMDS test bench [tmds_encoder_dvi_tb.v](hdl/test/tmds_encoder_dvi_tb.v). It should match the middle column of the Python output:

    30 010100000   2,   0, A1
    31 001011111   6,   4, B1
    32 111100000   3,   0, A0


## Resource Utilization
The display controller has not been extensively optimized for area but is still lightweight:

    Display          LUT     FF
    ---------------------------
    DVI on FPGA      227     70
    DVI BML 3-bit    144     26
    DVI BML 24-bit   TBC    TBC
    VGA              134     26

For comparison an Artix A35T has 20,800 LUT6 and 41,600 FF: even the full TMDS implementation is using just 1% of the LUTs.

_Synthesized using Vivado 2018.3 with default options. Target Xilinx Artix 7._


## Porting
We strive to create generic HDL designs where possible. However, vendor-specific components are critical to certain functionality, such as high-speed clock generation. The display controller uses three Xilinx-specific components: all display options use the `MMCM` for clock generation, while TMDS encoding on the FPGA requires `OSERDESE2` and `OBUFDS`. Expanded hardware support will be available in future, but in the meantime, we offer the following advice:

* **MMCM** - Mixed-Mode Clock Manager: Replace with the clock or clock synthesizer of your choice, such as PLL. The simplest pixel clock to generate is usually the 40.0 MHz required by 800x600 60 Hz. Some of the pixel clocks are quite hard to generate accurately, which is where the MMCM's support for multiplying and dividing by eighths comes in handy, for example: `74.25 MHz = 100 MHz * 37.125 / 50`. Analogue VGA is relatively tolerant of inaccurate clock frequencies; DVI and HDMI much less so.
* **OSERDESE2** - SerDes: TMDS requires 10:1 serialization, which is supported by chaining two OSERDESE2 together. SerDes is generally the hardest part to port as FPGAs vary enormously in their capabilities. If native 10:1 serialization isn't available you can sometimes make use of logic designed for DDR memory or divide serialization into two 5-bit steps. Only needed for on-FPGA TMDS generation.
* **OBUFDS** - Differential Signalling: Provided your FPGA has at least four pairs of DS pins you should easily be able to use this design. Only needed for on-FPGA TMDS generation.
