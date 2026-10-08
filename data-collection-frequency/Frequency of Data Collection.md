# **Frequency of Data Collection** 

For all of these charts, the data collection frequency is exactly 1 second (1 Hertz). Every single row in your downloaded files represents a 1-second snapshot of your system. 

# 1. **System CPU** 

   - **user:** Processor used by apps/scripts (e.g., AI, stress test). 

   - **system:** Processor used by the OS background tasks. 

   - **iowait:** CPU sitting idle waiting for the hard drive to finish. 

2. **System RAM** 

   - **used:** Active memory consumed by your apps. 

   - **free:** Completely empty memory. 

   - **cached:** Memory temporarily holding files to make the OS faster. 

3. **Disk I/O** 

   - **in:** Data read from the disk. 

   - **out:** Data written to the disk (appears as a negative number). 

4. **Network Traffic** 

   - **received:** Download speed/spikes. 

   - **sent:** Upload speed (appears as a negative number). 

5. **GPU Utilization** 

   - **gpu:** Percentage of the graphics processor being used (spiked by AI). 

   - **memory:** Percentage of GPU VRAM being used. 

6. **System Load** 

   - **load1:** System overwhelm over the last 1 minute. 

   - **load5:** System overwhelm over the last 5 minutes. 

7. **Swap Memory** 

   - **used:** Hard drive space being used as emergency RAM. 

   - **free:** Available emergency RAM. 

# 8. **Per-Core CPU** 

- **user:** App usage specific to this single core. 

- **system:** OS usage specific to this single core. 

# **1. System CPU (system.cpu)** 

Notice how user jumps from 2% to 85% when a stress test starts. 

|**time**|**user**|**system**|**iowait**|**idle**|**steal**|
|---|---|---|---|---|---|
|2026-09-23 20:20:00|2.15|0.85|0.10|96.90|0|
|2026-09-23 20:20:01|2.10|0.90|0.15|96.85|0|
|2026-09-23 20:20:02|**85.50**|12.10|0.00|2.40|0|
|2026-09-23 20:20:03|**87.20**|11.50|0.00|1.30|0|



# **2. System RAM (system.ram)** 

Notice how used RAM jumps by ~2,000 MB, and free RAM drops to make room for it. 

|**time**|**free**|**used**|**cached**|**buffers**|
|---|---|---|---|---|
|2026-09-23 20:20:00|42486.43|6720.77|14500.94|490.01|
|2026-09-23 20:20:01|42484.89|6722.30|14500.95|490.01|
|2026-09-23 20:20:02|**40484.00**|**8722.30**|14500.95|490.01|
|2026-09-23 20:20:03|**40482.10**|**8724.50**|14500.95|490.01|



# **3. Disk I/O (system.io)** 

Notice how out (writes) becomes a massive negative number when the 2GB dummy file is created. 

|**time**|**in**|**out**|
|---|---|---|
|2026-09-23 20:20:00|12.5|-4.2|
|2026-09-23 20:20:01|0.0|-8.1|
|2026-09-23 20:20:02|0.0|**-440050.2**|
|2026-09-23 20:20:03|1.5|**-445012.8**|



# **4. Network Traffic (system.net)** 

Notice how received shoots up to a huge number when the 1GB test file starts downloading. 

|**time**|**received**|**sent**|
|---|---|---|
|2026-09-23 20:20:00|3.19|-3.49|
|2026-09-23 20:20:01|6.38|-4.72|
|2026-09-23 20:20:02|**55180.50**|-12.40|
|2026-09-23 20:20:03|**56240.10**|-14.20|



# **5. GPU Utilization (nvidia_smi.gpu_utilization)** 

Notice how the graphics card sits at 0% until the Ollama AI starts generating text. 

|**time**|**gpu**|**memory**|
|---|---|---|
|2026-09-23 20:20:00|0|0|
|2026-09-23 20:20:01|0|0|
|2026-09-23 20:20:02|**98**|45|
|2026-09-23 20:20:03|**100**|47|



# **6. System Load (system.load)** 

Notice how load1 (1-minute average) starts climbing rapidly during a stress test. 

|**time**|**load1**|**load5**|**load15**|
|---|---|---|---|
|2026-09-23 20:20:00|0.15|0.20|0.10|
|2026-09-23 20:20:01|0.15|0.20|0.10|
|2026-09-23 20:20:02|**1.50**|0.35|0.12|
|2026-09-23 20:20:03|**2.85**|0.60|0.15|



# **7. Swap Memory (system.swap)** 

If your RAM completely fills up, Linux starts writing memory to the hard drive (used). 

|**time**|**free**|**used**|
|---|---|---|
|2026-09-23 20:20:00|8192.00|0.00|
|2026-09-23 20:20:01|8192.00|0.00|
|2026-09-23 20:20:02|7936.00|**256.00**|
|2026-09-23 20:20:03|7680.00|**512.00**|



# **8. Per-Core CPU (cpu.cpu0)** 

This is just for Core 0. Notice how this specific core hits 100% user usage instantly. 

|**time**|**user**|**system**|**iowait**|**idle**|**steal**|
|---|---|---|---|---|---|
|2026-09-23 20:20:00|1.05|0.50|0.00|98.45|0|
|2026-09-23 20:20:01|1.10|0.20|0.00|98.70|0|
|2026-09-23 20:20:02|**100.00**|0.00|0.00|0.00|0|
|2026-09-23 20:20:03|**100.00**|0.00|0.00|0.00|0|



