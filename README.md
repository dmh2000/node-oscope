node-oscope
===========
canvas + socket.io demo oscilloscope

![](/oscope.png)

A simple digital oscilloscope that takes data from a remote source and plots it like an oscope.

The project I am working at my day job is an embedded system that interfaces to some analog electronics. One of the
pieces of software we use is a display that looks like an oscilloscope display that show some voltages and frequencies
of the underlying circuits. We have a reasonably elaborate C++ program that runs on windows. It receives data from the embedded
system via UDP datagrams and displays them as waveforms on the screen. One of my coworkers made the idle suggestion that
it would be cool to have the display on a web browser so he wouldn't have to install the program on whatever computer
happened to be available when testing. I told him sure we can do that. I took it as a challenge. 

If you are an electrical engineer you can skip this paragraph. I'm not an EE, but as a software engineer 
I have interfaced to quite a few different embedded system devices. One type of device is the Analog to Digital Converter,
or A/D. An A/D device reads a voltage and converts it to a digital format that a CPU can deal with. Typical A/D's
provide a sample as an 8, 12 or 16 bit value, signed or unsigned, usually 2's complement but other formats could 
be used. Different A/D's can read different voltage ranges. Depending
on the circuitry there is no particular limit on the voltage range. When an A/D reads a voltage, it goes through a 
conversion process that takes some amount of time. The time required will determine how fast the CPU can read the digital value. 
This is the sample rate. Some exotic A/D's are very fast, and others on low-end devices run into the many microseconds.
At the low level the embedded software will read the output of the A/D by reading it from a dedicated address in the
memory map, as if it were a memory location that happens to be volatile. Depending on the CPU and OS, 
the embedded application might access it through a driver API or some other memory mapping setup. 
 
That said, for the purposes of this exercise just assume there is an embedded computer that has one or more A/D's and 
network access but no display. That computer can read the digital values and send them out as UDP datagrams in a binary
format (see the spec below). Then the task is get that data
into a web browser and display it. Since a web browser cannot natively receive UDP datagrams from an arbitrary source, 
there needs to be some infrastructure to take the datagrams and do something with them so they show up on a web page. Here 
are the parts I use to put this together:

* an embedded computer with a custom program to read data and send formatted UDP datagrams to a specified IP address
* a computer running a node.js web server
* a node application that:
    * acts as sort of a reverse proxy between the client browsers and the embedded system
    * generates a real time feed of data to the client browser using websockets (socket.io)
* a web application served by the node server that implements the oscilloscope display

Let's build up this application in steps. 

Step 1 : embedded system simulator
----------------------------------
For now instead of a real embedded computer we will use a simple C or javascript/node program that sends simulated data. This will make
it easier to test. We can make versions of the program with different data formats, rates etc. There is code in the
repository under 'sim' that has a bare bones Visual Studio C version , a Linux compatible C version and a node.js/javascript version. (not much different
between the two C versions other than some data types and initialization). I won't go into the implementation. You can inspect the
code to see what it is doing.

Code is in test/oscope-linux/oscope.c, test/oscope-win32/oscope.c or test/oscope-js/oscope-sim.js

Step 2 : node.js UDP receiver backend
-------------------------------------
A node.js module that receives the formatted UDP datagrams and parses them into something that is friendlier to
the javascript environment.values

* lib/receiver.js : udp datagram receiver 
* test/receiver-test.js : simple Mocha test fixture
    * run the oscope.c simulator 
    * >mocha test/receiver-test.js

simplified code excerpt from lib/receiver.js:

    // parse UDP datagram into a trace object
    var trace = {
      channel         : 0,     // display channel : 1,2
      length          : 0,     // unsigned 16 bit integer, number of samples, maximum 600
      sample          :new Int16Array(MAX_SAMPLES)
    };
  
    function init(port,trace_callback) {
      // create the datagram socket
      sock = dgram.createSocket('udp4');
  
      sock.on("message",function(data,rinfo){
        // 'data' is a mode Buffer. use the node.js API for reading binary data from a Buffer
        trace.channel        = data.readUInt16BE(0);
        trace.length         = data.readUInt16BE(2);
  
        // get samples
        offset = 4;
        for(i=0;i<len;i++) {
          trace.sample[i] = data.readInt16BE(offset);
          offset += 2;
        }
  
        // callback to output handler
        trace_callback(null,trace);
      });
  
      // bind to the expected port
      sock.bind(port,function() {
        console.log('bound to ' + port);
      });
  
    }

Step 3 : A node.js module that can stream data to the web app
-------------------------------------------------------------
* lib/socket.js  : a module that uses socket.io to send the real time data stream to the browser
* localhost:3000/socket-test.html : test web page
    * npm start 
    * open socket-test.html in browser

simplified code excerpt from lib/socket.js:

    function init(sio) {
      io = sio;
  
      // listen for socket.io connections
      io.sockets.on('connection', function (socket) {
        socket.on('disconnect', function () {
          console.log('disconnect');
        });
      });
    }
  
    // this function is passed to the receiver.js module as the trace callback
    // it is called whenever a trace datagram is received and parsed
    function send(err,trace) {
      if (err) // handle the error
      
      // emit an event back to the client
      io.sockets.sockets.forEach(function(v,i,a) {
        v.volatile.emit('trace',JSON.stringify(trace));
      });
    }
    
Step 4 : A web app to actually display the digital data.
--------------------------------------------------------

* node-oscope.js       : server side code, Express.js scaffold to serve the web pages and realtime data.
* public/js/oscope.js  : client side code to render the display
* views/index.html     : web page to display the canvas and controls
 
This web app will display the waveforms on a Canvas elements. It will include Jquery for DOM manipulation and Bootstrap
to format the page and make it responsive.

The Canvas display looks something like an oscilloscope screen arranged in a Bootstrap grid with controls to the right. 
The operator can set parameters for the vertical and horizontal axes.

Terminology:
* counts     : values returned from an A/D are typically in units of 'counts'.
* volts      : A/D's measure volts, although with the proper circuitry other signal types can be measured. but they always
translate to volts on the display.
 
### RUN/STOP
Starts or stops the display of traces. When stopped, it displays the last traces received.

### VERTICAL PARAMETERS

* Bits per Sample    : this determines the range a sample can be in counts 
    * 16 bits =  65536, 12 = 4096 and 8 = 256
* Voltage Range      : sets the range of volts that the A/D sample can represent. in this case the user can enter any range
* Volts Per Division : there are 10 vertical divisions, 5 for + values and 5 for - values. This parameter sets the number 
of volts that can be displayed. for example if volts/division is 1 then the display can show +- 5 volts. if the sampled
voltage is outside that range signals outside it will be off the screen.
* Vertical Position : moves the x axis of each channel up or down to position them as desired

### HORIZONAL PARAMETERS

* Samples Per Second   : the rate at which individual samples are taken by the embedded system. 
* Seconds per Division : there are 10 horizontal divisions. This parameter sets the size in seconds or fractions of
seconds for a single division. 

### CURSORS
Cursors are used to measure in the x or y axis. Select a cursor then position it on the display with the mouse left button.

### SCALING

So now we have the samples coming in, and we have parameters defining the screen layout. We need to determine
how to display the traces. 

For the vertical scale, which shows the amplitude of the signal, we need to translate from 'counts' to pixels. Here
is the formula for that translation:

volts  per yaxis = volts per division * 10 divisions in the yaxis
pixels per yaxis = height of canvas
 
**(voltage range/counts per sample) *  (pixels per yaxis / volts per yaxis) * counts in a sample = height in pixels** 

and since we are displaying a signed value, divide height in pixels by 2 to get the number of pixels offset from the 
x axis.

For the horizontal scale, which shows the x axis spacing of individual sample points. We use samples per second, 
seconds per division and x axis size. is the formula for that translation:

seconds per xaxis = seconds per division * 10 divisions

pixels  per axis  = width of x axis in pixels

**(seconds per xaxis / samples per second) * pixels per xaxis = pixels/sample**


Both of these scaling parameters can be recomputed when a control changes one of the value. They don't need to 
be recalc'ed every time a trace is displayed. On the other hand, the calculations are short and the code is a lot
simpler if they are just recomputed on every paint.

* * *
 
Data Format
-----------
When interfacing between applications using network messages, it is important to document the exact structure
  and content of the messages. 
  
* Each datagram will have a minimal header and then a variable size array of data elements. 
* The data will be binary in network byte order (bit-endian). 
* Sizes are in bytes

<pre>
    OFFSET   SIZE    TYPE        DESCRIPTION
    0        2       uint16_t    channel number  [1 or 2] (the display will be able to display two 'traces')
    2        2       uint16_t    length N        [0-600] count of samples in the message, maximum 600
    4        2*N     int16_t     samples         array of N signed 16 bit samples with a range of -32768 .. 32767.
</pre>

When designing this format, there are some tradeoffs to make. Should the message format specify more about the data? Or should
the web UI let the user specify what they are looking at? In a real oscilloscope, the data is just voltages at the end
of a probe. The operator
dials in the scaling and such using the oscope UI. For this app we will take that approach. The data will be very barebones
and the operator will need to dial in what it needs to look like. The other alternative would be for the data to be richer, with
specification of various parameters such as voltage range, time scale, name tag, etc. That would be a valid approach too and
would make sense if the embedded system were smarter and the web UI needed to be more automatic. 

One other important point is what format the samples are in. Are they 8, 12 or 16 bit values? Signed or unsigned? To keep it simple
we will let the embedded system send a single format and let the server and UI side specify what it is. Let's assume
the data is always signed 16 bits. If the embedded system is reading 8 or 12 bit data, it will have to sign extend that data to  16 
bits. We could have had multiple message formats and let the server side handle conversion of data. 

Why the restriction to 600 samples? Its an arbitrary application specific decision for a couple of reasons. First, it means that each datagram
will fit in the typical Maximum Transmission Unit (MTU) of an ethernet network, which is usually 1500 octets. That's not required for datagrams but it does mean there 
won't be fragmentation at least on the local network. Second, and probably more important, is that the Canvas element on the web page will be scaled in
multiples/fractions of 600 pixels in the X axis. A display isn't enhanced that much if you have more points than there are pixels in the
physical screen (fancy pixel shading aside). And I picked 600 because it fits a layout on a full size screen and can be divided down by 2's for
responsive display on smaller devices. Again, its kind of arbitrary and depends on the application. On the con side, if you zoom in to a trace
by setting the seconds/division field to smaller values, if you don't have many samples per division you can see the quantization of 
the trace line. 
    
Run It
-----------
* the UI is responsive, so feel free to run it on a tablet or phone
* clone the repository
* npm install
* bower install
* start one of the simulator versions (the node.js/javascript is easiest to start up)
    * cd test/oscope-js
    * node oscope-sim.js
* npm start
* open a browser to to ip-addr:3000
