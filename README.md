# SysMonitor++

**A command-line system monitoring tool for Linux**

## Project Overview

**SysMonitor++** is a command-line system monitoring tool for Linux that uses system calls and the `/proc` filesystem to display CPU usage, memory statistics, and active processes. The tool provides both interactive menu-driven operation and command-line options for one-time checks or continuous monitoring.

### Key Features
- Real-time CPU usage monitoring
- Memory usage statistics (Total, Used, Free)
- Top 5 CPU-consuming processes
- Continuous monitoring mode with configurable refresh intervals
- Session logging to `syslog.txt`
- Interactive menu interface
- Command-line argument support

---

## Hardware and Software Requirements

### Hardware Requirements
- Linux-based system (x86, x86_64, ARM)
- Minimum 512 MB RAM
- Access to `/proc` filesystem

### Software Requirements
- **Operating System**: Linux (kernel 2.6+)
- **Compiler**: GCC (GNU Compiler Collection) version 4.8 or higher
- **Libraries**: Standard C library (glibc)
- **Tools**: make (optional, for build automation)

### System Permissions
- Read access to `/proc/stat`, `/proc/meminfo`, and `/proc/[PID]/*`
- Write access to current directory (for `syslog.txt`)

---

## Compilation & Execution

### Compilation

```bash
gcc sysmonitor.c -o sysmonitor
```

### Execution Modes

#### 1. Interactive Menu Mode
```bash
./sysmonitor
```
Launches an interactive menu with options to view CPU usage, memory usage, top processes, or enable continuous monitoring.

#### 2. Single Module Display
```bash
# Display CPU usage once
./sysmonitor -m cpu

# Display memory usage once
./sysmonitor -m mem

# Display top 5 processes once
./sysmonitor -m proc
```

#### 3. Continuous Monitoring Mode
```bash
# Monitor every 2 seconds
./sysmonitor -c 2

# Monitor every 5 seconds
./sysmonitor -c 5
```

#### 4. Help
```bash
./sysmonitor -h
```

### Stopping the Program
- In continuous monitoring or interactive mode: Press `Ctrl+C`
- The program will gracefully save logs and exit

---

## Example Outputs

### CPU Usage Output
```
=== CPU Usage ===
CPU Usage: 23.4%
```

### Memory Usage Output
```
=== Memory Usage ===
Total Memory:  7892 MB
Used Memory:   4523 MB
Free Memory:   3369 MB
Usage:         57.3%
====================
```

### Top 5 Processes Output
```
=== Top 5 Active Processes ===
PID        Process Name                   CPU Time        CPU %
=======================================================================
1234       chrome                         1523456         15.23%
5678       firefox                        1234567         12.35%
9012       systemd                        987654          9.88%
3456       gnome-shell                    876543          8.77%
7890       Xorg                           765432          7.65%
```

### Interactive Menu
```
=== SysMonitor++ Main Menu ===
1. CPU Usage
2. Memory Usage
3. Top 5 Processes
4. Continuous Monitoring
5. Exit
Enter your choice:
```

### Continuous Monitoring Output
```
=== Continuous Monitoring ===
Timestamp: 2025-12-19 15:30:45

=== CPU Usage ===
CPU Usage: 18.7%

=== Memory Usage ===
Total Memory:  7892 MB
Used Memory:   4612 MB
Free Memory:   3280 MB
Usage:         58.4%
====================

=== Top 5 Active Processes ===
PID        Process Name                   CPU Time        Relative %
=======================================================================
[... process list ...]
```

### Sample syslog.txt
```
[2025-12-19 15:28:12] Session started
[2025-12-19 15:28:15] CPU Usage: 23.4%
[2025-12-19 15:28:18] Memory - Total: 7892MB, Used: 4523MB, Free: 3369MB (57.3%)
[2025-12-19 15:28:22] Top 5 processes displayed: Top process PID=1234 (chrome) with 1523456 CPU time (15.23%)
[2025-12-19 15:30:45] Continuous monitoring started
[2025-12-19 15:35:12] SIGINT received
[2025-12-19 15:35:12] Session ended
```

---

## Module Documentation

### 1. Shared Components Module

**Purpose**: Provides common utilities used across all modules.

#### Functions

**`char* getCurrentTimestamp()`**
- Returns current timestamp in "YYYY-MM-DD HH:MM:SS" format
- Uses `time()` and `localtime()` system calls
- Returns pointer to static buffer

**`void writeLog(const char *message)`**
- Writes timestamped message to `syslog.txt`
- Automatically flushes buffer for real-time logging
- Safe to call even if log file is not open

**`void handleSignal(int sig)`**
- Signal handler for `SIGINT` (Ctrl+C)
- Ensures graceful shutdown and log closure
- Prevents data loss on interruption

**`void displayHelp()`**
- Displays usage information and command-line options
- Shows examples for different execution modes

---

### 2. CPU Usage Module (Contributor 1)

**Purpose**: Monitors CPU usage by reading `/proc/stat` and calculating utilization percentages.

#### How It Works
1. Reads CPU statistics from `/proc/stat` using low-level `open()` and `read()` system calls
2. Parses user, nice, system, idle, iowait, irq, and softirq values
3. Calculates CPU usage by comparing current values with previous readings
4. Formula: `CPU Usage = 100 × (1 - idle_delta / total_delta)`

#### Functions

**`int parseCPUStats(const char *buffer, ...)`**
- Parses the "cpu" line from `/proc/stat` buffer
- Extracts 7 fields: user, nice, system, idle, iowait, irq, softirq
- Returns 0 on success, -1 on failure

**`double calculateCPUUsage(unsigned long long user, ...)`**
- Calculates CPU usage percentage from current and previous values
- Maintains static variables for delta calculation
- Returns -1.0 on first run (no previous data)
- Returns percentage (0.0-100.0) on subsequent calls

**`void getCPUUsage()`**
- Main entry point for CPU monitoring
- Handles first-call initialization (requires two readings)
- Displays formatted output
- Logs results to syslog.txt

---

### 3. Memory Usage Module (Contributor 2)

**Purpose**: Retrieves and displays system memory statistics from `/proc/meminfo`.

#### How It Works
1. Opens `/proc/meminfo` using `open()` system call
2. Reads file contents into buffer
3. Parses `MemTotal` and `MemFree` values using `strstr()` and `strtol()`
4. Calculates used memory: `Used = Total - Free`
5. Converts kilobytes to megabytes for display

#### Functions

**`void getMemoryUsage()`**
- Reads memory statistics from `/proc/meminfo`
- Calculates total, used, free memory in MB
- Computes usage percentage
- Displays formatted output
- Logs results to syslog.txt

---

### 4. Top Processes Module (Contributor 3)

**Purpose**: Identifies and displays the top 5 CPU-consuming processes.

#### How It Works
1. Scans `/proc` directory for numeric entries (PIDs)
2. Reads process name from `/proc/[PID]/comm`
3. Reads CPU time from `/proc/[PID]/stat` (utime + stime)
4. Reads total system CPU time from `/proc/stat`
5. Stores process information in array
6. Sorts processes by total CPU time (descending)
7. Calculates CPU percentage relative to total system CPU time
8. Displays top 5 with actual CPU percentages

#### CPU Percentage Calculation
The module calculates CPU percentage using the formula:
CPU % = (Process CPU Time / Total System CPU Time) × 100
Where:
- **Process CPU Time** = utime + stime (from `/proc/[PID]/stat`)
- **Total System CPU Time** = sum of all CPU time fields from `/proc/stat` (user + nice + system + idle + iowait + irq + softirq)

This provides an accurate representation of each process's share of total system CPU resources since boot. The percentages represent cumulative CPU consumption, not instantaneous usage.

#### Data Structure

```c
struct ProcessInfo {
    int pid;
    char name[256];
    unsigned long utime;       // User mode CPU time
    unsigned long stime;       // Kernel mode CPU time
    unsigned long total_time;  // Total CPU time
    double cpu_percent;        // CPU percentage (relative to total system CPU)
};
```

#### Functions

**`int isNumeric(const char *str)`**
- Helper function to check if string contains only digits
- Used to identify PID directories in `/proc`
- Returns 1 if numeric, 0 otherwise

**`int readProcessName(int pid, char *name, size_t name_size)`**
- Reads process name from `/proc/[PID]/comm`
- Removes trailing newline
- Returns 0 on success, -1 if process no longer exists

**`int readProcessStat(int pid, unsigned long *utime, unsigned long *stime)`**
- Reads CPU time from `/proc/[PID]/stat`
- Parses fields 14 (utime) and 15 (stime)
- Handles process name with spaces using parenthesis parsing
- Returns 0 on success, -1 on failure
  
**`unsigned long long getTotalCPUTime()`**
- Reads total system CPU time from `/proc/stat`
- Sums all CPU time fields (user, nice, system, idle, iowait, irq, softirq)
- Returns total CPU time in clock ticks
- Returns 0 on error
  
**`int compareProcesses(const void *a, const void *b)`**
- Comparison function for `qsort()`
- Sorts processes by total_time in descending order
- Used to identify top CPU consumers

**`void listTopProcesses()`**
- Main entry point for process monitoring
- Opens `/proc` directory and scans for processes
- Retrieves total system CPU time for percentage calculation
- Collects up to 1024 process entries
- Sorts and displays top 5 processes
- Calculates CPU percentages relative to total system CPU
- Logs summary with CPU percentage to syslog.txt

#### Understanding the Output

The **CPU %** column shows:
- What percentage of total system CPU time each process has consumed since it started
- Example: A process showing 15.23% has used 15.23% of all available CPU time across all cores since boot
- The sum of all processes' CPU percentages will approach 100% (the total system CPU capacity)
- This is different from instantaneous CPU usage, which would require delta calculations over time

**Note**: These are cumulative percentages representing lifetime CPU consumption, not current/instantaneous CPU usage rates.

---

### 5. Main Control & Continuous Monitoring (Contributor 4)

**Purpose**: Provides user interface and orchestrates monitoring operations.

#### Functions

**`void displayMenu()`**
- Displays interactive menu with 5 options
- Handles user input and dispatches to appropriate modules
- Validates input and clears buffer on errors
- Runs in loop until user chooses to exit

**`void continuousMonitor(int interval)`**
- Implements continuous monitoring mode
- Clears screen and displays timestamp
- Calls all three monitoring modules in sequence
- Sleeps for specified interval
- Runs until interrupted by Ctrl+C

**`int main(int argc, char *argv[])`**
- Parses command-line arguments
- Sets up signal handler for Ctrl+C
- Opens log file (`syslog.txt`)
- Dispatches to appropriate mode based on arguments
- Handles cleanup on exit

#### Command-Line Argument Handling
- No arguments → Interactive menu
- `-h` → Display help
- `-m [cpu|mem|proc]` → Single module execution
- `-c <seconds>` → Continuous monitoring with interval

---

## File Structure

```
TMN4133-SysMonitor/
│
├── sysmonitor.c          # Main source file
├── syslog.txt            # Auto-generated log file
├── README.md             # Documentation file
```

### sysmonitor.c Structure

```
sysmonitor.c
├── Header Includes & Definitions
├── Global Variables & Function Prototypes
│
├── [SHARED COMPONENTS]
│   ├── getCurrentTimestamp()
│   ├── writeLog()
│   ├── handleSignal()
│   └── displayHelp()
│
├── [CPU USAGE MODULE]
│   ├── Static variables for delta calculation
│   ├── parseCPUStats()
│   ├── calculateCPUUsage()
│   └── getCPUUsage()
│
├── [MEMORY USAGE MODULE]
│   └── getMemoryUsage()
│
├── [TOP PROCESSES MODULE]
│   ├── struct ProcessInfo
│   ├── isNumeric()
│   ├── readProcessName()
│   ├── readProcessStat()
│   ├── getTotalCPUTime()
│   ├── compareProcesses()
│   └── listTopProcesses()
│
├── [MAIN CONTROL MODULE]
│   ├── displayMenu()
│   ├── continuousMonitor()
│   └── main()
```

---

## Additional Resources

### Useful Linux Commands for Testing

#### Comparing SysMonitor++ with Other Tools
```bash
# Run SysMonitor++ and top side by side
# Terminal 1
./sysmonitor -c 2

# Terminal 2
top

# Compare CPU readings
watch -n 1 'echo "SysMonitor++:"; ./sysmonitor -m cpu; echo ""; echo "System:"; mpstat 1 1'
```

### Performance Considerations

1. **CPU Usage Calculation**: Requires two readings with a time interval to calculate delta
2. **Process Scanning**: Scanning all PIDs can be intensive on systems with many processes
3. **Continuous Mode**: Lower intervals (<1 second) may impact system performance
4. **File I/O**: All operations use low-level system calls for efficiency
5. **CPU Percentage Calculation**: Reads `/proc/stat` in addition to individual process stats


### Troubleshooting

**Problem**: "Permission denied" when reading /proc files  
**Solution**: Ensure you have read permissions. Some /proc files require root access.

**Problem**: CPU usage shows 0% or incorrect values  
**Solution**: First reading is always skipped. Wait for second calculation or use continuous mode.

**Problem**: Process information incomplete  
**Solution**: Processes may terminate between reading different files. This is normal behavior.

**Problem**: Cannot write to syslog.txt  
**Solution**: Check write permissions in the current directory.

*Problem**: Process CPU percentages seem low or don't add up to 100%  
**Solution**: The percentages shown are cumulative lifetime usage, not instantaneous rates. Newly started processes will show lower percentages even if currently CPU-intensive.

---

## GitHub Repository

**Repository**: TMN4133-SysMonitor  
**URL**: https://github.com/NCTjinn/TMN4133-SysMonitor

### Contributing

This project was developed as a collaborative effort with four contributors:
1. **Contributor 1**: CPU Usage Module
2. **Contributor 2**: Memory Usage Module
3. **Contributor 3**: Top Processes Module
4. **Contributor 4**: Main Control & Continuous Monitoring

---

## License

This project is developed for educational purposes as part of the TMN4133 course.

---

## References

- Linux Programmer's Manual: `man proc`
- Linux System Calls: `man 2 open`, `man 2 read`, `man 2 close`
- Directory Operations: `man 3 opendir`, `man 3 readdir`
- Signal Handling: `man 2 signal`
- Time Functions: `man 2 time`, `man 3 strftime`
- Process Statistics: `man 5 proc` (for `/proc/[pid]/stat` format)
---

**Last Updated**: 19 December 2025
