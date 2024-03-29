https://github.com/Keul125/traductions-histoires-de-debug

Source : [jakepoz.com/debugging-behind-the-iron-curtain/](https://www.jakepoz.com/debugging-behind-the-iron-curtain/)

Debugging Behind the Iron Curtain
=====================

Sergei is a veteran of the early days of the computing industry as it was developing in the Soviet Union. I had the pleasure of working and learning from him over the past year, and in that time I picked up more important lessons about both life and embedded programming than any amount of school could ever teach. The most striking lesson is the story of how and why, in late summer of 1986, Sergei decided to move his family out of the Soviet Union.

In the 1980s, my mentor Sergei was writing software for an SM-1800, a Soviet clone of the PDP-11. The microcomputer was just installed at a railroad station near Sverdlovsk, a major shipping center for the U.S.S.R. at the time. The new system was designed to route train cars and cargo to their intended destinations, but there was a nasty bug that was causing random failures and crashes. The crashes would always occur once everyone had gone home for the night, but despite extensive investigation, the computer always performed flawlessly during manual and automatic testing procedures the next day. Usually this indicates a race condition or some other concurrency bug that only manifests itself under certain circumstances. Tired of late night phone calls from the station, Sergei decided to get to the bottom of it, and his first step was to learn exactly which conditions in the rail yard were causing the computer to crash.

He first compiled a history of all occurrences of the unexplained crashes and plotted their dates and times on a calendar. Sure enough, a pattern was clearly visible. By observing the behavior for several more days, Sergei saw he could easily predict the timing of future system failures.

He soon figured out that the rail yard computer malfunctioned only when the cargo being processed was live cattle coming in from northern Ukraine and western Russia heading to a nearby slaughterhouse. In and of itself this was strange, as the local slaughterhouse had in the past been supplied with livestock from farms located much closer, in Kazakhstan.

As you may know, the Chernobyl Nuclear Power Plant disaster occurred in 1986 and spread deadly levels of radiation which to this day make the nearby area uninhabitable. The radioactivity caused broad contamination in the surrounding areas, including northern Ukraine, Belarus, and western Russia. Suspicious of possibly high levels of radiation in the incoming train cars, Sergei devised a method to test his theory. Possession of personal Geiger counters was restricted by the Soviet government, so he went drinking with a few military personnel stationed at the rail yard. After a few shots of vodka, he was able to convince a soldier to measure one of the suspected rail cars, and they discovered the radiation levels were orders of magnitude above normal.

Not only were the cattle shipments highly contaminated with radiation, the levels were high enough to randomly flip bits in the memory of the SM-1800, which was located in a building close to the railroad tracks.

There were often significant food shortages in the Soviet Union, and the government plan was to mix the meat from Chernobyl-area cattle with the uncontaminated meat from the rest of the country. This would lower the average radiation levels of the meat without wasting valuable resources. Upon discovering this, Sergei immediately filed immigration papers with any country that would listen. The computer crashes resolved themselves as radiation levels dropped over time.