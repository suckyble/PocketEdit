## Cracking the Command Checksum: A Brute-Force Approach 

To communicate with the pedal, every command sent must be validated with a correct 2-byte checksum (CRC). Since the algorithm for generating this checksum was unknown, a brute-force method was developed to discover the correct checksum for every required command.

This approach was made possible by two key factors:

1.  A reliable **success acknowledgement** and **nonsuccess acknowledgement** message sent back from the pedal upon receiving a valid command.
2.  A surprisingly small **search space** for the checksum value.

-----

### The Methodology

The cracking process was automated using a custom HTML/JavaScript tool that leverages the **Web Bluetooth API** to communicate directly with the pedal from a browser.

The core logic of the brute-force script is as follows:

1.  **Isolate the Payload**: For any given command (e.g., changing the FX module order), the static header (`8080f0`) and footer (`f7`) are known. The 14-byte payload that defines the specific action is also known. The only missing part is the 2-byte CRC that sits between the header and the payload.

    > `[HEADER]` `[UNKNOWN CRC]` `[PAYLOAD]` `[FOOTER]`
    > `8080f0` `XXXX` `000100...` `f7`

2.  **Define the Search Space**: A critical discovery was that the 2-byte (4-hex-character) checksum doesn't use all 65,536 possible values. Instead, it follows a specific `0X0Y` pattern (e.g., `0406`, `0D0B`). This reduces the search space from `FFFF` (65,536) possibilities to just `0F0F` (**256** possibilities), making a brute-force attack extremely fast.

3.  **Iterate and Test**: The script programmatically iterates through all 256 possible checksums from `0000` to `0F0F`. In each iteration, it:

      * Constructs a full 20-byte command with the candidate checksum.
      * Sends the command to the pedal via Web Bluetooth.
      * Listens for a response.

4.  **Listen for the "Golden" ACK**: After a command is sent, the script waits for a short timeout period (e.g., 150ms).

      * **If the pedal sends back the specific success acknowledgement** (`8080f00b02000100000003010400080000f7`), the script knows the candidate checksum was correct. It records the successful command and moves on to the next payload. 
      * **If non success ACK is received** the checksum is considered incorrect, and the script immediately tries the next one in the sequence. 

This process is repeated for every unique payload until a complete map of all 720 working FX order commands is generated.


### Core Logic Snippet

The heart of the cracker is this simplified JavaScript loop, which demonstrates the methodology:

```javascript
async function findCorrectChecksum(payload) {
    // Iterate through the 256 possible checksums in the 0X0Y format
    for (let x = 0; x < 16; x++) {
        for (let y = 0; y < 16; y++) {
            // Construct the candidate checksum (e.g., "0A0B")
            const checksum = `0${x.toString(16)}0${y.toString(16)}`;
            const testCommand = `8080f0${checksum}${payload}f7`;

            // Send the command via Web Bluetooth
            await writeCommand(testCommand);
            
            // Wait for a response or timeout
            const wasSuccessful = await waitForAck();

            if (wasSuccessful) {
                // If we get a success ACK, we found it!
                return checksum.toUpperCase();
            }
        }
    }
    // If no checksum worked, return null
    return null;
}
```
