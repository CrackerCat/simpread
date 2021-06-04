> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [guidedhacking.com](https://guidedhacking.com/threads/il2cpp-is-for-kool-kids-only-android-il2cpp-breakdown.17580/?__cf_chl_captcha_tk__=3ef2e52eb3ffab51955b9486e59c7dc01128b4eb-1622769294-0-AeYS5yN7QP-jDaKgKhD_X_Tb2lYXMQda2UFF5X6hkUOjVfGXiF0b4u0BdQptf_BvjuTyNvQ9UidkCzC4HPI0LTJwVYPMekkOQ_XNxoiBFaV2ZODQyJRe1DxEHPHgKhXQbz3YyDQd34X0L036z68kBgXC9p4jue2BnOWvUhXstpzpcXtsVx4XO7LI0T79QlKlG_HKk6c0LZfbEbgVctqFGiLHcupRBdYPt5ioiXaRxEObtiqpHiAFSx9bwqs_VCfPANtzXV2aw0JbsKrWSxq-UzLq8nrkiGUdAr6NJtazlPLIOy1gKG1y5X-_1naCO0zKepKHvt0clQm13u-Em1FJqf1ESil2mAwm2DKE1S0eFg8uXHh0b3vSg2YRvMTLDkmoErxSTsRA-RmLETviUMILYz9y1X1Cv2-AwOWlu-Cy7PYtLMFzzEU2a1PmLM6GKMpm_e3m117zGUEjKx57J80IEK0fPjBRQzKsFCguq1WvzVuKmT_pPw8OG2bKvMXm3-H9oaziPw3EQ6c5A9bU9zRBeMTcRSUUlSXxWXlczExFhtbnKu2jREiimZqA429lyWvb995851tBLxETyykThjviHPMBv71-fY-TKCpEPjKpFTIHmMyt75rt17IWXpP8f0ii6SxgrkGBRjmGDDynC737J3NyDYYL-q9l76rFCZtGPasCsirilBIQWT3S1Ak5GsecQ1JUo59k7sGyP8lUcJBptpusb2KKn5srI5AJiku0HFGqrR5A12FZ7Nw7-K7HJNSJBQ)

> Game Name: Forward Assault Anticheat: Codestage and a signature check. How long you been coding/hacki......

 [![](https://guidedhacking.com/data/avatars/s/243/243988.jpg?1620571796)](/members/cryoatomizer.243988/) 

#### [cryoatomizer](/members/cryoatomizer.243988/)

**Full Member**

May 8, 2021

3

138

0

[Today at 11:35 AM](/threads/il2cpp-is-for-kool-kids-only-android-il2cpp-breakdown.17580/post-110939)

*   [#1](/threads/il2cpp-is-for-kool-kids-only-android-il2cpp-breakdown.17580/post-110939)

Game Name: Forward Assault  
Anticheat: Codestage and a signature check.  
How long you been coding/hacking? 1 year.  
Coding Language: C++  
So, after my previous [Function Pointers Tutorial](https://guidedhacking.com/threads/a-more-in-depth-tutorial-on-function-pointers.17476/) , I decided that a in-depth explanation of useful unity core-modules functions would be quite nice.  
Since most of the good android games are il2cpp, and also because of the fact that il2cpp/mono games are the starting point for android cheat developing, I think it deserves a mini "documentation" on useful functions in the core-modules dll that you get after dumping the game.  
Core-Module functions are very useful for a variety of things including:  
Esp  
Aimbot  
Player Lists.  
Those are some common ones, but there are a few more I will discuss in this thread.  
**Things you need to follow this tutorial:  
-Knowledge of C++  
[Octowolves template](https://github.com/octowolve/Hooking-Template-With-Mod-Menu)  
Or  
[LGLS TEMPLATE](https://github.com/LGLTeam/Android-Mod-Menu)  
(if you prefer another template that's also fine of course)  
-Basic knowledge on making mod menus for android  
-Know and be able to make [Function Pointers](https://guidedhacking.com/threads/a-more-in-depth-tutorial-on-function-pointers.17476/)  
-A brain**  
Assuming you have these, move on to the documentation. If not, f*ck off, I don't need people who ask questions without even doing the required steps before following a tutorial.  
Please don't ask me questions unrelated to this tutorial in the comments. If you want to learn about making menus for android, read the other threads in this section or get the f*ck out.  
For this tutorial ill be using the game [Forward Assault](https://play.google.com/store/apps/details?id=com.blayzegames.newfps&hl=en_US&gl=US). No, I wont show you how to bypass the signature check that kicks you from the game, if you want to still test it, just play with bots instead of online. Code stage does nothing now days, so I wouldn't worry about it. Please don't go in the comments and say "iT dOeSnT KiCk mE fRoM tHe gAmE". The check could be removed at any time.  
Alright, Alright, lets get to the tutorial.  
Dump the game(ofc)  
Load the dlls into dnspy and find UnityEngine.CoreModule.dll (it might be called something different, just look for core modules)  
To further check if you are in the right dll, look for class Transform or Camera.  
I will split this into sections related by class.  
Section 1: Class **Transform**  
Probably the most useful class imo.  
Function:

```
private extern void get_position_injected(out Vector3 value);

```

This function lets you get the position of any object/transform.  
Commonly used in esp and aimbot.  
get_position (in the same class) is an alternate to this function.  
Usage:  
Make a **[Function Pointer](https://guidedhacking.com/threads/a-more-in-depth-tutorial-on-function-pointers.17476/)**  
Usage:

```
get_position_injected(enemyplayer);

```

Normally this wont work, you will have to get the position of a transform, using get_transform in class Componant.  
Ill go over get_transform later in this tutorial, but incase you cant wait, you can make this function to get the position of a player using a transform.  
Function:

```
Vector3 GetPlayerPosition(void *component){
    Vector3 out;
    void *transform = get_transform(component);
    get_position_Injected(transform, &out);
    return out;
}

```

With that function you can get the position of a player easier, instead of having to pass in get_position_injected and get_transform every time you need to get the position of something.  
public void set_rotation(Quaternion value);  
you can use this to set the rotation of your player to something else, for aimbot.  
Usage:  
Make a **[Function Pointer](https://guidedhacking.com/threads/a-more-in-depth-tutorial-on-function-pointers.17476/)**  
Again, to use this, you also need to use get_transform from class componant, and get_main from class camera.  
Usage:

```
set_rotation(Component_get_transform(get_main()), customrotation);

```

If the game doesnt have an instance variable of your players rotation, this function is very useful.  
Function:

```
private extern void set_localScale_injected(ref Vector3 value);

```

or  
Function:

```
public void set_localScale(Vector3 value)

```

These 2 functions both work to change the size of a player. In some games you can shoot the big player, in some games you cant.  
Usage:  
Make a **[Function Pointer](https://guidedhacking.com/threads/a-more-in-depth-tutorial-on-function-pointers.17476/)** for the function  
We can make a function that makes setting the scale to a player easier using get_transform.  
C++:

```
void setScale(void *component, Vector3 scale) {
    void* transform = get_transform(component);
    set_localScale_Injected(transform, scale);
}

```

Then, to use this function, we need to call it in update, and set the scale of a player or object.  
Like so:  
Call Function:

```
void *MyPlayer;

void(*old_Update)(void *instance);
void Update(void *instance)
{
    if(instance){
        if(get_isMine(instance)){
            MyPlayer = instance;
        } else {
                if (isScale) {
                    setScale(instance, Vector3(40.0f, 40.0f, 40.0f));
                }
            }
        }
    }
    old_Update(instance);
}

```

Ive also excluded my player from getting scaled up, but if you want yours to be scaled up, feel free to remove the check.  
Well, thats about it for class transform, but ill leave some honorable mentions for some more functions in it, but I wont be demonstrating their usage.  
1. Lookat (aimbot)  
2. Forward  
3. Eulerangles  
Section 2: Class **Camera**  
Class Camera doesn't really contain a lot of player related things, but rather a lot of things useful for esp, or sometimes aimbot.  
Function:

```
public static extern Camera get_main();

```

This is a function that gets your main camera, it can be used for things like aimbot with set_rotation, or getting your main camera for WorldToScreen in esp.  
Usage for aimbot:  
Make a **[Function Pointer](https://guidedhacking.com/threads/a-more-in-depth-tutorial-on-function-pointers.17476/)**  
Usage:

```
set_rotation(Component_get_transform(get_main()), customrotation);

```

Usage with esp will be covered later in this thread. If you need it for anything else, just call it as a function, it can fit anywhere.  
Usage:

```
public static extern Camera get_current();

```

An alternate to get_main, this function gets your current camera instead. This should only be used incase get_main isnt working. If both dont work, look for an instance to the camera in your players class.  
Function:

```
public Vector3 WorldToScreenPoint(Vector3 position)

```

This function is golden. Amazing. It is unity's WorldToScreen! If you dont know what WorldToScreen is, dont even bother reading this part. But for those who do, you know how valuable this is.  
Usage:  
Make a **[Function Pointer](https://guidedhacking.com/threads/a-more-in-depth-tutorial-on-function-pointers.17476/)**  
Usage:

```
Vector3 WorldToPlayerPosition = WorldToScreenPoint(get_main(), current->Location);

```

WorldToScreenPoint_Injected is an alternate that works fine as well, and you can even make a function to make it easier to call.  
Function:

```
Vector3 WorldToScreen(Vector3 in){
    Vector3 out;
    WorldToScreenPoint_Injected(get_main(), &in, 2, &out);
    return out;
}

```

2 is the default eye for unity.  
Usage with the function above:  
Usage:

```
Vector3 WorldToPlayer = WorldToScreen(current->location);

```

Honorable Mentions for this class.  
get_projectionMatrix  
get_worldToCameraMatrix  
These 2 can be used to calculate your viewmatrix, for a custom WorldToScreen.  
Function:

```
public extern void set_fieldOfView(float value);

```

This function, for most games, sets your gun or camera fov. Commonly used if you cant find a field/different function to your players fov.  
Usage:  
Make a **[Function Pointer](https://guidedhacking.com/threads/a-more-in-depth-tutorial-on-function-pointers.17476/)**  
Usage:

```
set_fieldOfView(100.0f);

```

Section 3: Class Component  
This class only has one actual "useful" function.  
As mentioned earlier, when getting positions we must use get_transform.  
Function:

```
public extern Transform get_transform();

```

Usage:  
Make a **[Function Pointer](https://guidedhacking.com/threads/a-more-in-depth-tutorial-on-function-pointers.17476/)**  
Usage:

```
Vector3 guidedhacking = get_position(get_transform(enemyobject));

```

Or, plug it into a function to make getting our position easier, mentioned above in Section 1, function "get_position_injected".  
Section 4: Class CharacterController  
C++:

```
 public extern void set_radius(float value);

```

Credits: [Unity Universal Noclip](https://github.com/jbro129/Universal-Unity-NoClip)  
Usage:  
Make a **[Function Pointer](https://guidedhacking.com/threads/a-more-in-depth-tutorial-on-function-pointers.17476/)**  
Get your character controller  
Usage:

```
set_radius(CharacterController, 100.0f);

```

Those are pretty much all the functions I've found to be useful, if you want a function added to this, comment or shoot me a pm, Ill add it as long as I find it of value, and give you credits.  
I highly advise you to use or take a look at these functions when you come across an il2cpp game, (most of you probably still mess with them/ used to when you were learning), there are quite a few good il2cpp games out there for beginners and for experienced people. These functions save you quite a lot of time, and are almost necessary for things like aimbot or esp. Most of these functions are straight-forward to use, and should be pretty easy once you are comfortable with hooking and function pointers.  
If you have any questions, (skids and pasters f*ck off please, you smooth brain headasses aren't needed anywhere), shoot me a pm, or leave a comment. Ill be happy to respond to anyone who respects my time and isn't someone who asks a question that could be answered by google. I am not your f*cking personal robot.  
I plan to post many more in-depth tutorials on the android section related to mod menus, just be patient, as I am working on other things, but its the summer, so expect quite a few more tutorials when I set my mind to it.  

Last edited: Today at 3:34 PM

*   ![](data:image/gif;base64,R0lGODlhAQABAIAAAAAAAP///yH5BAEAAAAALAAAAAABAAEAAAIBRAA7)
*   ![](data:image/gif;base64,R0lGODlhAQABAIAAAAAAAP///yH5BAEAAAAALAAAAAABAAEAAAIBRAA7)

Reactions: [MrNatas1, Petko123 and Rake](/posts/110939/reactions)