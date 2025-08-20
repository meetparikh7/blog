---
title: Introducing waylockrs & rust-y lessons learnt
description: waylockrs is a wayland screen-lock utility
date: 2025-07-25
tags: Rust, Wayland
---

If you use a Wayland compositor that is not part of a Desktop environment like GNOME or KDE, you have to install a bunch of utilities for doing desktop stuff - like toolbars, launchers, etc. A lot of these utilities are written in C. I implemented a lock screen utility called [waylockrs] for such compositors in Rust, and this post explains the why, how and some small Rust learnings.

I had not developed an application using Wayland or the modern Linux Graphic stack before and it was a fun learning experience for me. This post should not require any Linux graphic stack knowledge but hopefully sheds some light on it.

# Why

[swaylock] is a popular screen locking utility for wayland. In the `ext-session-lock-v1` wayland protocol, which is used for this, the screen lock utility signals the compositor that the user has succesfully authenticated. This means there is flexibility in implementation of the `session-lock`, i.e. it can be a screen saver or a lock-screen or a captcha challenge for when the robots take over. It also means, that if there were any bug in such a lock screen, the user can gain access to your computer, and say access your email or your active SSH sessions, or worse, the top-secret project plan that you have been procastinating over for a year.

Unfortunately the attack surface is rather large for a "practical lock-screen", something that displays a image, maybe the clock and some typing indicator. There is a possibility of a buffer overflow or use-after-free attack to get the lock screen to send the unlock signal to the compositor. My aim was to write a drop-in replacement for swaylock in Rust, which can then be improved upon. As of now, swaylock is probably much more safer and hardened by years of development.

# The moving parts

We use the following libraries to implement [waylockrs]. (Feel free to skip this section
)

## Wayland & Smithay Client Toolkit

Wayland itself works as a client-server model. The server is the wayland compositor and the client is the application, in this case the lock-screen. They talk in terms of messages and object handles. The object handles can be created and released by the client and the server frees them when they have no dangling references (like an `Arc<_>` but across server/client). The messages can refer to these objects, for example `mouseA` moved `10px`.

Wayland is split out into protocols, each dealing with some aspect of a desktop, say one for monitors, one for input, one for windows and so on. On startup of an application you are provided a global object which lists all the protocols available, and you chose to instantiate an object for each protocol you care about.

The [SCTK](https://github.com/smithay/client-toolkit) and other Smithay crates abstract away the wayland protocol. Typically when using the SCTK you have a global state object and then you implement a bunch of traits on them, one for each wayland protocol you want to handle. Then these trait handlers can be called in an event loop model. More details can be found in the [Smithay book](https://smithay.github.io/book/intro.html). They even have a [helpful example](https://github.com/Smithay/client-toolkit/blob/master/examples/session_lock.rs) which locks and auto-unlocks the screen in 5 seconds.

## PAM

Most modern linux distros ship with PAM aka Pluggable Authentication Modules. This C library represents a Linux/BSD API allows you to authenticate the user and do more like fingerprint auth etc. For the purpose of authentication, it requires you to create a context object and pass a bunch of function pointers that the OS will call, for prompting for password etc. Luckily the [pam-client](https://crates.io/crates/pam-client) crate abstracts away the C API for us.

## Cairo

Wayland is concerned around displays, surfaces and compositing, i.e laying out different windows on the screen. It does not give us drawing primitives like circles or textures, but instead gives you buffers of whatever size you request and allows you to paint those buffers on a "surface" which can be a (part of a) window or in the case of the lock screen the entire display.

A lock sreen without a background image or a typing indicator is not super user friendly. This is where graphics libraries like Cairo & Skia come in. They are the same ones used in browsers to render web pages. Cairo is the more user-friendly one and has friendly Rust [bindings](https://docs.rs/crate/cairo-rs/latest). It also allowed me to port over swaylock's drawing code much easily (as it also uses Cairo).

# The learnings

## Double Buffering

Wayland and other graphics stuff like games in general use two memory buffers - one which is shown on the screen, and the other one which is being rendered too. Once rendering is finished, you "commit" the new buffer and the compositor choses when to display it.

For example if you were able to prepare the frame faster than the screen refresh rate, the compositor will wait for the next refresh to wait. If you had asked for the frame callback, the compositor would "call you" (emit a message in the channel) when this frame was drawn.

Since the compositor is always displaying a buffer that is stable and not being mutated, there is no screen tearing or flickering.

However implementing this is somewhat hard. A Wayland client can have multiple surfaces. Each surface can have subsurfaces. Subsurfaces maybe commited and "damaged" async, which means you have the power to not destroy a surface if it has not changed. We also need to handle the "configure" callback which tells us the surface size, when the window (or say the monitor resolution) changes. This callback is the one which actually lets us allocate the buffers as we now know the size of the area we are drawing and the number & size of channels.

For example in our lock screen, we have a background subsurface that displays an image and an indicator subsurface which displays the current time and a typing indicator. We ideally want to draw them async and not destroy the background subsurface unless the monitor resolution or window size has changed.

Thus the logic is somewhat like:

1. Wait for the configure callback to allocate buffers
2. On configure, configure the surfaces and then invoke the draw function
3. In the draw function, for each subsurface:
	1. Write a functions that draw to the surface given the information `buffer, width, height, did_resize`. Here `buffer` should be the buffer not "attached" (not being used) by Wayland.
	2. This function can be smart and chose to not render if `did_resize = False` and/or there are no UI changes
	3. Thus this returns a boolean, which if `true`, call the wayland proxy objects to destroy the subsurface and commit the new buffer
	4. If a render did happen and a new frame has not been requested by another subsurface request a new frame.
4. When the new_frame callback happens call the draw function again from the application

Rust gives us the power to use `Option` to represent a lot of this state, especially when a buffer is configured or not, and has first class functions, which when taken as generics have no cost. Using this we are able to abstract away the logic in step 3 into a neat [`easy_surface`](https://github.com/meetparikh7/waylockrs/blob/main/src/easy_surface.rs) struct.

## `secstr` and how to represent the password buffer in memory

Representing the password in memory should be done with a "few security features":

1. On the password buffer being "used", i.e. on "drop", the resident memory should be zero-ed out
2. Ideally the password buffer should never be written to swap memory
	- swap memory acts as a cache in case the current memory overflows
	- It is usually backed by a disk or SSD, i.e. non-temporary storage
	- Even after the password has been used, and if it has been paged to swap storage, the disk will contain the password bytes

Linux provides [`mlock`](https://linux.die.net/man/2/mlock) / [`munlock`](https://linux.die.net/man/2/munlock) syscalls which prevent a chunk of memory from being paged out to swap. We need a container that can represent a string / vector of bytes and that should invoke these syscalls.

The flow here looks like:
1. Allocate a chunk of memory
2. On keypress, try to append some characters to the buffer
3. If the buffer does not have capacity
	1. Allocate a new buffer with enough capacity and `mlock` it
	2. Copy the original buffer and append the new characters
	3. Zeroize the previous buffer, `munlock` it and then release it
	4. Point the content to the new buffer
4. On `enter` key press, send this buffer to `PAM` which would handle freeing it.

Luckily the [`secstr`](https://crates.io/crates/secstr) crate handles this for us. However it does not provide all the methods available on strings as the other use-case for this is to load immutable keys / tokens in memory. We wrap around the `SecVec` provided within it to handle password entry user interactions, i.e. append new characters and backspace. On password enter we hand it over to the pam part.

## `PAM`, threading and channels using `calloop`

The Smithay crates operate in an event loop model which they call [`calloop`](https://docs.rs/calloop/latest/calloop/index.html). This exposes channels similar to [`std::sync`](https://doc.rust-lang.org/std/sync/mpsc/fn.channel.html) but that can interact with automatically emit an event in the `calloop` instead of blocking the thread.

The flow here is:
1. On startup the application creates a thread
2. This thread initiates a PAM context to authenticate the (signed in) user
3. The PAM and main threads communicates via a calloop channel
4. The main thread inserts this channel as a "source" in its calloop with a callback that will be called if there is a message
4. When the user presses the "enter" key, a message to authenticate the user is sent via this channel
5. The PAM thread on receiving this message will:
	- Try to authenticate the user via PAM
	- Note this is a blocking call and PAM will add additional delay if there have been too many incorrect password attempts
	- On response from PAM the auth state - success / failure is sent back to the main thread on the channel
6. While the PAM thread is authenticating, the main thread continues to run the calloop
	1. The main thread calloop keeps processing wayland events, continuing to render - displaying an "authenticating" message, updating the clock, etc
	2. If the auth channel has a new message, the corresponding registered callback is called which can access the application state and unlock the `session_lock` or set state to invalid password, which would be displayed in the next render.

The calloop model along with channels provides and easy way to tie together blocking and unblocking APIs without having to deal with Rust async which can be troublesome with the borrow checker.

## AI and backporting settings from swaylock

One of the main features of the custom DE dream is customizability, which means a decent amount of config options. I wanted a modern [TOML config](https://github.com/meetparikh7/waylockrs/blob/main/defaults.toml) for waylockrs instead of the CLI arguments is config adopted by swaylock. Note that the TOML arguments we use are still primarily strings / booleans so they can be configured bia CLI arguments.

However I wanted to ease migration from swaylock and automatically port a swaylock config if one is detected. This requires parsing the swaylock config man page, finding out the corresponding keys in waylocksrs' TOML config spec and create a mapping.

While I dont think LLMs as general purpose codign agents are great, LLMs excel at such small tasks, involving grokking small amounts of data and producing maybe invalid but still good enough code, especially if enough documentation / comments / language (i.e. English) are involved. I was able to quickly create a mapping between the two configs put it up in a separate module and easily port over swaylock configs.

# Summary

[waylockrs] is avaialble to try out via DNF [Copr] or via building from source. Do report issues and feedback on the [tracker](https://github.com/meetparikh7/waylockrs/issues).

Hopefully this blog was a good introduction to Wayland / PAM and Smithay! This was my first foray into the ecosystem, let me know your comments or if I made a mistake!


[waylockrs]: https://github.com/meetparikh7/waylockrs
[swaylock]: https://github.com/swaywm/swaylock
[Copr]: https://copr.fedorainfracloud.org/coprs/meetp7/waylockrs/
