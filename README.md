# **PocketEdit** 

**PocketEdit** is a powerful, browser-based web application designed to give you deep, real-time control over your **Sonicake Pocket Master** multi-effects pedal. It provides a user-friendly graphical interface to manage presets, edit effects, and visualize your entire signal chain—all without needing to install any software, since it's written in HTML and JavaScript.

This single, portable HTML file unlocks the full potential of your device, making tone creation and management faster and more intuitive than ever before.
It was created as a fun side project using 99% AI programming logic and 100% hard labour reverse engineering, learning about BLE/HCI snooping and understanding how this device works by looking at logs.


![screenshot](img/gui.jpg)

---

## **Online accessing the App**

You can now directly access PocketEdit online!
The latest version is always available directly in your chrome based browser:
**[https://suckyble.github.io/PocketEdit/](https://suckyble.github.io/PocketEdit/)**

## **Features**

* **Dual Connectivity:** Now supports **USB WebMIDI** (default) with a fallback to **Bluetooth**.
* **Expanded Compatibility:** USB MIDI support allows the editor to work in **Firefox** and other non-Chromium browsers.
* **Instant Preset Saving:** Brute-forcing the CRC is gone! Saving now uses a precise **CRC-8 calculation** for instant, reliable preset writing.
* **Real-Time Editing:** Tweak any parameter and hear the changes instantly on your device.
* **Real-Time Editing:** Tweak any parameter and hear the changes instantly on your device. 
* **Full Preset Management:** Easily browse, search, and load all 50 User and 50 Factory presets.  
* **Visual Signal Chain:** See your entire effects chain at a glance. Simply click a module to edit its parameters.  
* **Automatic Sync:** Upon connecting, the app automatically syncs all preset names, effect types, user-loaded IRs, and global settings from your device, ensuring the editor always matches your hardware.  
* **Unsaved Preset Warning:** The app tracks your changes and warns you if you try to switch presets without saving your modifications.  
* **Advanced Debugging Tools:** A hidden log panel contains manual sync buttons and a communication log for troubleshooting.
* **Tap Tempo:** Use a tap tempo button to calculate milliseconds for delay settings, with 1/8, 1/8 Dotted, 1/4 and 1/2 setting.

## **Requirements**

Before you begin, please ensure you have the following:

* **Hardware:** A **Sonicake Pocket Master** multi-effects pedal.
* **USB Connection (Recommended):** Works in **Google Chrome**, **Microsoft Edge**, and **Firefox**.
* **Bluetooth Connection:** Requires a **Chromium-based browser** (Chrome, Edge, Brave) that supports the **Web Bluetooth API**.
    * *Note: If your PC lacks Bluetooth, a [USB Bluetooth 5.0 Adapter](https://www.amazon.de/dp/B0CZD94YFR) is recommended.*

## **Getting Started**

Getting started with PocketEdit is simple.

1.  **Open the App:** Use the [Online Link](https://suckyble.github.io/PocketEdit/) or open a local copy of the `index.html`.
2.  **Connect your Device:** Click the **"Connect"** button in the top-left sidebar.
4.  **Choose Mode:** The app will default to **USB MIDI**. If you prefer wireless or USB is unavailable, select **"Use Bluetooth"** (Chromium based browser required!).
4.  **Pairing:** If using Bluetooth, select **"Sonic Master BLE"** from the browser popup and click **"Pair"**.
5.  **Automatic Sync:** A loading overlay will appear while the app syncs your device data. Once it disappears, you're ready to edit!
   
## **How to Use the Editor**

* **Loading a Preset:** Use the sidebar on the left to browse the **User** and **Factory** preset banks. Simply click on any preset in the list to load it onto your device.  
* **Editing an Effect:** The signal chain is displayed visually across the top. Click on any module (e.g., `DRV`, `AMP`, `DLY`) to bring up its detailed controls in the main panel below.  
* **Arrange the Signal Chain:** The movable modules (NR, FX1, FX2, DLY, RVB) can be reordered via drag and drop. Simply click and drag a module to a new position in the chain.  
* **Changing an Effect Type:** Within a module's control panel, use the "Type" or "Model" dropdown menu to switch between different effects (e.g., from a "Scream" overdrive to a "Red Fuzz").  
* **Saving Your Changes:** When you modify a preset, an asterisk (`*`) will appear next to its name, indicating it's unsaved. To save your changes, click the **⚙️ gear icon** in the top-right to open the Advanced Settings panel, then click the **"Save Current Preset"** button.

* ## **Credits**

* **USB WebMIDI & Firefox Support:** Huge thanks to [hnikolov](https://github.com/hnikolov) for implementing USB MIDI and the windowing logic.
* **CRC Logic:** Optimized from brute-force to CRC-8 calculation by [hnikolov](https://github.com/hnikolov).

Enjoy crafting your perfect tone\!
