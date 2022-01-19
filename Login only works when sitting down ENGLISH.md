https://www.reddit.com/r/talesfromtechsupport/comments/3v52pw/i_cant_log_in_when_i_stand_up/

I can't log in when I stand up.

# Medium

This is a second hand story told to me 20 years ago by someone who was already a veteran sysadmin back then, so it could have happened in the 80s or early 90s.

The scene is a factory making heavy machinery. They are modern and the factory floor had terminals connected to a mainframe for tracking parts and whatever else they needed it for.

One day a sysadmin gets a call from the factory floor and after the usual pleasantries the user says

    I can't log in when I stand up.

The sysadmin thinks that it's one of those calls again and goes through the usual

    Is the power on What do you see on the terminal Have you forgotten your password

The user interrupts

    I know what I'm doing, when I sit down I can log in and everything works, but I can't log in when I stand up.

The sysadmin tries to explain that there can be no possible connection between the chair and the terminal and sitting or standing should in no way affect the ability to log in. After a long back and forth on the phone, he finally gives up and walks to the factory floor to show the user that standing can't affect logging in.

The sysadmin sits down at the terminal, gets the password from the user, logs in and everything is fine. Turns to the user and says

    See It works, your password is fine.

The user answers

    Yeah, told you, now log out, stand up and try again.

The sysadmin obliges, logs out, stands up, types the password and invalid password. Ok, that's just bad luck. He tries again invalid password. And again invalid password. Baffled by this, the sysadmin tries his own mainframe account standing invalid password. He sits down and manages to log in just fine. This has now turned from crazy user to a really fascinating debugging problem.

The word spreads about the terminal with the chair as an input device and other people start flocking around it. Those are technical people in a relatively high tech factory, they are all interested in fun debugging. Production grinds to a halt. Everyone wants to try if they are affected, it turns out that most people can log in just fine, but there are certain people who can't log in standing and there are quite a few who can't log in regardless of standing or sitting.

After a long debugging session they find it. Turns out that some joker pulled out two keys from the keyboard and switched their places. Both the user and the sysadmin had one of those letters in the password. They were both relatively good at typing and didn't look down at the keyboard when typing when sitting. But typing when standing is something they weren't used to and had to look down at the keyboard which made them press the wrong keys. Some users couldn't type properly and never managed to log in. While others didn't have those letters in their passwords and the switched keys didn't bother them at all.