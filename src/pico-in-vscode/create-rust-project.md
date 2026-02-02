# Creating a Rust Project for Raspberry Pi Pico in VS Code (with extension)

We have already created the Rust project for the Pico manually and through the template. Now we are going to try another approach: using the Raspberry Pi Pico extension for VS Code.

## Using the Pico Extension

In Visual Studio Code, search for the extension "Raspberry Pi Pico" and ensure you're installing the official one; it should have a verified publisher badge with the official Raspberry Pi website. Install that extension.

<div class="image-with-caption" style="text-align:center; display:inline-block;">
    <img src="./images/Raspberry Pi Pico Vscode extension.png" alt="VSCode Extension for Raspberry Pi Pico" style="max-width:100%; height:auto; display:block; margin:0 auto;"/>
    <div class="caption" style="font-size:0.9em; color:#555; margin-top:6px;">VSCode Extension for Raspberry Pi Pico</div>
</div>

Just installing the extension might not be enough though, depending on what's already on your machine. On Linux, you'll likely need some basic dependencies:

```sh
sudo apt install build-essential libudev-dev
```

## Create Project

Let's create the Rust project with the Pico extension in VS Code. Open the Activity Bar on the left and click the Pico icon. Then choose “New Rust Project.”

<div class="image-with-caption" style="text-align:center; display:inline-block;">
    <img src="./images/Create Project Raspberry Pi Pico Vscode extension.png" alt="Create Project Raspberry Pi Pico Vscode extension" style="max-width:100%; height:auto; display:block; margin:0 auto;"/>
    <div class="caption" style="font-size:0.9em; color:#555; margin-top:6px;">Create Project</div>
</div>

Since this is the first time setting up, the extension will download and install the necessary tools, including the Pico SDK, picotool, OpenOCD, and the ARM and RISC-V toolchains for debugging.


## Project Structure

If the project was created successfully, you should see folders and files like this:

<div class="image-with-caption" style="text-align:center; display:inline-block;">
    <img src="./images/Raspberry Pi Pico Rust Project Created With VS Code Extension.png" alt="Raspberry Pi Pico Rust Project Created With VS Code Extension" style="max-width:100%; height:auto; display:block; margin:0 auto;"/>
    <div class="caption" style="font-size:0.9em; color:#555; margin-top:6px;">Project Folder</div>
</div>

## Switch Board to RP2040

The VS Code Pico extension may default to Pico 2 (RP2350). Since we are targeting the Raspberry Pi Pico based on RP2040, we need to switch the board configuration.

Click the "Switch Board" in the sidebar. From the list, choose Raspberry Pi Pico (RP2040).

<div class="image-with-caption" style="text-align:center; display:inline-block;">
    <img src="./images/Raspberry Pi Pico Vscode extension Switch Boards.png" alt="Raspberry Pi Pico Vscode extension Switch Boards" style="max-width:100%; height:auto; display:block; margin:0 auto;"/>
    <div class="caption" style="font-size:0.9em; color:#555; margin-top:6px;">Switch Board</div>
</div>

## Running the Program

Now you can simply click "Run Project (USB)" to flash the program onto your Pico and run it. Don't forget to press the BOOTSEL button when connecting your Pico to your computer. Otherwise, this option will be in disabled state.

<div class="image-with-caption" style="text-align:center; display:inline-block;">
    <img src="./images/Running Rust Project with Vscode for Raspberry Pi Pico (RP2040).png" alt="Running Rust Project with Vscode for Raspberry Pi Pico (RP2040)" style="max-width:100%; height:auto; display:block; margin:0 auto;"/>
    <div class="caption" style="font-size:0.9em; color:#555; margin-top:6px;">Flashing Rust Firmware into Raspberry Pi Pico</div>
</div>

Once flashing is complete, the program will start running immediately on your Pico. You should see the onboard LED blinking.
