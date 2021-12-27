> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [halove23.blogspot.com](https://halove23.blogspot.com/2021/09/zdi-21-1053-bypassing-windows-lock.html?m=1)

> In April 2021, I discovered a security flaw in Windows Recovery Environment Agent which allowed an un......

   22:52      

In April 2021, I discovered a security flaw in Windows Recovery Environment Agent which allowed an unauthenticated attacker to gain elevated access to a windows machine in a locked state.

Those research were based on Jonas findings related to bypassing lockscreen (you can find more here - https://twitter.com/jonaslyk/status/1301245145568997376?lang=en). He described a flaw, which allowed lock screen bypass using Ease of Access feature.

Looking at CVE-2020-1398, the bug existed in sticky keys pop-up 

[![](https://lh3.googleusercontent.com/-yqcIhJmqDLo/YTEqRqYTZcI/AAAAAAAABQY/VEU27jSlz_U90v-KFOMC-KbDV1snzSWnQCLcBGAsYHQ/s16000/image.png)](https://lh3.googleusercontent.com/-yqcIhJmqDLo/YTEqRqYTZcI/AAAAAAAABQY/VEU27jSlz_U90v-KFOMC-KbDV1snzSWnQCLcBGAsYHQ/image.png)

By clicking the link, an instance of settings will be spawned in the background. Then you’ll be simply able to bypass the lockscreen. Microsoft has patched the issue by removing the link as it no longer appears when being spawned in a lockscreen environment.

And to be clear this bug and its descendants need a condition. On Windows 10 machine, at least one user must have a Microsoft account linked to his local account. Otherwise, the bug isn't exploitable.

Now, I'll try to give a short explanation for you humans. Cause if I showed the video PoC you will be confused as hell.

[![](https://lh3.googleusercontent.com/-6dS1jHb-XhI/YTEr7-jpVeI/AAAAAAAABQg/6iiPpafS1ugerVFR9b_UxafC4k_2FpyiACLcBGAsYHQ/s16000/image.png)](https://lh3.googleusercontent.com/-6dS1jHb-XhI/YTEr7-jpVeI/AAAAAAAABQg/6iiPpafS1ugerVFR9b_UxafC4k_2FpyiACLcBGAsYHQ/image.png)

As you can see above, Windows can allow you to reset your password/pin if you had access to your Microsoft account. If you click on "I forgot my PIN" you will be redirected to something like this

[![](https://lh3.googleusercontent.com/-oLOHzB5yyo4/YTEt3UR3WrI/AAAAAAAABRI/fvSnrpMXnwcn_qJt-3OCrG8cpNsnK9p9gCLcBGAsYHQ/s16000/image.png)](https://lh3.googleusercontent.com/-oLOHzB5yyo4/YTEt3UR3WrI/AAAAAAAABRI/fvSnrpMXnwcn_qJt-3OCrG8cpNsnK9p9gCLcBGAsYHQ/image.png)

I have noticed a weird kind of behaviour when typing a wrong password, a small arrow next to the email address will be visible.  
This behaviour exists for some unknown reason, maybe a bug ? feature ? probably a bug.(apparently its a feature after the patch)

[![](https://lh3.googleusercontent.com/-ov4c9bL2B-I/YTEuZYMrKqI/AAAAAAAABRQ/Yhc4alhup4MK-Dnm05DHtFfQYIVF7l-0ACLcBGAsYHQ/s16000/image.png)](https://lh3.googleusercontent.com/-ov4c9bL2B-I/YTEuZYMrKqI/AAAAAAAABRQ/Yhc4alhup4MK-Dnm05DHtFfQYIVF7l-0ACLcBGAsYHQ/image.png)

Clicking there will take us to another page.As we can see that we’re allowed to login with another email address or even creating a new account.

[![](https://lh3.googleusercontent.com/-C7o253Jui3w/YTEuw8XKvgI/AAAAAAAABRY/tKpRxnsHBcgJFCRI2UxE6hlsV8xN1JlwACLcBGAsYHQ/s16000/image.png)](https://lh3.googleusercontent.com/-C7o253Jui3w/YTEuw8XKvgI/AAAAAAAABRY/tKpRxnsHBcgJFCRI2UxE6hlsV8xN1JlwACLcBGAsYHQ/image.png)

I tried to create a new account, login with it but it fails since the account doesn’t belong to the one we are trying to reset its password.

However, this small button right there attracted my attention and hmmm it's interesting

[![](https://lh3.googleusercontent.com/-fz028CE9nC8/YTEvJocQjzI/AAAAAAAABRg/o1LXm1SuAYMQSqRrTFY5_iHWxMJOnZzdwCLcBGAsYHQ/s16000/image.png)](https://lh3.googleusercontent.com/-fz028CE9nC8/YTEvJocQjzI/AAAAAAAABRg/o1LXm1SuAYMQSqRrTFY5_iHWxMJOnZzdwCLcBGAsYHQ/image.png)

By clicking on it we’ll see another pop-up dialogue, that has a link on it.

[![](https://lh3.googleusercontent.com/-6CIXm8gQD7w/YTEv9aYKhJI/AAAAAAAABRs/O_HMKq31hPcfk2ndKsoVuulJCiTuWsg5gCLcBGAsYHQ/s16000/image.png)](https://lh3.googleusercontent.com/-6CIXm8gQD7w/YTEv9aYKhJI/AAAAAAAABRs/O_HMKq31hPcfk2ndKsoVuulJCiTuWsg5gCLcBGAsYHQ/image.png)

Hmmm very interesting, a link ? in the lockscreen ? weird right. As usual, we’ll click on it and see what happen… clicking on it did absolutely nothing, BUT maybe something was spawned in the background and we can’t see it, as Jonas described in his lockscreen bypass, he used to enable narrator in order to navigate in background apps. I enabled narrator and got some very interesting results.

[![](https://lh3.googleusercontent.com/-haqsinpz9rM/YTEwPfMoxnI/AAAAAAAABR0/tYOOY8qcZrQ-7Pl82AFk2qrYU--fhlEUwCLcBGAsYHQ/s16000/image.png)](https://lh3.googleusercontent.com/-haqsinpz9rM/YTEwPfMoxnI/AAAAAAAABR0/tYOOY8qcZrQ-7Pl82AFk2qrYU--fhlEUwCLcBGAsYHQ/image.png)

When enabled and I click on the button, you can hear narrator saying “how do you want to open this”, and narrator’s focus is on something else not in Microsoft account window. We spawned an “Open With” window with narrator’s focus on it in the background; Typically the “Open With” window looks like this

[![](https://lh3.googleusercontent.com/-76a1NJpoXNE/YTEwfcsqYLI/AAAAAAAABR8/dUmO6-s3TCgEKxLbPhPm-fOQQK1CS6tmACLcBGAsYHQ/s16000/image.png)](https://lh3.googleusercontent.com/-76a1NJpoXNE/YTEwfcsqYLI/AAAAAAAABR8/dUmO6-s3TCgEKxLbPhPm-fOQQK1CS6tmACLcBGAsYHQ/image.png)

But only has two options the first is MS Edge and the second one is Internet explorer, we’ll dig with MS Edge since it’s selected by default, please **NOTE** that you might to **HOLD** Caps lock while using arrow keys to navigate.

After tests, as soon as we select OK we lose narrator’s focus and we’re no longer able to control background window.

We can have narrator’s focus again as soon as we repeat steps described above, we’ll have narrator’s focus again. But this time we’ll have it on MS Edge browser, at this point, we will need to elevate our privileges, the only way I can think of to execute arbitrary commands is to spawn a settings instance. This can be done by spawning another a new InPrivate window, (please NOTE: you won’t be able to see any of those, and things will be completely invisible you must use your ear to hear what narrator say and use it to navigate);

[![](https://lh3.googleusercontent.com/-mbMdkev3W6g/YTExCGe-EsI/AAAAAAAABSE/5dg73C94m383IhuB9JW0PgntxTr1j32kQCLcBGAsYHQ/s16000/image.png)](https://lh3.googleusercontent.com/-mbMdkev3W6g/YTExCGe-EsI/AAAAAAAABSE/5dg73C94m383IhuB9JW0PgntxTr1j32kQCLcBGAsYHQ/image.png)

Then you might need to go on “More details”

[![](https://lh3.googleusercontent.com/-VZBC0wq1zLM/YTExKMFTUnI/AAAAAAAABSI/_tAEcnKr9FIGLfkHyEadCRQQo9-cAGcHQCLcBGAsYHQ/s16000/image.png)](https://lh3.googleusercontent.com/-VZBC0wq1zLM/YTExKMFTUnI/AAAAAAAABSI/_tAEcnKr9FIGLfkHyEadCRQQo9-cAGcHQCLcBGAsYHQ/image.png)

Press enter and navigate to settings

[![](https://lh3.googleusercontent.com/-80cwSMrYB9o/YTExTN5jg0I/AAAAAAAABSQ/d4xq36BUETMVDJ9EZNO9SEXZbZT46A4TwCLcBGAsYHQ/s16000/image.png)](https://lh3.googleusercontent.com/-80cwSMrYB9o/YTExTN5jg0I/AAAAAAAABSQ/d4xq36BUETMVDJ9EZNO9SEXZbZT46A4TwCLcBGAsYHQ/image.png)

Which will redirect us to another page, keep navigating until you reach “Windows Diagnostic data setting” and then navigate using narrator to open and click enter again

[![](https://lh3.googleusercontent.com/-FzRPZs6SsEA/YTExa4guG1I/AAAAAAAABSY/rGBfonHen8oDBDm0qximCHxjSwRXN_8XgCLcBGAsYHQ/s16000/image.png)](https://lh3.googleusercontent.com/-FzRPZs6SsEA/YTExa4guG1I/AAAAAAAABSY/rGBfonHen8oDBDm0qximCHxjSwRXN_8XgCLcBGAsYHQ/image.png)

In settings navigate to “Home” and press enter

[![](https://lh3.googleusercontent.com/-MCpdqXd2Cq4/YTExhaqjUoI/AAAAAAAABSc/k6djdNrzZL0EEnL7OIHnX29aQrr2RZiTACLcBGAsYHQ/s16000/image.png)](https://lh3.googleusercontent.com/-MCpdqXd2Cq4/YTExhaqjUoI/AAAAAAAABSc/k6djdNrzZL0EEnL7OIHnX29aQrr2RZiTACLcBGAsYHQ/image.png)

Then navigate to “Devices”

[![](https://lh3.googleusercontent.com/-kWhhzU7EQV8/YTExmemuidI/AAAAAAAABSk/-LQfD8-m4m0_mSlOiNdZ5mLzCE7ATtO-wCLcBGAsYHQ/s16000/image.png)](https://lh3.googleusercontent.com/-kWhhzU7EQV8/YTExmemuidI/AAAAAAAABSk/-LQfD8-m4m0_mSlOiNdZ5mLzCE7ATtO-wCLcBGAsYHQ/image.png)

Navigate to Autoplay->Choose Autoplay Defaults->”Open folder to view files(File explorer)

[![](https://lh3.googleusercontent.com/-NjjOwtqGGvA/YTExr3uEbFI/AAAAAAAABSs/pieqYldHQ10SudssUe5m9s_i_-6sN9aBwCLcBGAsYHQ/s16000/image.png)](https://lh3.googleusercontent.com/-NjjOwtqGGvA/YTExr3uEbFI/AAAAAAAABSs/pieqYldHQ10SudssUe5m9s_i_-6sN9aBwCLcBGAsYHQ/image.png)

At this point, you might need to plug a USB device into the device. As soon as plugged narrator will have his focus on file explorer, now you can execute anything in the USB.  
In order to verify our findings, I made a simple batch script which will verify our findings  

mkdir c:\poc  
whoami /all >  c:\poc\whoami.log

And after the execution, we can observe a success

[![](https://lh3.googleusercontent.com/-xzkDW0uERUY/YTEyH5LOf8I/AAAAAAAABS4/cjqyTMbL7m43CS2IFu4fT77DhnO9uwqIACLcBGAsYHQ/s16000/image.png)](https://lh3.googleusercontent.com/-xzkDW0uERUY/YTEyH5LOf8I/AAAAAAAABS4/cjqyTMbL7m43CS2IFu4fT77DhnO9uwqIACLcBGAsYHQ/image.png)

Elevating privileges is easy since we’re marked as “NT AUTHORITY\Authenticated Users” as the majority of EoPs are reachable from those privileges.

PoC - https://youtu.be/9rXXfWN0h6A

(Also, looking for some opportunities to study computer science in UK or anywhere else, if you can help reach me out on my twitter.)