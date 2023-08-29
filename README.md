# Klipper-Macros

Skip to main content
Klipper

Log In


Build Sheet Manager & “Adjust Live Z”
Macros

garethky

3
Sep '22
This is a small group of macros that implements Prusa’s Build Sheet system and Adjust Live Z behavior in klipper. Specifically:

You can have as many build sheets as you like, each with a unique human readable name
Each sheet has its own Z offset
A sheet can be installed in the printer and this persists across restarts.
When you “Adjust Live Z” (AKA Baby Stepping) with your klipper UI of choice the offset change is saved instantly to the sheet that is installed.
Anything thats not a true “Adjust Live Z” call wont change the sheet’s z-offset value. You need to set MOVE=1 and ADJUST_Z= in your call to SET_GCODE_OFFSET. This is how Fluidd, Mainsail & Klipper Screen etc do it. All other call patterns wont change the sheet offset.
Requirements:

You have [save_variables] 59 set up
You need to call the APPLY_BUILD_SHEET_ADJUSTMENT in your PRINT_START macro after you calibrate your Z offset. This is when the sheet offset is applied.
Last but most important: Your PRINT_START macro must set your Z Offset. You can do this with a plugin like CALIBRATE_Z 63. Or, you can add a line to your PRINT_START to reset the Z offset to the correct value for your printer: SET_GCODE_OFFSET Z=0.0. This macro doesn’t try to keep track of the state of the z offset, that’s on you. If you don’t do this the APPLY_BUILD_SHEET_ADJUSTMENT call will apply the same adjustment multiple times and you will crash your printer.
Nice To Have:

You’ll probably want to make some small macros for each build sheet you have to make swapping them easy. E.g. :
[gcode_macro INSTALL_TEXTURED_SHEET]
gcode:
    INSTALL_BUILD_SHEET NAME="Textured PEI Sheet"
In your PRINT_START you can add a call to SHOW_BUILD_SHEET at the start and end. This prints the build sheet name and offset to the console. This can be helpful and remind you if you forgot to swap sheets!
Macros:

## Macro to install a build sheet
[gcode_macro INSTALL_BUILD_SHEET]
description: Install a build sheet and save settings
variable_parameter_NAME: "Unknown Build Sheet"
variable_parameter_OFFSET: 0.0
gcode:
    {% set svv = printer.save_variables.variables %}
    {% set sheet_name = (params.NAME | default("Unknown Build Sheet") | string) %}    
    {% set sheet_offset = (params.OFFSET | default(0.0) | float) %}
    {% set sheet_key = ("build_sheet " ~ (sheet_name | lower | replace(" ", "_"))) %}
    {% set sheet = (svv[sheet_key] | default(None)) %}
    {% if not sheet %}
        # the sheet does not exist, create it
        {% set sheet_literal = ('{"name": "%s", "offset": %.3f}' % (sheet_name, sheet_offset)) %}
        SAVE_VARIABLE VARIABLE="{sheet_key}" VALUE='{sheet_literal}'
    {% endif %}
    SAVE_VARIABLE VARIABLE="build_sheet installed" VALUE="'{sheet_key}'"
    SHOW_BUILD_SHEET

[gcode_macro SHOW_BUILD_SHEET]
gcode:
    {% set svv = printer.save_variables.variables %}
    {% set sheet_key = (svv["build_sheet installed"] | string) %}
    {% set sheet = (svv[sheet_key] | default(None)) %}
    {% if sheet %}
        {action_respond_info("Build sheet: %s, Offset: %.3fmm" % (sheet.name, sheet.offset))}
    {% else %}
        {action_respond_info("No build sheet installed") }
    {% endif %}

[gcode_macro SET_BUILD_SHEET_OFFSET]
description: Set an arbitrary offset for the build sheet
variable_parameter_OFFSET: 0.0
gcode:
    {% set svv = printer.save_variables.variables %}
    {% set sheet_offset = (params.OFFSET | default(0.0) | float) %}
    {% set sheet_key = (svv["build_sheet installed"] | string) %}
    {% set sheet = (svv[sheet_key] | default(None)) %}
    {% if sheet %}
        {% set sheet_literal = ('{"name": "%s", "offset": %.3f}' % (sheet.name, sheet_offset)) %}
        SAVE_VARIABLE VARIABLE="{sheet_key}" VALUE='{sheet_literal}'
    {% endif %}
    SHOW_BUILD_SHEET

[gcode_macro RESET_BUILD_SHEET_OFFSET]
description: Set the build sheet offset to 0.0
gcode:
    SET_BUILD_SHEET_OFFSET OFFSET=0.0

[gcode_macro SET_GCODE_OFFSET]
rename_existing: _BUILD_SHEETS__SET_GCODE_OFFSET
gcode:
    # this runs first so if the caller messed up the params it failes
    _BUILD_SHEETS__SET_GCODE_OFFSET {% for p in params
            %}{'%s=%s ' % (p, params[p])}{%
           endfor %}

    {% set svv = printer.save_variables.variables %}
    {% set sheet_key = (svv["build_sheet installed"] | string) %}
    {% set sheet = (svv[sheet_key] | default(None)) %}
    {% if sheet and (params.Z_ADJUST is defined) and ((params.Z_ADJUST | float) != 0.0) and ((params.MOVE | int) == 1) %}
        {% set offset = (sheet.offset + (params.Z_ADJUST | default(0.0) | float)) %}
        {% set sheet_literal = ('{"name": "%s", "offset": %.3f}' % (sheet.name, offset)) %}
        SAVE_VARIABLE VARIABLE="{sheet_key}" VALUE='{sheet_literal}'
        SHOW_BUILD_SHEET
    {% endif %}

[gcode_macro APPLY_BUILD_SHEET_ADJUSTMENT]
description: Applies the sheet offset adjustment for the installed build sheet, call from your PRINT_START macro
gcode:
    # get the offset from saved variables
    {% set svv = printer.save_variables.variables %}
    {% set sheet_key = (svv["build_sheet installed"] | string) %}
    {% set sheet = (svv[sheet_key] | default(None)) %}
    {% if sheet %}
        # log this to the console so if there is an issue with the print the user can see which sheet the printer thought it had installed
        {action_respond_info("Applying offset: %.3fmm for build sheet: %s" % (sheet.offset, sheet.name))}
        _BUILD_SHEETS__SET_GCODE_OFFSET Z_ADJUST={sheet.offset} MOVE=1
    {% else %}
        {action_respond_info("No build sheet installed, offset adjustment: 0.000mm") }
    {% endif %}

My file on GitHub: klipper-voron2.4-config/build-sheets.cfg at mainline · garethky/klipper-voron2.4-config · GitHub 33

Enjoy, let me know how its working for you, your suggestions and all my terrible spelling mistakes. :beers:

2


created
Sep '22
last reply
Sep '22
1
reply

garethky
Sep '22
You can add a menu to KlipperScreen to change build sheets like this:

[menu __main actions build_sheets]
name: Build Sheets
icon: bed-level

[menu __main actions build_sheets smooth_pei]
name: Smooth PEI
method: printer.gcode.script
params: {"script":"INSTALL_SMOOTH_PEI_SHEET"}

[menu __main actions build_sheets textured_pei]
name: Textured PEI
method: printer.gcode.script
params: {"script":"INSTALL_TEXTURED_PEI_SHEET"}

[menu __main actions build_sheets smoth_garolite]
name: Smooth Garolite
method: printer.gcode.script
params: {"script":"INSTALL_TEXTURED_GAROLITE_SHEET"}

[menu __main actions build_sheets textured_garolite]
name: Textured Garolite
method: printer.gcode.script
params: {"script":"INSTALL_TEXTURED_GAROLITE_SHEET"}


Hello! Looks like you’re enjoying the discussion, but you haven’t signed up for an account yet.
Tired of scrolling through the same posts? When you create an account you’ll always come back to where you left off. With an account you can also be notified of new replies, save bookmarks, and use likes to thank others. We can all work together to make this community great. heart


Sign Up
Maybe later
no thanks

New & Unread Topics

Start_print macro not extruding on primer lines
4
Macros
Sep '22

Query printer status information
0
Macros
Sep '22

Door Switch – Variable help needed
10
Macros
Nov '22

Extend RESPOND command with some color options
0
Macros
Sep '22

Animated screensaver support
3
Macros
May 2
Want to read more? Browse other topics in 
Macros
 or view latest topics.

