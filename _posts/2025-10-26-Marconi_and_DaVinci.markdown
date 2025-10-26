---
layout: post
title: "Marconi and Da Vinci"
date:   2025-10-26 12:45:00 -0500
categories: projects
tags : ['doxyfile', 'Documentation', 'firmware', 'PRT']
author : "Francesco Abate"

---

# The Brain and Nerves: How Davinci and Marconi Work Together 🧠🔗📡

In the PoliTo Rocket Team's avionics system, we don't rely on a single central computer. Instead, we use a distributed architecture built around two types of custom-designed boards: the Davinci Flight Computer (FC) and the Marconi Telemetry & Control board. Understanding how they collaborate is key to understanding our rocket's electronic heart.

## Defining the Roles: Brain vs. Nervous System

Think of the Davinci as the rocket's brain. Located in the primary compartments (nosecone and avionics bay), it's the computational powerhouse.

    Davinci's Responsibilities:

        Running complex algorithms like the Kalman filter for state estimation.

Executing the Flight State Machine (FSM) to determine the current flight phase.

Making critical decisions, like when to deploy parachutes.

Calculating control outputs, such as the airbrake angle.

Performing high-frequency data logging to its onboard QSPI flash memory.

The Marconi, on the other hand, acts like the rocket's nervous system. Found in multiple locations (nosecone, avionics bay, and near the airbrakes), it handles communication and interacts more directly with the outside world and specific actuators.

    Marconi's Responsibilities:

        Interfacing with the TESEO LIV3FL GNSS module for GPS positioning.

Communicating with the Ground Station via the E220 LoRa radio for telemetry downlink and command uplink.

Facilitating wireless communication between different parts of the rocket using XBEE radios.

Directly controlling local actuators, like the servomotors for the airbrakes via PWM signals.

Storing a unique board ID in its EEPROM.

## Physical Integration and Specialization

The rocket houses multiple boards in different configurations depending on the section's needs:

Nosecone: Contains a Davinci-Marconi duo. The Davinci performs primary flight computation and data logging, while the Marconi handles GPS and LoRa telemetry.

Avionics Bay (AVN Bay): This is the main hub, also featuring a Davinci-Marconi duo. The AVN Bay Marconi performs the same tasks as the nosecone Marconi plus it uses an XBEE radio to communicate commands to the airbrakes section.

Airbrakes: Contains only a Marconi board. This Marconi's sole purpose is to receive commands (specifically the target servo angle) from the AVN Bay Marconi via XBEE and drive the airbrake servomotor.

This distributed setup allows for specialized roles and keeps high-power computation (Davinci) separate from communication-heavy tasks (Marconi).

## Communication Pathways

The boards communicate through several channels:

Davinci <-> Marconi (Wired USART): This is the primary high-speed link within a duo (nosecone or AVN bay). The Davinci sends its processed sensor data, FSM state, calculated airbrake angle, and battery status to the Marconi every 10ms in a structured binary packet (DV-M Packet). The Marconi also sends data back, like the latest GPS velocity, to the Davinci.

AVN Marconi -> Airbrakes Marconi (Wireless XBEE): Used specifically for airbrake control. The AVN Marconi receives the target angle from the Davinci (via USART) and immediately broadcasts it wirelessly using its XBEE radio. The Airbrakes Marconi listens for these XBEE messages.

Marconi -> Ground Station (Wireless LoRa): The Marconis in the nosecone and AVN bay package the data received from their paired Davinci, add their own GPS and battery data, and transmit this combined telemetry packet (M-GS Packet) to the ground station approximately once per second via the LoRa radio. They can also receive commands back from the ground station via LoRa.

## Data Flow Example: Airbrake Control

Let's trace the command path for the airbrakes:

Davinci (AVN Bay): Runs the control algorithm and determines the needed servo angle (e.g., 45.0 degrees).

Davinci -> AVN Marconi (USART): Davinci includes 45.0 in its next DV-M packet sent over the wired link.

AVN Marconi (Processing): Receives the DV-M packet, extracts the 45.0 degree value.

AVN Marconi -> Airbrakes Marconi (XBEE): Broadcasts the 45.0 value wirelessly.

Airbrakes Marconi (Reception & Action): Receives the 45.0 value via its XBEE. Its firmware then calls the servo_moveto_deg() function to generate the correct PWM signal for the connected servomotor.

## Software Strategy: One Firmware, Multiple Roles

Despite having different tasks, all Marconi boards run the exact same firmware. This simplifies development and deployment. How do they know what to do?

EEPROM ID: During initial setup, each Marconi is programmed with a unique character ID ('P' for Airbrakes, 'A' for AVN Bay, 'D' for Nosecone - though the README uses 'D' for Nosecone in the packet description) stored in its onboard EEPROM.

Conditional Logic: On startup, the firmware reads this ID into a global variable. Throughout the code, simple if/else statements check this ID to enable or disable specific functionalities. For example:

        if (board_id == 'A' || board_id == 'D') { /* Initialize LoRa & Send Telemetry */ }

        if (board_id == 'P') { /* Initialize XBEE RX & Control Servo */ }

        if (board_id == 'A') { /* Initialize XBEE TX */ }

## Documentation Deep Dive

For a detailed look into the software structure, specific functions, data types, and configuration of each board, please refer to their respective Doxygen documentation sites:

[**Davinci Doxygen Documentation**]({% link _posts/2025-09-25-DaVinci.markdown %})

[**Marconi Doxygen Documentation**]({% link _posts/2025-10-26-Marconi.markdown %})

## Conclusion

The Davinci and Marconi boards form a synergistic pair. The Davinci handles the heavy lifting of flight computation and control, while the Marconi manages the vital tasks of communication, positioning, and localized actuation. This distributed architecture, connected by well-defined communication links, provides a robust and specialized foundation for our rocket's avionics system.
