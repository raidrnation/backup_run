# $language = "Python"
# $interface = "1.0"

import re
import datetime
import os
import platform
import shutil
import sys
import time
import subprocess

host = (
        "192.168.1.100",
        "192.168.1.101",
        "192.168.1.102",
        "192.168.1.103"
)

user = 'test'
passwd = crt.Dialog.Prompt("Enter password for", "Login", "", True)

MsgBox = crt.Dialog.MessageBox
Prompt = crt.Dialog.Prompt

# ========== Begin Globals ==========
g_bDebug = True
g_strConfigToSave = "running"
g_strAdditionalArgs = ""
strHome = os.path.expanduser("~")
g_strMyDocs = strHome.replace("\\", "/") + "/Documents"
g_strDateTimeTag = datetime.datetime.now().strftime("%Y%m%d_%H%M%S.%f")[:19]
g_strOperationalLogFile = ""
objTab = crt.GetScriptTab()
objTab.Screen.Synchronous = False
g_strSessionName = ""
g_strSessionPath = ""

# Collection of global args that can be supplied as script
# arguments, and their default values:
#---------------------------------------------
g_cScriptArgs = {
    "asa-uses-more":    False,
    "auto-close-app":   True,
    "confirm-filename": True,
    "interactive":      True,
    "use-sessmgr-tree": True
}
#---------------------------------------------


# Other global args that affect script behavior
#---------------------------------------------
g_bUseSessMgrTree = True
g_bInteractive = True
g_bConfirmFilename = True
g_bASAUsesMore = False
g_bAutoCloseApp = False
# NOTE: g_bAutoCloseApp Only applies to non-interactive
#       instances of this script. If you're testing
#       modifications to this script, you may want to set
#       this to false (or use the /auto-close-app:off script
#       argument) until you've worked out all the issues
#       with your modifications so that the SecureCRT
#       application does not automatically close each time
#       the script finishes:
#---------------------------------------------

# ========== End Globals ==========



# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
def Main():
    GetCurrentSessionNameAndPath()

    global g_bDebug
    global g_strConfigToSave
    global g_strAdditionalArgs
    global g_strMyDocs
    global g_strDateTimeTag
    global g_strOperationalLogFile
    global objTab
    global g_strSessionName
    global g_strSessionPath
    global g_cScriptArgs
    global g_bUseSessMgrTree
    global g_bInteractive
    global g_bConfirmFilename
    global g_bASAUsesMore
    global g_bAutoCloseApp

    g_strOperationalLogFile = (
## Directory named ~/Documents 
        g_strMyDocs +
## Name of logfile written to Documents directory
        "/Cisco-SaveDeviceConfigToFile({}-{})_log.txt".format(
        g_strSessionName, g_strDateTimeTag))

## First line of logfile 
    LogLine("################### Script starting ####################")
    LogLine("Session Name: {}".format(g_strSessionName))
    if g_strSessionName <> g_strSessionPath:
        LogLine("Session Path: {}".format(g_strSessionPath))

## Reads args for CRT button
    if crt.Arguments.Count > 0:
        strArg = str(crt.Arguments.GetArg(0)).lower()
    else:
        strArg = g_strConfigToSave

    g_strConfigToSave = strArg

    if crt.Arguments.Count > 1:
        # Look for additional args to add to the end of the specified command,
        # as well as any additional customization flags such as:
        #     /asa-uses-more:yes|on|true|1|no|off|false|0
        #     /interactive:yes|on|true|1|no|off|false|0
        #     /auto-close-app:yes|on|true|1|no|off|false|0
        #     /confirm-filename:yes|on|true|1|no|off|false|0
        #     /use-sessmgr-tree:yes|on|true|1|no|off|false|0
        for nArgIndex in range(1, crt.Arguments.Count):
            strAddArg = str(crt.Arguments.GetArg(nArgIndex))
            if not ProcessCommandLineArg(strAddArg):
                if g_strAdditionalArgs == "":
                    g_strAdditionalArgs = strAddArg
                else:
                    g_strAdditionalArgs = "{0} {1}".format(g_strAdditionalArgs, strAddArg)

    g_bASAUsesMore = g_cScriptArgs["asa-uses-more"]
    g_bAutoCloseApp = g_cScriptArgs["auto-close-app"]
    g_bConfirmFilename = g_cScriptArgs["confirm-filename"]
    g_bInteractive = g_cScriptArgs["interactive"]
    g_bUseSessMgrTree = g_cScriptArgs["use-sessmgr-tree"]

    if not (g_strConfigToSave == "running" or g_strConfigToSave == "startup"):
        strMsg = ('' +
            "Unrecognized config type: '" + g_strConfigToSave + "'\r\n" +
            "Expected either 'running' or 'startup'.\r\n\r\n" +
            "Exiting.")
        if g_bInteractive:
            MsgBox(strMsg)
        else:
            LogLine(strMsg)
        ExitScript()
        return


    if not objTab.Session.Connected:
        strMessage = "You must first be connected to run this script!"
        #FlashStatusText(strMessage)
        if g_bInteractive:
            MsgBox(strMessage)
        else:
            LogLine(strMessage)
        ExitScript()
        return

## Detect the shell prompt by getting all the text to the left of the cursor.
    objTab.Screen.Synchronous = True
    objTab.Screen.Send("\r")
## Continuing logfile 
    DisplayStatus("Detecting CLI shell prompt text...")
    while objTab.Screen.WaitForCursor(1):
        crt.Sleep(1)
    strPrompt = GetTextLeftOfCursor()
    if len(strPrompt) < 1:
        objTab.Screen.Send("\r")
        DisplayStatus("Waiting for prompt...")
        while objTab.Screen.WaitForCursor(1):
            crt.Sleep(1)

        strPrompt = GetTextLeftOfCursor()
        if len(strPrompt) < 1:
            strMsg = "No prompt detected! Press/Send Enter first."
            FlashStatusText(strMsg)
            if not g_bInteractive:
                ExitScript()
            return
    DisplayStatus("CLI shell prompt is: '{}'".format(strPrompt))

    # Attempt to check if we're in privileged/enable mode:
    DisplayStatus("Checking to ensure the device is in privileged/enable mode...")
    if strPrompt.rstrip()[-1] != '#':
        strMsg = "Must  'enable'  first!"
        FlashStatusText(strMsg)
        if not g_bInteractive:
            ExitScript()
        return
    DisplayStatus("  --> OK.")

    # Attempt to ensure we're not in (config) mode - bad place to
    # be in if running this script, right?
    DisplayStatus("Checking to ensure the device is *not* in (config) mode...")
## Var for getting hostname
    strHostname = GetHostname()
    if "config)" in strPrompt:
        strMsg = "Not in priv EXEC mode... exiting."
        FlashStatusText(strMsg)
        if not g_bInteractive:
            ExitScript()
        return
    DisplayStatus("  --> OK.")

    bIsASADevice = False
    strPagerOffCommand = "term len 0"
    strTermWidthCommand = "term width <columns>"
## Command show running|startup config
    strShowConfigCommand = "show " + g_strConfigToSave + "-config"

    nColsOrig = objTab.Session.Config.GetOption("Cols")
    DisplayStatus("Current session columns: {}".format(nColsOrig))

    DisplayStatus("Getting device's term info...")
    # Get Term size info
    objTab.Screen.Send("sh term\r")
    objTab.Screen.WaitForString("sh term")
    vWaitFors = [strPrompt,
            "Width: ",
            "Length: ",
            "Width = ",
            "TTY:",
            "--More--", "-- More --", "--more--", "-- more --"]
    while True:
        objTab.Screen.WaitForStrings(vWaitFors)
        DisplayStatus("Found a vWaitFor entry at index #{}: {}".format(
            len(vWaitFors), vWaitFors[objTab.Screen.MatchIndex]))
        if objTab.Screen.MatchIndex == 1:
            # Found strPrompt. We're done looping
            break

        elif objTab.Screen.MatchIndex == 2:
            # Found "Width: "
            nRemoteCols = objTab.Screen.ReadString(" columns")
            LogLine("nRemoteCols: {}".format(nRemoteCols))

        elif objTab.Screen.MatchIndex == 3:
            # Found "Length: "
            nRemoteRows = objTab.Screen.ReadString(" lines")
            LogLine("nRemoteRows: {}".format(nRemoteRows))

        elif objTab.Screen.MatchIndex == 4:
            # Found "Width = ". This means we're on an ASA device
            bIsASADevice = True
            # Read the number of cols up to the ',' character
            nRemoteCols = objTab.Screen.ReadString(",")
            LogLine("Is ASA Device: True\r\nnRemoteCols: {}".format(nRemoteCols))
            strPagerOffCommand = "term pager 0"
            if g_strConfigToSave == "running" and g_bASAUsesMore:
                strShowConfigCommand = "more system:{0}-config".format(g_strConfigToSave)
                LogLine("ASA: Using 'more system:{}-config' as command since /asa-uses-more option was enabled.".format(
                    g_strConfigToSave))
            else:
                LogLine("ASA: Using standard ""show {}-config"" command since /asa-uses-more option was NOT enabled.".format(
                    g_strConfigToSave))

        elif objTab.Screen.MatchIndex == 5:
            # Found "TTY:" which appears on nexus devices
            LogLine("Nexus device detected via 'TTY:' output")

        elif objTab.Screen.MatchIndex > 5:
            # This means that anything *but* the first 4
            # elements of vWaitFors was seen, so we need
            # to press SPACE to continue receiving output.
            objTab.Screen.Send(" ")
            continue

    # Turn off paging so that we don't have to deal with "more" prompts
    DisplayStatus("Turning off paging...")
    objTab.Screen.Send("{}\r".format(strPagerOffCommand))
    objTab.Screen.WaitForString(strPrompt)

    # ASA devices don't seem to allow setting term width outside of
    # global config mode; enter that mode now so we can set the width.
    if bIsASADevice:
        DisplayStatus("Entering global config mode for ASA device...")
        objTab.Screen.Send("conf term\r")
        objTab.Screen.WaitForString("(config)#")

    # Set the terminal width to avoid long lines wrapping.
    if int(nRemoteCols) < 132:
        DisplayStatus("Remote columns < 132; increasing to avoid lines wrapping.")
        if nColsOrig < 132:
            DisplayStatus("Adjusting cols to 132...")
            objTab.Session.Config.SetOption("Cols", 132)
            objTab.Screen.Send("term width 132\r")
        else:
            DisplayStatus("Adjusting cols to {}...".format(
                nColsOrig))
            objTab.Screen.Send("term width {}\r".format(
                nColsOrig))

        DisplayStatus("Waiting for term width command to complete...")
        if bIsASADevice:
            objTab.Screen.WaitForString("(config)#")
        else:
            objTab.Screen.WaitForString(strPrompt)

    # ASA devices don't seem to allow setting term width outside of
    # global config mode; exit that mode now that we've set the width.
    if bIsASADevice:
        DisplayStatus("Leaving global config mode for ASA device...")
        objTab.Screen.Send("end\r")
        objTab.Screen.WaitForString(strPrompt)

    # Accommodate any additional args to the sh startup-config or sh running-config
    # that were passed in as arguments to the script (as in: the button bar setup
    # has additional arguments after the "running"). For example:
    #    running all
    #    running brief
    #    running interface dialer 1
    #    startup linenum)
    strCmd = strShowConfigCommand
    if g_strAdditionalArgs == "":
        strCmd += "\r"
    else:
        DisplayStatus("Appending additional args to our show command...")
        strCmd += " " + g_strAdditionalArgs + "\r"

    # Now that we've aggregated any additional cmd args, let's send the command...
    DisplayStatus("Sending command: {}".format(strCmd.strip()))
    objTab.Screen.Send(strCmd)
    DisplayStatus("Waiting for the command to be received...")
    objTab.Screen.WaitForString(strCmd)
    DisplayStatus("Waiting for output to begin...")
    objTab.Screen.WaitForString("\n")

    # Now, Let's read/capture up to the point where the shell prompt appears...
    DisplayStatus("Reading " + g_strConfigToSave + "-config...")
    strConfig = objTab.Screen.ReadString(strPrompt)

    # Convert screen data EOLs to standard "\n". When the data gets
    # written out to the file, it will be converted as needed by
    # the underlying python write() calls because we're not opening
    # the file for write as binary.
    strConfig = strConfig.replace("\r\n", "\n")

    # Prepare to save the received data to a file...
    DisplayStatus("Saving {} bytes of captured config data to local file system...".format(
        len(strConfig.replace("\n", os.linesep))))

    DisplayStatus("  --> Date Time tag: {}".format(g_strDateTimeTag))
    DisplayStatus("  --> Session Path: {}".format(g_strSessionPath))
    strSessionName = g_strSessionName
    strSavedConfigsFolder = "Config-Saves/"
    if g_bUseSessMgrTree:
        strSavedConfigsFolder += g_strSessionPath

## Make sure that the relative config folder where configs are
## to be saved, hat it ends with a trailing "/" char:
    if strSavedConfigsFolder[-1] != "/":
        strSavedConfigsFolder += "/"

    DisplayStatus("  --> Saved Configs Folder Name: {}".format(strSavedConfigsFolder))

## Construct actual folder where configs will be saved.
    strActualFolder = g_strMyDocs + "/" + strSavedConfigsFolder

    DisplayStatus("  --> Saved Configs Path: {}".format(strActualFolder))

    # Check to make sure that the destination folder exists...
    if not os.path.exists(strActualFolder):
        # create folder tree for config file we're about to save.
        try:
            DisplayStatus("Attempting to create folder: '{}'...".format(strActualFolder))
            os.makedirs(strActualFolder)
            DisplayStatus("  --> Successfully created folder: '{}'".format(strActualFolder))
        except Exception as objException:
            strMsg = "Unable to create destination folder for saved configs: \r\n{}".format(str(objException))
            DisplayStatus("  --> {}\r\n\r\nExiting.".format(strMsg))
            if g_bInteractive:
                MsgBox(strMsg)
            else:
                FlashStatusText(strMsg)
                ExitScript()
            return

    # If actual folder variable was hard-coded or otherwise modified,
    # make sure it has a trailing "/" charcter:
    if strActualFolder[-1] != "/":
        strActualFolder += "/"

    # Build up the save-save file path, including the session name,
    # remote device address, type (start vs. running) and the date
    # and time tag.
    strActualFilePath = strActualFolder + strSessionName + "_" + \
        objTab.Session.RemoteAddress + "_" + g_strConfigToSave + \
            "-config_" + g_strDateTimeTag + ".txt"
    DisplayStatus("Destination file name: {}".format(strActualFilePath))

    # Prompt (if needed) for the file name/path to be confirmed:
    strFilename = strActualFilePath
    if g_bConfirmFilename and g_bInteractive:
        DisplayStatus("Prompting for filename (default: {})".format(strFilename))
        strFilename = Browse(
            "Choose where to save your " + g_strConfigToSave + "-config",
            "Save",
                strFilename,
            "Text Files (*.txt)|*.txt||")
        DisplayStatus("Filename returned by user: {}".format(strFilename))
        if strFilename == "":
            FlashStatusText("Script cancelled")
            return
    else:
        DisplayStatus("Not prompting for filename (either running interactively, or /confirm-filename option val=off).")

    # Write the config output data to the local file:
    DisplayStatus("Opening destination file...")
    try:
        with open(strFilename, "w") as objFile:
            DisplayStatus("Writing config to file...")
            objFile.write(strConfig)
    except Exception as objException:
        strMsg = "Error opening or writing to the config file: {}".format(
            str(objException))
        DisplayStatus("  --> {}\r\n\r\nExiting.".format(strMsg))
        if g_bInteractive:
            MsgBox(strMsg)
        else:
            FlashStatusText(strMsg)
            ExitScript()
        return
    #objTab.Screen.Send("exit\r")
    # If on Windows platform, and running interactively,
    # bring up explorer with the file selected...
    DisplayStatus("Script Completed")
##    if g_bInteractive:
##        if sys.platform == "win32" and g_bInteractive:
##            strCmd = "explorer /e,/select,\"" + strFilename.replace("/", "\\") + "\""
##            subprocess.call(strCmd)
##        return
##    else:
##        if "SSH" in objTab.Session.Config.GetOption("Protocol Name"):
##            objTab.Session.Config.SetOption("Force Close On Exit", True)
##        ExitScript()
    crt.Session.Disconnect()

# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
def DisplayStatus(strStatus):
    global objTab
    objTab.Session.SetStatusText(strStatus)
    LogLine(strStatus)

# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
def GetHostname():
    strPrompt = GetTextLeftOfCursor()
    objMatch = re.match(r'^([0-9a-zA-Z\_\-\.]+)', strPrompt)
    if objMatch:
        return objMatch.group(1)
    else:
        FlashStatusText("No match on hostname pattern!")
        return

# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
def GetTextLeftOfCursor():
    global objTab
    nRow = objTab.Screen.CurrentRow
    nCol = objTab.Screen.CurrentColumn - 1
    strTextLeftOfCursor = objTab.Screen.Get(nRow, 1, nRow, nCol)
    return strTextLeftOfCursor


# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
def FlashStatusText(strMsg):
    LogLine("Flashing Status Text: {}".format(strMsg))
    global objTab
    nShortPause = 200
    nLongPause = 400
    for i in range(1,5):
        objTab.Session.SetStatusText(strMsg)
        crt.Sleep(nLongPause)
        objTab.Session.SetStatusText("")
        crt.Sleep(nShortPause)

    objTab.Session.SetStatusText(strMsg)

#~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
def Browse(strMessage, strButtonCaption, strDefaultPath, strFilter):
    strPlatform = sys.platform
    # Windows version of SecureCRT allows FileOpenDialog to return
    # a path to a file that doesn't yet exist... But Linux/Mac versions
    # of FileOpenDialog() require an existing file. So, use the nicer
    # interface in Windows, and on Linux/Mac, simply present an input
    # box that will allow for path to be changed, if desired. If you
    # are on Linux/Mac, and don't like the default path, simply change
    # the value of the g_strMyDocs variable (globally defined) as well
    # the value of the strSavedConfigsFolder variable defined
    # in Main() above.

    if strPlatform == "win32":
        return crt.Dialog.FileOpenDialog(
            strMessage,
            strButtonCaption,
            strDefaultPath,
            strFilter)
    else:
        return crt.Dialog.Prompt(
            strMessage,
            strButtonCaption,
            strDefaultPath)

# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
def LogLine(strMsg):
    global g_strOperationalLogFile
    global g_bDebug

    if not g_bDebug:
        return

    # make sure that all path separators are forward "/" instead of "\"
    g_strOperationalLogFile = g_strOperationalLogFile.replace("\\", "/")

    # Check to make sure that the parent folder for the destination log file
    # exists...
    strLogFolder = os.path.dirname(g_strOperationalLogFile)
    if not os.path.exists(strLogFolder):
        # create folder tree for config file we're about to save.
        try:
            os.makedirs(strLogFolder)
        except Exception as objException:
            strMsg = "Unable to create destination folder [{}] for debug logs: \r\n\r\n{}".format(
                strLogFolder, str(objException))
            crt.Dialog.MessageBox(strMsg)
            sys.exit()

    strDateTimeTag = datetime.datetime.now().strftime("%Y%m%d_%H%M%S.%f")[:19]

    try:
        with open(g_strOperationalLogFile, "a") as objFile:
            objFile.write("{0}: {1}\n".format(strDateTimeTag, strMsg))
    except Exception as objException:
        strMsg = (
            "Failed to create or write to debug log file " +
            "[{}]:\r\n\r\n{}\r\n".format(g_strOperationalLogFile, str(objException)) +
            "\r\n\r\n" +
            "  --> Does the parent folder exist?\r\n" +
            "\r\n" +
            "  --> Do you have adequeate permissions?\r\n" +
            "\r\n\r\n" +
            "Please correct the g_strOperationalLogFile " +
            "to reflect a folder that exists and where " +
            "you have adequate permissions to create and " +
            "write to files.")
        crt.Dialog.MessageBox(strMsg)
        sys.exit()

# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
def ExitScript():
    global g_bAutoCloseApp
    global objTab

    if g_bAutoCloseApp:
        if crt.GetTabCount() > 1:
            objTab.Session.Disconnect()
            objTab.Activate()
            objTab.Screen.SendSpecial("MENU_TAB_CLOSE")
        else:
            objTab.Session.Disconnect()
            objTab.Activate()
            objTab.Screen.SendSpecial("MENU_TAB_CLOSE")
            crt.Quit()

# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
def ProcessCommandLineArg(strCLIArg):
    global g_cScriptArgs
    LogLine("Processing Script argument: {}".format(strCLIArg))
    strArgRaw = strCLIArg.lower()
    objMatch = re.match("/(\S+)\:(\S+)", strArgRaw)
    if objMatch:
        strOptionName = objMatch.group(1)
        strOptionValue = objMatch.group(2)
        if not strOptionName in g_cScriptArgs:
            LogLine("  ##### ERROR: Unsupported value ({}) supplied for arg: {}".format(
                strOptionValue, strOptionName))
            return False

        if re.match("^(?:no|false|0|off)$", strOptionValue):
            # Set the value for this opt in the g_cScriptArgs
            # collection to False
            g_cScriptArgs[strOptionName] = False
        elif re.match("^(?:yes|true|1|on)$", strOptionValue):
            # Set the value for this opt in the g_cScriptArgs
            # collection to True
            g_cScriptArgs[strOptionName] = True
        else:
            LogLine("  ##### ERROR: Unsupported value for option '{}': {}".format(
                strOptionName, strOptionValue))
            LogLine("               Expected: yes|no|true|false|1|0|on|off")
            return False

        DisplayStatus("  Script arg '{}' specified with value: {}".format(
            strOptionName, strOptionValue))
        return True
    else:
        LogLine("  --> Unprocessed script arg: {}".format(
            strArgRaw))
        return False

# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
def GetCurrentSessionNameAndPath():
    global g_strSessionName, g_strSessionPath, objTab
    g_strSessionPath = objTab.Session.Path
    strSessionName = g_strSessionPath
    g_strSessionName = strSessionName

    # Determine if the current session path is in a sub-folder within
    # the session manager so that we can extract just the session name
    # at the end of the session path...
    g_strSessionPath = g_strSessionPath.replace("\\", "/")
    objMatch = re.match(r'(.*)[/]([^/]+)$', g_strSessionPath)
    if objMatch:
        g_strSessionPath = objMatch.group(1)
        g_strSessionName = objMatch.group(2)

# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

for i in range(0, len(host)):
    cmd = "/SSH2 /L %s /PASSWORD %s /C 3DES /M SHA1 %s" % (user, passwd, host[i])
    crt.Session.Connect(cmd)
    Main()
