# Why ABC9?

ABC9 is a terrible name for a surprisingly useful pass. But what is it, and why do I recommend it so much?

ABC, from the University of California, Berkeley, is a logic toolbox that Yosys uses for a few different things in an FPGA flow: fine-grained optimisation and LUT mapping.
By default, the `abc` pass in Yosys uses some pretty old but robust optimisation commands, followed by an old but robust LUT mapper.
The `abc9` pass uses much newer optimisers and mapping, for a modest improvement in optimisation, and a huge improvement in LUT mapping.

I will talk about LUT mapping, because this is to me the selling point of ABC9 to the average user.

## LUT mapping

When Yosys wants ABC to map logic to LUTs, it hands ABC a network of gates and asks it to find the best mapping it can.
ABC and ABC9 approach this problem in very different ways.

### ABC: the unit delay model, simple and efficient

Let's consider a highly simplified view of an FPGA:
- An FPGA is made up of a network of inputs that connect through LUTs to a network of outputs. These inputs may actually be pins, flops, memory blocks, or whatever, but they're all the same to us.
- Each LUT has a delay between an input and its output of 1 delay unit, and this is constant for all inputs of a LUT, and for all sizes of LUT up to the maximum LUT size allowed; e.g. the delay between the input of a LUT2 and its output is the same as the delay between the input of a LUT6 and its output.
- A LUT may take up a variable number of area units. This is constant for each size of LUT; e.g. a LUT4 may take up 1 unit of area, but a LUT5 may take up 2 units of area, but this applies for all LUT4s and LUT5s.

This is known as the "unit delay model", because each LUT uses one unit of delay.

From this view, the problem ABC has to solve is finding a mapping of the network to LUTs that has the lowest delay, and then optimising the mapping for size while maintaining this delay.

This approach has advantages:
- It is simple and easy to implement.
- Working with unit delays is fast to manipulate.
- It reflects *some* FPGA families, for example, the iCE40HX/LP fits the assumptions of the unit delay model quite well (almost all synchronous blocks, except for adders).

But this approach has drawbacks, too:
- The network of inputs and outputs with only LUTs means that a lot of combinational cells (multipliers and LUTRAM) are invisible to the unit delay model, meaning the critical path it optimises for is not necessarily the actual critical path.
- LUTs are implemented as multiplexer trees, so there is a delay caused by the result propagating through the remaining multiplexers. This means the assumption of delay being equal isn't true in physical hardware, and is proportionally larger for larger LUTs.
- Even synchronous blocks have arrival times (propagation delay between clock edge to output changing) and setup times (requirement for input to be stable before clock edge) which affect the delay of a path.

### ABC9: the generalised delay model, realistic and flexible

The unit delay model doesn't fit FPGA hardware very well. We need to generalise it. Let's try this view instead:
- An FPGA is made up of a network of inputs that connect through LUTs and combinational boxes to a network of outputs. These boxes have specified delays between inputs and outputs, and may have an associated network ("white boxes") or not ("black boxes"), but must be treated as a whole.
- Each LUT has a specified delay between an input and its output in arbitrary delay units, and this varies for all inputs of a LUT and for all sizes of LUT, but each size of LUT has the same associated delay; e.g. the delay between input A and output is different between a LUT2 and a LUT6, but is constant for all LUT6s.
- A LUT may take up a variable number of area units. This is constant for each size of LUT; e.g. a LUT4 may take up 1 unit of area, but a LUT5 may take up 2 units of area, but this applies for all LUT4s and LUT5s.

This is known as the "generalised delay model", because it has been generalised to arbitrary delay units. ABC9 doesn't actually care what units you use here, but the Yosys convention is picoseconds.
Note the introduction of boxes as a concept. While the generalised delay model does not require boxes, they naturally fit into it to represent combinational delays.
Even synchronous delays like arrival and setup can be emulated with combinational boxes that act as a delay.
This is further extended to white boxes, where the mapper is able to see inside a box, and remove orphan boxes with no outputs, such as adders.

Again, ABC9 finds a mapping of the network to LUTs that has the lowest delay, and then minimises it to find the lowest area, but it has a lot more information to work with about the network.

The result here is that ABC9 can remove boxes (like adders) to reduce area, optimise better around those boxes, and also permute inputs to give the critical path the fastest inputs.