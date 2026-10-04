<h1 align="center">
  <!-- <img src="doc/logo.png" width="200" style="padding: 5px;" /> -->
  TetraOP
  <br>
</h1>
<div align="center">

## Release Notes: Performance Update and DSP Optimization

This update introduces a critical architectural revision of the source code aimed at stabilizing the audio thread, reducing CPU usage, and ensuring glitch-free (dropout-free) execution, even under heavy polyphonic and modular processing loads.

### 1. Elimination of String Lookups in the Audio Thread

*   **Description:** Replaced `params.getRawParameterValue("string")` calls within the `processBlock` function with pre-initialized `std::atomic<float>*` pointers.
*   **Technical Rationale:** String-based lookups within the parameter tree (`ValueTreeState`) involve variable and excessively high time complexity for a real-time audio thread. Requesting parameters by name during every buffer cycle forced the CPU to perform constant text-based searches. Caching atomic pointers during plugin instantiation (in the constructor) transforms this into a direct, thread-safe memory access with O(1) constant time complexity, eliminating CPU usage spikes during automation.

### 2. Memory Pre-allocation (Zero-Allocation Audio Thread)

*   **Description:** Moved the declaration, sizing, and allocation of the oversampling buffer (`osBuffer`) from the `processBlock` routine to the `prepareToPlay` function.
*   **Technical Rationale:** Heap memory allocation (creating new buffers or dynamically resizing them) during audio processing is an OS-dependent operation with non-deterministic execution time. If the audio thread is forced to wait for the OS to allocate memory, audio dropouts inevitably occur. Statically pre-allocating the maximum required size during `prepareToPlay` ensures that the DSP operates in a "lock-free" and "allocation-free" context.

### 3. Streamlining the FX Oversampling Workflow

* **Description:** Optimization of routing and elimination of redundant processing cycles associated with upsampling and downsampling, particularly for the `Distortion` module.
* **Technical Rationale:** The polyphase filters (Half-Band Polyphase IIR) used for oversampling are among the most computationally intensive processes for a plugin. Previously, the signal was oversampled, scaled back to the original sample rate, and—in the event of saturation—oversampled again. By consolidating the `processSamplesUp` and `processSamplesDown` logic within the FX chain and avoiding the instantiation of additional transient buffers, the number of calculations required for anti-aliasing filtering has been drastically reduced without compromising final audio quality.

### 4. Vector Acceleration (SIMD) for Visual Metering

* **Description:** Integration of the JUCE-provided `buffer.getMagnitude()` method to calculate and store RMS/Peak values ​​for UI updates.
* **Technical Rationale:** Calculating absolute values ​​and averages to provide visual feedback (meters) can consume valuable DSP resources if processed sample-by-sample using traditional `for` loops. By utilizing JUCE primitives, the compiler delegates the operation to the CPU's SIMD (Single Instruction, Multiple Data) instructions, which process entire blocks of memory simultaneously; this reduces the overhead required to communicate signal amplitude to the GUI to a mere fraction of the original cost.

## Build

```bash
git clone --recurse-submodules https://github.com/tiagolr/tetraop.git

# windows
cmake -G "Visual Studio 18 2026" -DCMAKE_BUILD_TYPE=Release -S . -B ./build
cmake -B build
cmake --build build

# linux
sudo apt update
sudo apt-get install libx11-dev libfreetype-dev libfontconfig1-dev libasound2-dev libxrandr-dev libxinerama-dev libxcursor-dev
cmake -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=Release -S . -B ./build
cmake --build ./build --config Release

# macOS Intel/Silicon
cmake -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=Release -DCMAKE_OSX_ARCHITECTURES="x86_64;arm64" -DCMAKE_OSX_DEPLOYMENT_TARGET="11.0" -S . -B ./build
cmake --build ./build --config Release
```
