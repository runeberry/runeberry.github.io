---
layout: blog-post
title: Keycosystem
---
I spent a majority of the weekend working on an input management system for MonoGame.

I had a couple of goals in mind for this input management system:

* Generalize inputs so that there is no code difference between programming for a keyboard key, gamepad button, joystick, trigger, mouse cursor, or scroll wheel. Or hopefully any other type of HID output that you can think of.
* Make it extremely simple to bind actions to keys, and to rebind them at runtime. I want to be able to bind Input -> Action in one line, and it just _works_.
* Make it framework-agnostic. I'm planning on using MonoGame, but I want to be able to swap out all the vital parts that MonoGame uses (i.e. reading inputs from devices) with something else without having to rewrite the underlying engine. Or at least with only needing a few minor tweaks.

In order for any of these to happen, I need to figure out what these different types of input have in common. Here's what I came up with.

```csharp
public class KeyState : IReadOnlyKeyState
{
    public KeyState(int key)
    {
        this.Key = key;
    }

    public int Key { get; }

    public bool IsPressed { get; set; }

    public long IsPressedLastChange { get; set; }

    public int DigitalX { get; set; }

    public int DigitalY { get; set; }

    public long DigitalLastChange { get; set; }

    public float AnalogX { get; set; }

    public float AnalogY { get; set; }

    public long AnalogLastChange { get; set; }
}
```

Going back to [this comment](http://xboxforums.create.msdn.com/forums/t/2189.aspx#12431) which I referenced in a previous post, I wanted a representation that would work for both digital and analog inputs. There seem to be 3 valuable types of measurements when we're looking at user input, and in closer detail:

* __IsPressed__: Obviously for buttons and keyboard keys, we simply want to know whether or not they're pressed.
* __DigitalX/Y__: For some types of controls like d-pads, we want to know how something is positioned in discrete units along two axes. For example, a digital (1, 1) might mean "up-right", while (-1, -1) might mean "down-left". In addition, mouse cursor position and scroll wheel values are expressed in discrete units. At least they are in MonoGame, which is what I'm using for reference in building this.
* __AnalogX/Y__: Joysticks provide input on a smooth scale along two axes. This could be expressed as (1.0, 1.0) for fully-extended to the top-right, or maybe (-0.5, 0.0) for halfway-extended to the left. Some triggers (see Xbox controllers) provide smooth input along one axis, as well, but other triggers (see Switch Pro controller) are digital buttons.

I would expect any given input to only provide one of these three measurements. But the beauty of this model is that you can set all three measurements for any given one input, if you establish some rules on what that translation looks like. Here's how my implementation looks at the moment:

* __IsPressed__: If a button is pressed, its DigitalX/Y are 1 and AnalogX/Y are 1.0. Essentially, "everything is 100% on".
* __DigitalX/Y__: A d-pad is considered "pressed" if any of its buttons are depressed (i.e. if DigitalX or Y are non-zero). AnalogX/Y simply matches the digital values, but in floating-point form - a digital press is the equivalent of being "100% on". Mouse cursor and scroll wheel inputs do not have an IsPressed equivalent. The scroll wheel button, when available, is considered a separate input, usually "Mouse Button 3".
* __AnalogX/Y__: A joystick is considered "pressed" whenever it's not sitting in its deadzone. Stick buttons like L3/R3 are considered buttons and handled in their own right. These values can round up or down to a DigitalX/Y of 1 or -1 whenever they're pointing in the appropriate direction and outside of the deadzone.

On top of that, the gametick on which an input changes is recorded along with the data, so that deltas can be calculated by the consuming function if needed. I won't get into the details of how the binding and action triggers work tonight, but the [code is all on GitHub now](https://github.com/dolphinspired/dolphengine/tree/1d4f33b1eb2ce42bfb3e172a55ea1e5d5decf496), so feel free to take a look. I've linked the current latest commit so that code will stay relevant to this post.

---

I've been able to code something that meets the three goals outlined above using this model, but I think there is still some room for improvement. Some thoughts:

### Class vs. Struct

As you can see in the code above, I made KeyState a class with a read-only interface. The updating system uses KeyState directly, while actions that react to the key's current state will only get the IReadOnlyKeyState.

The other option would be to have a struct with readonly properties. But because this KeyState is cached in multiple dictionaries and iterated over every update loop, I didn't want to copy the entire value type or replacing items in the dictionary that frequently. Referencing and updating the properties seems like it would be more performant in the way this particular object is used. I think I'll need some benchmarks to know for sure - but I probably won't change this unless there's a significant performance gain.

3rd option, class with mutable fields? That would make for a gross API, which is kind of what I set out to avoid, but it _could_ be pretty fast... or maybe not noticeable at all.

### Input combinations

Right now have programmed special inputs into my MonoGame implementation of this system which are not a part of MonoGame itself. For example, you can bind a control to "DPadUp" or "RightThumbstickX" (controls that MG identifies), but I've also added "DPad" and "RightThumbstick", where you get one KeyState that represents all axes associated with that control. DPadUp would simly act as a button, where DPad acts as a 2-axis digital control. RightThumbstickX acts as one analog value on the X-axis, while RightThumbstick observes both RightThumbstickX and RightThumbstickY to give you a comprehensive KeyState that reflects both.

This works great in code, but I worry that it's blurring the line between KeyState reflecting "the state of an input" and a "derivative of several inputs". Just because it works for my specific case doesn't necessarily make it a great generic control. Someone else might want to combine both thumbsticks into one control, and that won't work because I only have two analog values in KeyState.

I'm also afraid might not be scalable enough to include something like modifier keys. I don't have a way yet to bind something to Shift+C, for example. How would I do that? Make 170-ish additional key ids for "Shift+{every keyboard key}"? Then 170 more for Ctrl, and 170 more for Alt? What about combining them with mouse buttons? I'm

Here's the [current list](https://github.com/dolphinspired/dolphengine/blob/1d4f33b1eb2ce42bfb3e172a55ea1e5d5decf496/DolphEngine/Input/Implementations/MgInput.cs) of generic constants I have defined for MonoGame's inputs, along with some combination inputs that I made up. Every new combination I add results in additional code being written [here](https://github.com/dolphinspired/dolphengine/blob/1d4f33b1eb2ce42bfb3e172a55ea1e5d5decf496/DolphEngine/Input/Implementations/MonoGameObserver.cs#L137). It doesn't look like a good road to go down.

### Reading text strings

Expanding upon the user input example above, I would really like to have a method that could listen to keyboard input and build a string based off of it. But that is a very PC-specific requirement. How do you hook into game console keyboards? I'm wondering if I could build an interface that would work with PC or console keyboards, but I have my doubts because they're so incredibly different.

I'm thinking of Stardew Valley as an example, since I've played it recently and it also uses MonoGame. When you input text on a PC, the game keeps running and just listens for keyboard input, and it updates the displayed string as you type. But when playing on a console (at least on Switch), the game completely pauses and waits for your input to be provided from the console's keyboard, upon which it's displayed all at once. Is there a way to consolidate these two behaviors into something generic? Has it already been done?

The more I think about it, the more I'm starting to think I need an intermediate layer to group KeyStates together. I bet I could shrink the KeyState down to just one digital and one analog value (along with "last change" ticks), and use a grouping class of some sort to tie them all together an interpret what the keys mean in bulk.

---

The next step is probably going to be addressing the key-combos problem in a better way, then getting a rough demo going. If the API proves nice to use, then I'll move onto unit testing the heck out of it. I would love to wrap up input in the next 2 weeks, but that'll probably depend on how deep I delve into this key-combos thing. If I don't post back in 2 weeks, send help.