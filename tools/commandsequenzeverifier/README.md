# Command Verification Tool

A browser-based utility to test and validate sequences of hexadecimal commands on a Bluetooth-enabled hardware pedal. This tool ensures that a list of generated commands is correct, sent in the right order, and properly acknowledged by the device , providing a real-time log for debugging and confirmation.

-----

## Overview

The Command Verification Tool provides a simple interface to connect to a Bluetooth device and execute a series of commands sequentially. It listens for a specific success acknowledgment (ACK) from the device after each command is sent and reports the success or failure of each step. This is essential for verifying that cracked or reverse-engineered commands behave as expected.

-----

## Key Features

  - **Web Bluetooth Connectivity**: Directly connects to the target hardware pedal via the browser using the Web Bluetooth API.
  - **Command Sequence Execution**: Sends a user-provided list of commands one by one , with a configurable delay between each transmission.
  - **Real-Time Logging**: Displays a timestamped log of all actions , including sent commands (`SENT`) , received data (`RECV`) , and status messages , with color-coding for clarity.
  - **ACK Verification**: Listens for a specific success `ACK` code from the device after sending each command  and reports whether it was received within the timeout period.
  - **Structured Command Parsing**: Parses commands formatted as key-value pairs, such as `'EQ lowhz -50': '8080f0...f7'`.
  - **User-Friendly Interface**: Provides a clear status indicator for the Bluetooth connection , a progress bar for the verification sequence , and simple controls to start and stop the process.

-----

## How to Use

1.  **Paste Commands**: Paste your commands into the **"Commands to Verify"** text area.
2.  **Connect to Pedal**: Click the **"1. Connect to Pedal"** button. A browser pop-up will appear to search for and select your Bluetooth device. The status indicator will turn green upon a successful connection.
3.  **Configure Delay**: Adjust the **Delay (ms)** value to set the waiting time between sending commands.
4.  **Start Verification**: Click the **"2. Start Verification"** button to begin sending the command sequence to the device.
5.  **Monitor Log**: Observe the **"Verification Log"** for real-time feedback on each command's success or failure.
6.  **Stop**: Click the **"Stop Verification"** button at any time to halt the sequence.

-----

## How It Works 

### Bluetooth Connection 

The tool uses the Web Bluetooth API to scan for devices advertising a specific `SERVICE_UUID` (`03b80e5a-ede8-4b33-a751-6ce34ec4c700`). Once a device is selected, it connects and retrieves the necessary service and characteristic (`7772e5db-3868-4112-a1a9-f2669d106bf3`). It then subscribes to notifications on the characteristic to listen for incoming data from the pedal, such as acknowledgment codes.

### Command Parsing

When "Start Verification" is clicked, the tool reads the content of the text area and uses a regular expression to find all valid command structures.

```regex
/'([^']+)':\s*'([a-fA-F0-9]{50,})'/g
```

This expression specifically looks for the `name`: `command` format, extracting both the human-readable name for logging and the hex command to be sent.

### Verification Loop & ACK Handling

The tool iterates through the parsed commands sequentially in a verification loop.
For each command, the process is as follows:

1.  The command is sent to the pedal using the `writeValueWithoutResponse` Bluetooth function.
2.  The tool immediately begins listening for a specific `SUCCESS_ACK_CODE` (`8080f00b02000100000003010400080000f7`) from the device.
3.  A timeout (`ACK_TIMEOUT`) of 400ms is initiated. If the correct ACK is received before the timeout, the command is logged as a **SUCCESS**. If the timeout occurs first, it's logged as a **FAILED**.
4.  The tool then pauses for the user-defined delay before processing the next command in the sequence.
