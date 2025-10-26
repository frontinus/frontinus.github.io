---
layout: post
title:  "Tutorial: Measuring execution time on STM32 with DWT"
date:   2025-10-26 13:24:00 -0500
categories: tutorials
tags : ['Tutorial', 'Stm32', 'Embedded', 'PRT']
author : "Francesco Abate"
---

# ⏱ High-Precision Code Timing on STM32 with the DWT Cycle Counter

Ever wondered exactly how long a specific part of your STM32 code takes to run? Whether you're debugging, optimizing critical sections, or verifying RTOS task timing, having a precise measurement tool is invaluable. While you could toggle GPIOs and use an oscilloscope, there's a built-in hardware feature on most ARM Cortex-M cores (like those in STM32s) perfect for the job: the DWT (Data Watchpoint and Trace) cycle counter.

This counter increments with every single CPU clock cycle, offering nanosecond-level resolution. This tutorial shows you how to use a simple header file, profiler.h, to easily measure code execution time using this powerful feature.

## Prerequisites

    An STM32 Project (e.g., in STM32CubeIDE).

    The profiler.h header file (as provided in your prompt) added to your project (e.g., in your Core/Inc folder).

    Your SystemCoreClock variable must be correctly configured by your STM32CubeMX setup (it usually is by default).

## Understanding profiler.h

The profiler.h header gives you a set of easy-to-use tools:

    initDWT() / PROFILER_INIT(): A function (or macro alias) to set up the DWT counter.

    ProfileResult: A struct to hold the timing measurement results.

    PROFILER_START(p_result): A macro to record the starting cycle count.

    PROFILER_STOP(p_result): A macro to record the ending count and calculate the elapsed time in cycles, nanoseconds, microseconds, and milliseconds.

    ENABLE_PROFILING: A define you can comment out to completely remove all profiling code and overhead from your release builds.

## Step-by-Step Guide

Here’s how to integrate the profiler into your code:

### Step 1: Initialize the DWT Counter

Once, at the beginning of your main() function (after HAL initialization and clock setup), call the initialization function:
C

    #include "profiler.h" // Include the header

    int main(void) {
        HAL_Init();
        SystemClock_Config();
        MX_GPIO_Init(); // Initialize other peripherals...

        PROFILER_INIT(); // Initialize the DWT counter <--- ADD THIS

        // ... Rest of your main function (RTOS init, etc.)
    }

### Step 2: Declare a Result Variable

In the function where you want to measure execution time, declare a variable of type ProfileResult. This variable will store the start time and the final calculated durations.
C

    #include "profiler.h"
    #include <stdio.h> // For printf

    void MyFunctionToProfile(void) {
        ProfileResult timing_info; // Variable to hold results
        char buffer[100];

    // ... some code ...
    }

### Step 3: Wrap Your Code with START/STOP Macros

Place the PROFILER_START() macro immediately before the code block you want to measure and the PROFILER_STOP() macro immediately after. Pass the address (&) of your ProfileResult variable to both macros.
C

    void MyFunctionToProfile(void) {
        ProfileResult timing_info;
        char buffer[100];

        // ... some code before measurement ...

        PROFILER_START(&timing_info); // <-- Start timing here

        // --- Code to be measured START ---
        volatile uint32_t delay_count = 1000; // Example: Simple delay loop
        while(delay_count--);
        // --- Code to be measured END ---

        PROFILER_STOP(&timing_info); // <-- Stop timing here

        // ... some code after measurement ...
    }

### Step 4: Access and Use the Results

After PROFILER_STOP() has executed, the timing_info variable will be populated with the results. You can access these fields:

    timing_info.elapsed_cycles: Time in raw CPU clock cycles (uint32_t).

    timing_info.elapsed_ns: Time in nanoseconds (float).

    timing_info.elapsed_us: Time in microseconds (float).

    timing_info.elapsed_ms: Time in milliseconds (float).

You can then print these values, log them, or use them in calculations:
C

    void MyFunctionToProfile(void) {
        ProfileResult timing_info;
        char buffer[100];

        // ... code before ...

        PROFILER_START(&timing_info);
        volatile uint32_t delay_count = 1000;
        while(delay_count--);
        PROFILER_STOP(&timing_info);

        // Print the results (ensure you have UART/printf configured)
        sprintf(buffer, "Task took: %lu cycles, %.3f us (%.3f ms)\r\n",
                timing_info.elapsed_cycles,
                timing_info.elapsed_us,
                timing_info.elapsed_ms);
        // Replace HAL_UART_Transmit with your preferred output method
        HAL_UART_Transmit(&huart_debug, (uint8_t*)buffer, strlen(buffer), HAL_MAX_DELAY);

        // ... code after ...
    }

## ⚙️ How It Works (Briefly)

    PROFILER_INIT(): Enables the CoreDebug trace and the DWT cycle counter (DWT->CYCCNT).

    PROFILER_START(): Reads the current value of the free-running DWT->CYCCNT register and stores it.

    PROFILER_STOP(): Reads the DWT->CYCCNT register again. It calculates the difference between the stop and start times to get elapsed_cycles. It then converts cycles to nanoseconds using the system clock frequency (SystemCoreClock) and derives microseconds and milliseconds from that.

## ✅ Disabling Profiling for Release

One of the best features of this header is the ENABLE_PROFILING define. If you comment out this line in profiler.h:
C

    // #define ENABLE_PROFILING // Profiling is now disabled

Then all the profiling macros (PROFILER_INIT, PROFILER_START, PROFILER_STOP) compile to absolutely nothing. This means you can leave the profiling code in place, and it will have zero performance impact on your final release build. The ProfileResult struct also becomes minimal to avoid wasting RAM.

## 💡 Advantages and Considerations

    High Precision: Measures time down to a single CPU clock cycle.

    Low Overhead (when enabled): Reading the DWT register is very fast (usually 1-2 clock cycles). The calculation in PROFILER_STOP adds a small overhead after the measurement.

    Zero Overhead (when disabled): Compiles out completely.

    Hardware-Based: Doesn't rely on software timers, which can have lower resolution or jitter.

⚠️ Keep in Mind:

    CPU Time Only: This method measures the time the CPU spends executing your code. It does not include time spent waiting for peripherals (like DMA completion or blocking HAL calls) unless the CPU is actively busy-waiting.

    Interrupts: If an interrupt occurs during your measured code block, the time spent in the Interrupt Service Routine (ISR) will be included in the measurement. This is usually desirable, as it reflects the real-world execution time.

    Clock Accuracy: The accuracy of the conversion to nanoseconds/microseconds/milliseconds depends entirely on the accuracy of the SystemCoreClock variable. Ensure your clock configuration is correct.

## Conclusion

Using the DWT cycle counter via the profiler.h header provides a simple yet powerful way to get accurate performance measurements for your STM32 code. It's an excellent tool for optimization, debugging timing-sensitive issues, and gaining a deeper understanding of your firmware's runtime behavior.
