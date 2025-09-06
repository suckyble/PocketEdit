# Universal Command Analyzer

An intelligent parsing tool designed to reverse-engineer and organize device snooped commands from parameter captures that use a `valuemap`. Unlike tools that blindly test for possibilities, the analyzer uses a logic-based approach to decode the structure of existing, working commands.

---

## How to Use

1.  **Paste Data**: Paste the raw snooped data into the **"1. Snooped Data Input"** text area.
2.  **Analyze**: Click the **"Analyze Commands"** button to process the data.
3.  **Review Results**: Check the **"3. Analysis Results"** box for a sorted list of commands and a report on any missing values.
4.  **Generate Payloads**: If missing values are found, the **"Generate & Download Missing Payloads"** button will activate, allowing you to save the generated commands as a `.txt` file.

---

## How It Works

The analysis process is as follows:

-   **Data Ingestion and Command Extraction**: The user pastes unformatted text containing captured device communications into the input field. The tool scans this data for valid command patterns, specifically any sequence that starts with `` `8080f0` `` and ends with `` `f7` ``, and compiles a list of all unique commands found.

-   **Marker-Based ID Parsing**: For each unique command, the tool searches for a specific, constant hexadecimal sequence (`` `0e0101040800` ``) that acts as an internal marker. Using this marker as a reliable anchor point, the tool accurately extracts the **Module** and **Algorithm IDs** from their fixed positions:
    -   The **Module ID** is the single byte (2 hex characters) immediately following the marker.
    -   The **Algorithm ID** is the single byte located exactly 8 bytes (16 hex characters) after the **Module ID**.

-   **Value Decoding (A Dual-System Approach)**: The tool examines the final 4 bytes (8 hex characters) of the command, known as the `` `value block` ``, to determine its specific setting. It uses two distinct systems for decoding:
    -   **Standard `ValueMap` Logic**: For most parameters, the `` `value block` `` is compared against an internal library of known value maps that cover the full **-50 to +100** integer range.
    -   **Special Case Handling (Delay Time)**: When a command is identified as **Module 7, Algorithm 1**, the tool switches to a separate, lookup map that decodes the `` `value block` `` into a millisecond value between **20ms and 1000ms**.

-   **Sorting and Gap Analysis**: All identified commands are sorted and grouped hierarchically, first by **Algorithm ID** and then by **Module ID**. The tool then generates a clean, readable report that pinpoints any gaps in the captured value sequences for each parameter group. For the special **Delay Time** case, it checks for gaps against the entire known **20-1000ms** range.

-   **Payload Generation and Export**: If the Gap Analysis finds missing values, an option to generate the missing commands becomes available. This export function works by:
    1.  Using a valid, existing command from the user's input as a **template** for that specific parameter group.
    2.  Looking up the correct 8-character hex `` `valuePart` `` for each missing integer.
    3.  Substituting this new `` `valuePart` `` into the template to create a complete, valid command.
    4.  Compiling all generated commands into a single `` `.txt` `` file that the user can download for further use.
