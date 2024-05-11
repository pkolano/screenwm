Screen Window Manager (Screenwm)
================================

1. Introduction

Screenwm is a window management extension to GNU Screen that adds (1)
multiple region layouts within a single session, (2) vi-like movement
within complex layouts, (3) window associations by region, and (4)
control of remote/embedded sessions using standard key bindings.

In Screen, a single terminal can be divided up into a set of regions
constructed from arbitrary horizontal and vertical splits.  Each region
displays one particular window at a time from the set of windows shared
across all regions.  The window in each region can be changed as
desired.  With the introduction of vertical splits, there are several
problems with this model.

First, the basic Screen model of a pool of windows shared among all
regions does not match typical usage patterns.  With the introduction of
cheap widescreen displays, users typically have many different terminals
side-by-side, each running a separate instance of Screen.  With vertical
splits, multiple terminals can be multiplexed in a single full-width
terminal, which provides many advantages such as movement/cut-paste
between windows without the use of the mouse and multiple remote windows
displayed simultaneously through a single SSH connection.  When this is
done, however, the natural separation provided by individual Screen
instances is lost and the user must now manage and cycle through the
complete set of windows.

Second, use of remote Screen sessions is a hassle as the Screen escape
key must be depressed twice to send a command to the remote session.
Alternatively, the user can define a different key binding that sends
two escape sequences, but then must remember to use the other binding
when working with the other session while using the normal binding the
rest of the time.

Finally, Screen can only work with one region layout at a time and
layouts must be manually managed by the user by combining then
resplitting the main terminal.  Thus, for example, if the user wishes to
switch between a set of remote side-by-side windows running in a remote
Screen session to a set of local side-by-side windows running in the
local session, they must manually split the terminal, then in each
region, must manually select the window that they wish to work with.
Quick transitions between the two layouts is impossible as this tedious
process must be done every time.

Enter Screenwm.  Screenwm solves all of these problems using an external
Perl executable and a set of Screen key bindings.  All basic Screen
operations are intercepted by key bindings and fed to Screenwm, which
keeps track of Screen's state in an external directory structure.
Screenwm then uses this information to derive an appropriate Screen
command sequence, which is fed back to the Screen session.  Using this
information, Screenwm can manage multiple region layouts simultaneously
with instant and automatic transitions between them.  It allows vi-like
movement between regions using a single keypress no matter how many
regions it is actually cycling through.  It provides windows associated
to regions instead of to a session, thus facilitating typical usage
patterns.  Finally, it allows a window to be tagged as a remote/embedded
Screen session, which allows the same key bindings used for local Screen
control to be passed on when appropriate to the remote session, thereby
allowing the user to transparently work across multiple sessions.

This project is similar to minimal X11 window managers such as
Ratpoison.  The advantage, however, is that it does not require an X11
session.  Thus, complex terminal window layouts can be utilized remotely
with just a single SSH connection, which is ideal for administration of
remote servers or accessing hosts at work/home from home/work.


2. Concepts

Screenwm is essentially a high-level controller for Screen that uses
low-level Screen primitives to provide enhanced capabilities.  The central
concept of Screenwm is a "layout", which is defined to be an arrangement of
regions.  In Screen, a region is created by splitting the main terminal
horizontally or vertically.  Screenwm supports multiple virtual layouts,
which are reconstructed dynamically within Screen's single layout whenever
the user moves between virtual layouts.  While layouts are now supported
natively in Screen, using the "only" primitive to allow a single region to
take over the whole terminal destroys the current layout.  In Screenwm, 
regions can be expanded and then the layout reconstructed at will.
Note that it has not been investigated if similar functionality could be
provided by Screen primitives such as autosave of layouts.

Screenwm is enabled by defining a set of Screen key bindings of the
form "bind <key> exec screenwm <command> <arguments>".  When the given
key is pressed, Screenwm is executed with the given command name and
arguments.  It then constructs an appropriate Screen command sequence to
perform the given action and sends that command back to the Screen
session using its "-X" option.

Screenwm currently supports the following standard Screen commands.
See the Screen man page for details of each.  Commands marked with an
asterisk (*) must be processed by Screenwm for correct operation.

    detach
  * down
    info
  * kill
  * next
    only
  * other
  * prev
  * quit
    redisplay
  * remove
  * screen
  * select <number>
  * split
  * up
  * vert_split (only w/ vertical split patch)

Screenwm currently supports the following unique commands:

    layout_kill - kill the current layout and move to the previously
                  selected layout

    layout_next - move to the next layout in numerical order

    layout_other - move to the previously selected layout

    layout_prev - move to the previous layout in numerical order

    layout_redisplay - reconstruct the current layout

    layout_screen - create a new layout

    layout_select <number> - move to layout <number>

    left - move to the region to the left of the current region
           (if multiple regions share a leftward edge, move the region
           to the left that has been visited most recently)

    right - move to the region to the right of the current region 
            (if multiple regions share a rightward edge, move the
            region to the right that has been visited most recently)

    window_meta - tag a window as a meta window (i.e. running another
                  remote/embedded instance of Screen)

    window_nometa - untag a meta window

Screenwm supports the following options:

    --meta - if the current window has been tagged as a meta window,
             pass the given command on to the remote/embedded Screen

    --meta=only - like --meta, but only send on the command if
                  the current window occupies its entire layout (i.e.
                  the layout consists of a single unsplit region)


3.  Usage

Screenwm starts in the default layout that has a single region with a
single window.  Each layout may be treated as a standard Screen session.
Thus, you may split regions, create windows, etc. as is normally done in
Screen.  You may move between different regions in the layout using
left (^ah), right (^al), up (^ak), and down (^aj).  You may move between
windows within the current region using the standard Screen commands
next (^an), prev (^ap), select (^a [0-9]), and other (^ao).  Note that
in Screenwm, however, each region has its own set of windows.  Thus,
next and prev cycle through the windows associated with the current
region instead of through all windows as is the behavior of Screen.
Similarly, select moves to the nth window in the region and other moves
to the previously selected window in the region.

To create a new layout, use layout_screen (^aC).  The new layout will
have a single region with a single window, which can, once again, be
split, etc.  You can move between layouts using layout_next (^aN),
layout_prev (^aP), layout_select (^a shift [0-9]), or layout_other
(^aO).  If more than one layout exists, you can kill a layout using
layout_kill (^aE).  Note that this action destroys all windows in the
layout without confirmation so any applications you have running in
those windows will be terminated.  You may kill regions within a layout
using the standard remove command (^aX), after which the layout will be
reconstructed as if that region never existed.  All windows in the
region will be destroyed.

If you are connecting to a remote/embedded Screen session within a
Screenwm window, you may tag that window as a meta window (^Am), which
will forward all commands specified as meta commands to that session.
By default, all commands will be forwarded to the remote/embedded
session except most layout commands and window_meta/nometa commands (see
the "screenwmrc" file for specifics).  Thus, you can use normal key
bindings to control the remote/embedded session.  In fact, by sending a
window_meta command to the remote/embedded session (^aam), you can
control arbitrarily nested Screen sessions using the exact key
bindings used by the local session.  Note that the remote/embedded
Screen session must be running the Screenwm extension for this
functionality to work properly.  By default, the detach command (^ad)
clears the window_meta setting, which can also be cleared explicitly
using window_nowmeta (^aM).


4.  Notes

Do not exit the shells running in windows directly (e.g. using ^d or
exit).  Always use the window kill command (^ae) instead.  Screenwm
cannot intercept these events, thus if you exit directly, its window
state will become corrupted.  The last window in a region cannot be
killed, thus you must either kill the region (^aX), kill the layout
(^aE), or exit Screen (^a^\).  If you exit a window shell by accident,
you can recover by displaying the current window list (^a") and deleting
the symlink of the number under the "name" field that does not exist in
the directory "~/.screenwm.$STY/current/current".

Screenwm commands are always invoked with an exec command, thus only
operate correctly in windows with shells.  For this reason, Screenwm
will not operate properly in the blank window (^a-), nor with the zombie
command.  If you find yourself in the blank window, display the current
window list (^a"), move to a window with a shell, then reconstruct the
layout (^aR).

Certain aspects of copy mode do not operate correctly in remote/embedded
sessions that have been split vertically.  This is due to the fact that
the local Screen does not know about splits that have occurred within
the remote/embedded session.  Thus, for example, the yank commands (y
and Y) will yank the entire line across all the remote/embedded splits.
The workaround is to use the margin commands (c and C) to set the
margins or you can temporarily join all remote/embedded regions (^aQ),
copy the text, then use layout_redisplay (^AR) to reconstruct the
regions.

When you reattach a remote/embedded screen sessions (e.g. using
"screen -r"), the remote/embedded layout will not be displayed
correctly, thus you must perform a layout_redisplay in that session.
You will probably also wish to retag the window as a meta window, thus
the typical sequence after a reattach is ^am^aR.

Multiple instances of Screenwm may be run on the same host.  All
information about the current session is stored in the directory
"~/.screenwm.$STY", where $STY is an environment variable set by Screen.
This directory is destroyed only when exiting Screen normally (^a^\).
If you kill Screen through other means, you may have leftover instances
of these directories.

Screenwm does not currently operate correctly with region resizing.
When moving between layouts, regions will revert to their default size.
Also, movement between regions will not be accurate if regions are
resized such that regions that shared an edge before resizing do not
share an edge after resizing and vice versa.  Support for resizing may
be added in the future.

See the "INSTALL" document for additional notes.


Questions, comments, fixes, and/or enhancements welcome.

--Paul Kolano <pkolano@gmail.com>

