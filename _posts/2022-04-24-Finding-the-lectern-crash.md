# Finding the lectern crash
A few weeks ago, a crash involving shift clicking an item into the lectern block, while the handled screen of the lectern was opened, was found. This is how it was discovered
## Background
Minecraft has "handled screens", which are essentially just GUIs bound to a block, that can modify the block's state. This is used for inventories in chests, for example. Or interacting with furnaces.
### The bug
The lectern conveniently also has that screen bound to it, but in a slightly different way

## Looking into it
![](https://s2.loli.net/2022/04/24/qm2lbjOMnwLerVv.png)

Looking at this image, we can see what happens when we click the lectern block. At the 2nd IF-instruction, we can see that this checks if we are on the server. If it is, we then send the screen of the lectern to the client opening it. Let's see what happens in there.

![](https://s2.loli.net/2022/04/24/sky4Y2JNoxULO7p.png)

In here, we can see, that the screen is a handled screen, which is always a bit wonky. Let's see how this screen is implemented then.

The function opens a new HandledScreen, supplied by the LecternBlockEntity class. Let's look into that ScreenHandler then.

## The screen handler
The screen handler in the lectern HandledScreen is a bit weird, since it doesn't have too much code. This also means, that it does not take any sort of clicking that was not intended into account. Let's look at what happens when we shift click, for example.


![](https://s2.loli.net/2022/04/24/u4a1BHejm3ViTGK.png)

First step: get into the internalOnSlotClick method

![](https://s2.loli.net/2022/04/24/cYR6wCpH8zOJlUW.png)

Second step: get into that exact if statement, by setting our button id to 0 or 1, and our action type to QUICK_MOVE

At this point, this is guaranteed to crash. Why? Because of that one line that comes after that if statement

![](https://s2.loli.net/2022/04/24/ru9JOndiKTzWyX4.png)

This for statement has 3 properties: initial condition, should progress and after progressed. Two parts are very interesting tho, the initial condition and the "should progress" instruction. The initial condition just gives us a reference to the already put in book, because the LecternScreenHandler does not override the transferSlot() method, and the default just returns whatever is in said slot of the ScreenHandler. Then, the "should progress" stage, this is where it gets interesting.

We check if the stack is empty, which it is not because it has the written book we are viewing in it, and then check if the stack is equal to the stack we have in the slot, which as we know, is the same. So this just loops over and over, because the "after progressed" part **ALSO JUST TAKES THE SLOT ITEM AND PUTS IT INTO THE REF**. Basically, we are creating an infinite loop that never exits, just with one packet sent. **This is how the crash works.**

## The fix
You can fix this easily by just not allowing shift clicks into the HandledScreen, or disallowing the player to take said slot out of the inventory. It is such a simple fix that I cannot believe this still works on vanilla.
