# game render loop vr injector
Library that finds the render function in a x64 executable.

The Library is built on the call trace project.
A project or tool or whatever that traces function calls in an executable by using breakpoints.

## The call trace project

For reverse engineering it would be useful to get the exact call trace of a binary
Exactly what jumps where what sections of code calls what addresses and so on.

This information would allow one to find where functions are located in the code. Even if you did not have debug symbols.
Compilation optimisations might make this impossible in some cases. But in some cases it should work.

There are various tools that promise this ability. None of them work except for simple binaries.

### Dynamic instrumentation tools
Due to the emense classic nr of instructions that need to be tracked implementing such feature with a classic debugger is to slow.
The solution to this is a Dynamic instrumentation tool. These exist but are complex beasts.
On linux you have callgrind a part of the valgrind project. Which should work in wine but does not for some reason. On windows you have DynamoRIO.

There is also a tool called frida that allows you to inject javascript in to any x86 binary. But it is slow and crashy.

I have been unable to get callgrind to work partly due to how modern games tend to use launchers that verify the executable. I have not been able to use DynamoRIO mostly cause i struggle to understand where to begin.

I would like to be able to attach after the process has spawned either by replacing a dll or simply by attaching a debugger to the process id.


## Work so far

### Branch tracing

Initially i tried implementing this as a full fledged dynamic instrumentation tool on top of a debugger. This code is located in [track_calls_using_instruction_tracing.py](track_calls_using_instruction_tracing.py) Instead of stepping each instruction in the debugger, decompiles blocks of the executable and adds breakpoints att all branches. Then when it gets to those branches it adds new breakpoints. This continues forever.
This proved to be to slow, especially considering the use of python(which is very slow) as the debugger scripting engine.

### Call tracing
Th file [track_calls_using_debug_events.py](track_calls_using_debug_events.py) also traces using breakpoints but it decompiles the entire executable module by module on startup then adds breakpoints on the main executable calls and returns. This has proved to be an effective learning platform. However it will not work if the executable is using self-modifying code.

Both of these implementations have issues with multithreaded code as the debugger will catch any thread and the code has not been written to take that in to account.


### next step
I have been thinking of new solution that does not capture the call trace as it happens.
It works in reverse based on the assumption that the code you are interested in will run in a loop.

Basically you start by selecting a API call to a dll (direct X) for example.
1. You add a breakpoint in that API call.
2. You look at the return adress then decompile from that adress forward until you hit a ret instruction. (techinically this might fail as the code may jump away so technically you need to trace every jump after this. But assuming a normal boring compiler just decompiling from that point untill there is ret should work)
3. You add a breakpoint at that ret instruction
4. When that breakpoint hits, you obtain the return address from the stack which is from where the current function was called.
5. Then you step one instruction back from that return address and add a breakpoint.
6. Now we hope that the function is run in a loop and that the breakpoint is called again.
7. When these breakpoints hit this is when we now where the calling function starts.
8. Rinse and repeat until you are at the top level.

You may want to capture multiple of each break point as each function may be called from multiple locations.


## Current work

- [x] The symbol loader from winappdbg is extremely slow and very very bad it almost never finds the true function. Implemented a new symbol loader which loads .pdb files.
- [x] Initially we looked at directX 9 due to its simplicity, we need to start looking at the reversing of dx11. Maybe we should build a text dx11 project.
- [ ] We will want to be able to decode the arguments of the API functions we add breakpoints to so that we can differentiate different rendering passes for example. Need to figure out how to do that.
- [ ] Implement the loop based debugger


# Stereo injection
Many stereo injection plugins for games does alternative frame rendering; that is they don't alter the games render code. The mods simply move the camera every other frame and sends every other frame to each eye.
This causes almost instant nausea and is a horrible experience. It is better to simply not have stereoHead tracking at all. Head tracking is 80% of the experience so if you can manage that it is often enough.

The beter way is to actually render the game twice. This can be done in two ways.
* Traditional Rendering:
In a basic setup, you’d indeed update the view/projection matrices for each eye and issue separate draw calls. This means the scene is rendered twice—once for each eye.
* Single-Pass Stereo (Multi-View Rendering):
With advanced techniques, you can leverage multiple viewports along with instancing. In single-pass stereo, you submit your geometry once and use the GPU to transform it twice (or more) for each eye, each using a different viewport and camera matrix. This reduces CPU overhead by avoiding multiple draw calls even though conceptually, two different views are being produced.

Both generate the same image but **Single-Pass Stereo** is more efficient so will in theory allow for higher FPS. But it can also be allot harder to patch in since you need to modify shader code as well as the normal game code. **Traditional Rendering** may be easy to pull off depending argumentson how the games rendering works.
If you are lucky like in the example dx11 file in [example_simple_dx11_render.cpp](example_simple_dx11_render.cpp) you have a clean render function all you need to do is find it, change the camera matrix and run the function an extra time on each loop.
If you are not able to do that you need to capture all DX11 calls.
Then either save their buffers and argumensts then run them again or copy all the arguments and buffers and simultaneously run them in a different dx11 instance.

