# **PocketEdit** 

PocketEdit is a powerful, browser-based web application designed to give you deep, real-time control over your Sonicake Pocket Master multi-effects pedal. It provides a user-friendly graphical interface to manage presets, edit effects, and visualize your entire signal chain, all without needing to install any software.

This single, portable HTML file unlocks the full potential of your device, making tone creation and management faster and more intuitive than ever before.
It was created as a fun side project using 99% AI programminglogic and 100% hard labour reverse engineering.

## **Features**

* **Real-Time Editing:** Tweak any parameter and hear the changes instantly on your device. All controls are synced live via Bluetooth.  
* **Full Preset Management:** Easily browse, search, and load all 50 User and 50 Factory presets.  
* **Visual Signal Chain:** See your entire effects chain at a glance. Simply click a module to edit its parameters.  
* **Automatic Sync:** Upon connecting, the app automatically syncs all preset names, effect types, user-loaded IRs, and global settings from your device, ensuring the editor always matches your hardware.  
* **Unsaved Preset Warning:** The app tracks your changes and warns you if you try to switch presets without saving your modifications.  
* **Advanced Debugging Tools:** A hidden log panel contains manual sync buttons and a communication log for troubleshooting.

## **Requirements**

Before you begin, please ensure you have the following:

* A compatible multi-effects pedal (e.g., Sonicake Pocket GT).  
* A computer or mobile device with Bluetooth functionality.  
  * If your computer does not have built-in Bluetooth, a USB Bluetooth 5.0 adapter can be used. This model was successfully tested during development: [UGREEN Bluetooth 5.0 USB Adapter](https://www.amazon.de/dp/B0CZD94YFR)  
* **Google Chrome** (or a modern Chromium-based browser like Microsoft Edge or Brave) that supports the **Web Bluetooth API**. *This is essential for the app to connect to your device.*

## **Getting Started**

Getting started with PocketEdit is simple.

1. **Open the File:** Open the `PocketEdit.html` file in your Chrome browser.  
2. **Connect to your Device:** Click the blue **"Connect"** button in the top-left sidebar.  
3. **Pair with Bluetooth:** A browser window will pop up. Select **"Sonic Master BLE"** from the list of available devices and click **"Pair"**.  
4. **Automatic Sync:** The app will automatically perform a one-time sync of all presets, names, and settings. A loading overlay will appear while this happens. Once it disappears, you're ready to start editing\!

## **How to Use the Editor**

* **Loading a Preset:** Use the sidebar on the left to browse the **User** and **Factory** preset banks. Simply click on any preset in the list to load it onto your device.  
* **Editing an Effect:** The signal chain is displayed visually across the top. Click on any module (e.g., `DRV`, `AMP`, `DLY`) to bring up its detailed controls in the main panel below.  
* **Arrange the Signal Chain:** The movable modules (NR, FX1, FX2, DLY, RVB) can be reordered via drag and drop. Simply click and drag a module to a new position in the chain.  
* **Changing an Effect Type:** Within a module's control panel, use the "Type" or "Model" dropdown menu to switch between different effects (e.g., from a "Scream" overdrive to a "Red Fuzz").  
* **Saving Your Changes:** When you modify a preset, an asterisk (`*`) will appear next to its name, indicating it's unsaved. To save your changes, click the **⚙️ gear icon** in the top-right to open the Advanced Settings panel, then click the **"Save Current Preset"** button.

Enjoy crafting your perfect tone\!
