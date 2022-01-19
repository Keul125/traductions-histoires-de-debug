https://www.gamedeveloper.com/programming/my-hardest-bug-ever

As a programmer, you learn to blame your code first, second, and third... and somewhere around 10,000th you blame the compiler. Well down the list after that, you blame the hardware. 

This is my hardware bug story.

Among other things, I wrote the memory card (load/save) code for Crash Bandicoot. For a swaggering game coder, this is like a walk in the park; I expected it would take a few days. I ended up debugging that code for 6 weeks. I did other stuff during that time, but I kept coming back to this bug -- a few hours every few days. It was agonizing.

The symptom was that you'd go to save your progress and it would access the memory card,  and almost all the time, it worked normally... But every once in a while the write or read would time out... for no obvious reason. A short write would often corrupt the memory card. The player would go to save, and not only would we not save, we'd wipe their memory card. D'Oh.

After a while, our producer at Sony, Connie Booth, began to panic. We obviously couldn't ship the game with that bug, and after six weeks I still had no clue what the problem was. Via Connie we put the word out to other PS1 devs -- had anybody seen anything like this? Nope. Absolutely nobody had any problems with the memory card system.

About the only thing you can do when you run out of ideas debugging is divide and conquer: keep removing more and more of the errant program's code until you're left with something relatively small that still exhibits the problem. You keep carving parts away until the only stuff left is where the bug is.

The challenge with this in the context of, say, a video game is that it's very hard to remove pieces. How do you still run the game if you remove the code that simulates gravity in the game? Or renders the characters? 

What you have to do is replace entire modules with stubs that pretend to do the real thing, but actually do something completely trivial that can't be buggy. You have to write new scaffolding code just to keep things working at all. It is a slow, painful process.

Long story short: I did this. I kept removing more and more hunks of code until I ended up, pretty much, with nothing but the startup code -- just the code that set up the system to run the game, initialized the rendering hardware, etc. Of course, I couldn't put up the load/save menu at that point because I'd stubbed out all the graphics code. But I could pretend the user used the (invisible) load/save screen and asked to save, then write to the card.

I ultimately ended up with a pretty small amount of code that exhibited the problem -- but still randomly! Most of the time, it would work, but every once in a while, it would fail. Almost all of the actual Crash code had been removed, but it still happened. This was really baffling: the code that remained wasn't really doinganything.

At some moment -- it was probably 3am -- a thought entered my mind. Reading and writing (I/O) involves precise timing. Whether you're dealing with a hard drive, a compact flash card, a Bluetooth transmitter -- whatever -- the low-level code that reads and writes has to do so according to a clock. 

The clock lets the hardware device -- which isn't directly connected to the CPU -- stay in sync with the code the CPU is running. The clock determines the Baud Rate -- the rate at which data is sent from one side to the other. If the timing gets messed up, the hardware or the software -- or both -- get confused. This is really, really bad, and usually results in data corruption.

What if something in our setup code was messing up the timing somehow? I looked again at the code in the test program for timing-related stuff, and noticed that we set the programmable timer on the PS1 to 1kHz (1000 ticks/second). This is relatively fast; it was running at something like 100Hz in its default state when the PS1 started up. Most games, therefore, would have this timer running at 100Hz.

Andy, the lead (and only other) developer on the game, set the timer to 1kHz so that the motion calculations in Crash would be more accurate. Andy likes overkill, and if we were going to simulate gravity, we ought to do it as high-precision as possible!

But what if increasing this timer somehow interfered with the overall timing of the program, and therefore with the clock used to set the baud rate for the memory card?

I commented the timer code out. I couldn't make the error happen again. But this didn't mean it was fixed; the problem only happened randomly. What if I was just getting lucky?

As more days went on, I kept playing with my test program. The bug never happened again. I went back to the full Crash code base, and modified the load/save code to reset the programmable timer to its default setting (100 Hz) before accessing the memory card, then put it back to 1kHz afterwards. We never saw the read/write problems again.

But why?

I returned repeatedly to the test program, trying to detect some pattern to the errors that occurred when the timer was set to 1kHz. Eventually, I noticed that the errors happened when someone was playing with the PS1 controller. Since I would rarely do this myself -- why would I play with the controller when testing the load/save code? -- I hadn't noticed it. But one day one of the artists was waiting for me to finish testing -- I'm sure I was cursing at the time -- and he was nervously fiddling with the controller. It failed. "Wait, what? Hey, do that again!"

Once I had the insight that the two things were correlated, it was easy to reproduce: start writing to memory card, wiggle controller, corrupt memory card. Sure looked like a hardware bug to me.

I went back to Connie and told her what I'd found. She relayed this to one of the hardware engineers who had designed the PS1. "Impossible," she was told. "This cannot be a hardware problem." I told her to ask if I could speak with him.

He called me and, in his broken English and my (extremely) broken Japanese, we argued. I finally said, "just let me send you a 30-line test program that makes it happen when you wiggle the controller." He relented. This would be a waste of time, he assured me, and he was extremely busy with a new project, but he would oblige because we were a very important developer for Sony. I cleaned up my little test program and sent it over.

The next evening (we were in LA and he was in Tokyo, so it was evening for me when he came in the next day) he called me and sheepishly apologized. It was a hardware problem.

I've never been totally clear on what the exact problem was, but my impression from what I heard back from Sony HQ was that setting the programmable timer to a sufficiently high clock rate would interfere with things on the motherboard near the timer crystal. One of these things was the baud rate controller for the memory card, which also set the baud rate for the controllers. I'm not a hardware guy, so I'm pretty fuzzy on the details.

But the gist of it was that crosstalk between individual parts on the motherboard, and the combination of sending data over both the controller port and the memory card port while running the timer at 1kHz would cause bits to get dropped... and the data lost... and the card corrupted.

This is the only time in my entire programming life that I've debugged a problem caused by quantum mechanics.

 

Dave Baggett was the first employee at Naughty Dog and one of two programmers on Crash Bandicoot. Dave now focuses on curing inbox overload at his new startup, Inky.