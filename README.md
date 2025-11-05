# C++ File Handling Complete Guide

## Core Concepts

**File**: Persistent storage on disk. Data remains after program ends.

**Stream**: Flow of data between program and file. Think of it as a pipeline.

**Buffer**: Temporary memory holding data before writing to disk (improves performance).

---

## Stream Class Hierarchy (Understanding the Design)

```
ios_base (base class - formatting, state)
    ↓
  ios (error states, buffer management)
    ↓
  ┌─────────┬─────────┐
istream   ostream   iostream (both input + output)
    ↓         ↓         ↓
ifstream  ofstream  fstream
```

**Key Insight**: Each class inherits ALL methods from parents above it.

---

## ifstream vs ofstream vs fstream - When to Use What?

### ifstream (Input File Stream)
```cpp
ifstream file("data.txt");
```
**Use when**: Only reading from file
- **Advantages**: 
  - Intent is clear (read-only)
  - Safer (can't accidentally write)
  - Won't create/truncate file if opened with wrong mode
  - Slightly more efficient (no write buffer allocation)
- **Default mode**: `ios::in`

### ofstream (Output File Stream)
```cpp
ofstream file("output.txt");
```
**Use when**: Only writing to file
- **Advantages**:
  - Intent is clear (write-only)
  - Can't accidentally read
  - Automatically truncates file by default
- **Default mode**: `ios::out | ios::trunc`

### fstream (File Stream - Bidirectional)
```cpp
fstream file("data.txt", ios::in | ios::out);
```
**Use when**: Need both reading AND writing
- **Use cases**:
  - Random access files (seek, read, modify, write back)
  - Database-like files
  - Binary files with header + data
  - When you need to read THEN write without reopening
- **No default mode**: MUST specify `ios::in` and/or `ios::out`
- **Important**: Without `ios::trunc`, preserves existing content

### Why Not Always Use fstream?

**4 Reasons**:
1. **Code clarity**: `ifstream` signals "I only read" to other developers
2. **Safety**: Can't accidentally corrupt files with read-only stream
3. **Performance**: No unused buffer allocation for unused direction
4. **Compiler optimization**: More specific types enable better optimizations

**Example of clarity**:
```cpp
// Clear intent
void processData(ifstream& input, ofstream& output) {
    // input can only read, output can only write
}

// Ambiguous
void processData(fstream& file) {
    // What operations are performed? Unclear!
}
```

---

## Complete Method Reference

### Category A: Constructors & File Operations

#### ifstream / ofstream / fstream
```cpp
// Constructors
ifstream()                                    // Default constructor
ifstream(const char* filename, ios::openmode mode = ios::in)
ifstream(const string& filename, ios::openmode mode = ios::in)

// File operations
void open(const char* filename, ios::openmode mode = ios::in)
void open(const string& filename, ios::openmode mode = ios::in)
bool is_open() const                          // Check if file is open
void close()                                  // Close file
```

**Inner mechanism**: 
- Constructor calls `open()` internally
- `open()` allocates buffer, opens OS file handle
- `close()` flushes buffer, releases handle, deallocates buffer
- Destructor calls `close()` automatically (RAII)

---

### Category B: Stream State & Error Checking

#### Inherited from ios
```cpp
bool good() const      // No error, ready for I/O
bool eof() const       // End-of-file reached
bool fail() const      // Logical error (e.g., format mismatch)
bool bad() const       // Read/write error (hardware/corruption)
explicit operator bool() const  // Same as !fail()
bool operator!() const // Same as fail()

void clear(iostate state = goodbit)  // Reset error flags
iostate rdstate() const              // Get current state
void setstate(iostate state)         // Set additional state flags
iostate exceptions() const           // Get exception mask
void exceptions(iostate except)      // Set exception mask
```

**State flags** (can be combined with `|`):
- `goodbit` (0) - No errors
- `eofbit` - End of file reached
- `failbit` - Logical error
- `badbit` - Read/write error

**Inner mechanism**:
```cpp
// Each read/write updates internal state
file >> x;  // If fails, sets failbit
if (file.fail()) { /* handle */ }

// Must clear before retrying
file.clear();  // Resets to goodbit
file >> x;     // Try again
```

**Key point**: Once error flag is set, ALL subsequent operations fail until `clear()` is called.

---

### Category C: Input Methods (ifstream / fstream with ios::in)

```cpp
// Single character
istream& get(char& ch)                       // Extract one char
int get()                                    // Return char as int (-1 on EOF)
istream& get(char* s, streamsize n)          // Read n-1 chars or until '\n'
istream& get(char* s, streamsize n, char delim)  // Custom delimiter

// String reading
istream& getline(char* s, streamsize n)      // Read line (n-1 chars max)
istream& getline(char* s, streamsize n, char delim)  // Custom delimiter

// Note: Global function (not member)
istream& getline(istream& is, string& str)   // Read line into string
istream& getline(istream& is, string& str, char delim)

// Multiple characters
istream& read(char* s, streamsize n)         // Read exactly n bytes (binary)
streamsize readsome(char* s, streamsize n)   // Read available bytes (non-blocking)

// Position inspection
int peek()                                   // Look at next char without extracting
istream& putback(char c)                     // Return char to stream
istream& unget()                             // Return last read char

// Skipping
istream& ignore(streamsize n = 1, int delim = EOF)  // Skip n chars or until delim
streamsize gcount() const                    // Chars extracted by last unformatted input

// Formatted input
istream& operator>>(type& val)               // Extract int, float, string, etc.

// Synchronization
int sync()                                   // Synchronize with underlying device
```

**Inner mechanism deep dive**:

**get() vs operator>>**:
```cpp
char ch;
file.get(ch);    // Reads 'a', ' ', '\n' - EVERYTHING
file >> ch;      // Skips whitespace, reads 'a' only

// Example with "a b\n"
file.get(ch);    // ch = 'a'
file.get(ch);    // ch = ' '
file.get(ch);    // ch = 'b'
file.get(ch);    // ch = '\n'
```

**getline() behavior**:
```cpp
// Reads until '\n', extracts '\n' but doesn't store it
getline(file, line);  // "Hello\n" → line = "Hello"

// The newline is REMOVED from stream
```

**read() for binary**:
```cpp
char buffer[100];
file.read(buffer, 100);  // Reads EXACTLY 100 bytes

// No null terminator added! Manual handling needed:
buffer[file.gcount()] = '\0';
```

**gcount() critical detail**:
```cpp
file.read(buffer, 100);
if (file.gcount() < 100) {
    // Either EOF or error
    if (file.eof()) {
        // Normal: file smaller than 100 bytes
    } else if (file.fail()) {
        // Error: read operation failed
    }
}
```

---

### Category D: Output Methods (ofstream / fstream with ios::out)

```cpp
// Single character
ostream& put(char c)                         // Write one char

// Multiple characters  
ostream& write(const char* s, streamsize n)  // Write n bytes (binary)

// Formatted output
ostream& operator<<(type val)                // Insert int, float, string, etc.

// Buffer control
ostream& flush()                             // Force write buffer to disk
```

**Inner mechanism**:

**Buffer mechanics**:
```cpp
file << "Hello";  // Data goes to BUFFER (RAM), not disk yet

// Buffer flushes to disk when:
// 1. Buffer full (~8KB typically)
// 2. file.flush() called explicitly
// 3. endl encountered (writes '\n' + flush)
// 4. File closed
// 5. Program exits normally

file << endl;     // Equivalent to: file << '\n' << flush;
```

**Critical point**: If program crashes before flush, buffered data is LOST!

**Safe pattern**:
```cpp
file << "Critical data" << flush;  // Immediate write
// or
file << "Critical data" << endl;   // Newline + flush
```

**write() vs operator<<**:
```cpp
// operator<< converts to text
int x = 65;
file << x;           // Writes "65" (2 bytes)

// write() writes raw bytes
file.write((char*)&x, sizeof(x));  // Writes binary (4 bytes)

// Reading back:
file >> x;                         // Reads "65" as integer
file.read((char*)&x, sizeof(x));   // Reads raw 4 bytes
```

---

### Category E: Position Management (Seeking/Telling)

```cpp
// Get current position
streampos tellg()                            // Get position (input)
streampos tellp()                            // Put position (output)

// Set position
istream& seekg(streampos pos)                // Absolute position (input)
istream& seekg(streamoff off, ios::seekdir dir)  // Relative position
ostream& seekp(streampos pos)                // Absolute position (output)
ostream& seekp(streamoff off, ios::seekdir dir)  // Relative position

// Seek directions
ios::beg    // Beginning of stream
ios::cur    // Current position
ios::end    // End of stream
```

**Inner mechanism**:

**Why separate get/put pointers?**
```cpp
fstream file("data.txt", ios::in | ios::out);

// Two independent pointers:
file.seekg(10, ios::beg);  // Read pointer at byte 10
file.seekp(20, ios::beg);  // Write pointer at byte 20

// Can read from byte 10, write to byte 20 independently
char data[5];
file.read(data, 5);   // Reads bytes 10-14
file.write("XYZ", 3); // Writes at bytes 20-22
```

**streampos vs streamoff**:
- `streampos`: Absolute position (like "byte 42")
- `streamoff`: Offset (like "+10 bytes" or "-5 bytes")

**Seeking examples**:
```cpp
// Get file size
file.seekg(0, ios::end);     // Move to end
streampos size = file.tellg(); // Get position = size
file.seekg(0, ios::beg);     // Reset to start

// Read last 100 bytes
file.seekg(-100, ios::end);
char buffer[100];
file.read(buffer, 100);

// Skip 50 bytes forward
file.seekg(50, ios::cur);
```

**Critical gotcha**:
```cpp
// Text mode seeking is UNRELIABLE
ifstream file("data.txt");  // Text mode
file.seekg(10, ios::beg);   // May not be byte 10 on Windows!
// Reason: '\n' → '\r\n' conversion makes positions unpredictable

// Always use binary mode for seeking:
ifstream file("data.txt", ios::binary);
file.seekg(10, ios::beg);   // Reliable
```

---

### Category F: Buffer Management

```cpp
// Get buffer pointer
streambuf* rdbuf() const                     // Get associated buffer
streambuf* rdbuf(streambuf* sb)              // Set new buffer

// Synchronization
basic_ios& copyfmt(const basic_ios& rhs)     // Copy format settings
```

**Inner mechanism**:

**What is rdbuf()?**
```cpp
// Every stream has a streambuf managing actual I/O
ifstream file("data.txt");
streambuf* buf = file.rdbuf();

// streambuf handles:
// - Actual OS file operations
// - Buffering strategy
// - Character encoding

// Super efficient file copy:
ofstream dest("copy.txt");
dest << src.rdbuf();  // Direct buffer-to-buffer transfer
```

**Custom buffering**:
```cpp
// Default buffer: ~8KB
// Custom buffer for performance:
const int BUFSIZE = 64 * 1024;  // 64KB
char buffer[BUFSIZE];
file.rdbuf()->pubsetbuf(buffer, BUFSIZE);
```

---

### Category G: Formatting (Inherited from ios_base)

```cpp
// Flags
fmtflags flags() const                       // Get format flags
fmtflags flags(fmtflags fmtfl)              // Set format flags
fmtflags setf(fmtflags fmtfl)               // Set specific flags
fmtflags setf(fmtflags fmtfl, fmtflags mask) // Set flags with mask
void unsetf(fmtflags mask)                   // Clear flags

// Width & Precision
streamsize width() const                     // Get field width
streamsize width(streamsize wide)            // Set field width
streamsize precision() const                 // Get precision
streamsize precision(streamsize prec)        // Set precision

// Fill character
char fill() const                            // Get fill character
char fill(char fillch)                       // Set fill character

// Locale
locale imbue(const locale& loc)              // Set locale
locale getloc() const                        // Get locale
```

**Inner mechanism**:

**Format flags**:
```cpp
// Number base
file.setf(ios::hex, ios::basefield);  // Hexadecimal
file << 255;  // Writes "ff"

// Float format
file.setf(ios::fixed, ios::floatfield);
file.precision(2);
file << 3.14159;  // Writes "3.14"

// Boolean format
file.setf(ios::boolalpha);
file << true;  // Writes "true" not "1"
```

**Using manipulators (easier)**:
```cpp
#include <iomanip>

file << hex << 255;              // ff
file << fixed << setprecision(2) << 3.14159;  // 3.14
file << setw(10) << setfill('0') << 42;       // 0000000042
```

---

### Category H: Tie & Sync

```cpp
// Stream tying (flushing relationship)
ostream* tie() const                         // Get tied stream
ostream* tie(ostream* tiestr)               // Set tied stream

// Sync with C stdio
static bool sync_with_stdio(bool sync = true) // Sync with printf/scanf
```

**Inner mechanism**:

**Stream tying**:
```cpp
// cin is tied to cout by default
cin.tie(&cout);

// What this means:
cout << "Enter name: ";  // Goes to buffer
cin >> name;             // AUTOMATICALLY flushes cout first!

// Without tying:
cout << "Enter name: ";  // Stays in buffer
cin >> name;             // User sees nothing, confused!
```

**sync_with_stdio**:
```cpp
// Default: C++ streams sync with C stdio (slow)
ios::sync_with_stdio(false);  // Disable sync = faster

// But now can't mix:
cout << "C++";
printf("C");     // Output may be out of order!

// Common optimization:
int main() {
    ios::sync_with_stdio(false);  // Speed up
    cin.tie(nullptr);             // Untie cin/cout for speed
    // Now only use cout/cin, never printf/scanf
}
```

---

## Complete Method Summary Table

| Method | ifstream | ofstream | fstream | Purpose |
|--------|----------|----------|---------|---------|
| **Constructors & File Ops** |
| `open()` | ✓ | ✓ | ✓ | Open file |
| `close()` | ✓ | ✓ | ✓ | Close file |
| `is_open()` | ✓ | ✓ | ✓ | Check if open |
| **State Checking** |
| `good()` | ✓ | ✓ | ✓ | No errors |
| `eof()` | ✓ | ✓ | ✓ | End of file |
| `fail()` | ✓ | ✓ | ✓ | Logical error |
| `bad()` | ✓ | ✓ | ✓ | Read/write error |
| `clear()` | ✓ | ✓ | ✓ | Reset error flags |
| `rdstate()` | ✓ | ✓ | ✓ | Get state flags |
| `setstate()` | ✓ | ✓ | ✓ | Set state flags |
| `exceptions()` | ✓ | ✓ | ✓ | Get/set exception mask |
| **Input Methods** |
| `get()` | ✓ | ✗ | ✓ | Read character |
| `getline()` | ✓ | ✗ | ✓ | Read line |
| `read()` | ✓ | ✗ | ✓ | Read bytes |
| `readsome()` | ✓ | ✗ | ✓ | Read available bytes |
| `peek()` | ✓ | ✗ | ✓ | Look ahead |
| `putback()` | ✓ | ✗ | ✓ | Return character |
| `unget()` | ✓ | ✗ | ✓ | Undo last get |
| `ignore()` | ✓ | ✗ | ✓ | Skip characters |
| `gcount()` | ✓ | ✗ | ✓ | Count last read |
| `operator>>` | ✓ | ✗ | ✓ | Formatted input |
| `sync()` | ✓ | ✗ | ✓ | Sync with device |
| **Output Methods** |
| `put()` | ✗ | ✓ | ✓ | Write character |
| `write()` | ✗ | ✓ | ✓ | Write bytes |
| `operator<<` | ✗ | ✓ | ✓ | Formatted output |
| `flush()` | ✗ | ✓ | ✓ | Flush buffer |
| **Position Management** |
| `tellg()` | ✓ | ✗ | ✓ | Get read position |
| `tellp()` | ✗ | ✓ | ✓ | Get write position |
| `seekg()` | ✓ | ✗ | ✓ | Set read position |
| `seekp()` | ✗ | ✓ | ✓ | Set write position |
| **Buffer Management** |
| `rdbuf()` | ✓ | ✓ | ✓ | Get/set buffer |
| **Formatting** |
| `flags()` | ✓ | ✓ | ✓ | Get/set format flags |
| `setf()` | ✓ | ✓ | ✓ | Set format flags |
| `unsetf()` | ✓ | ✓ | ✓ | Clear format flags |
| `width()` | ✓ | ✓ | ✓ | Field width |
| `precision()` | ✓ | ✓ | ✓ | Float precision |
| `fill()` | ✓ | ✓ | ✓ | Fill character |
| `imbue()` | ✓ | ✓ | ✓ | Set locale |
| **Tying & Sync** |
| `tie()` | ✓ | ✓ | ✓ | Tie streams |
| `sync_with_stdio()` | ✓ | ✓ | ✓ | Sync with C I/O |

---

## Opening Files - Deep Dive

### File Modes (ios flags)

| Mode | Binary | Purpose | Creates if Missing | Preserves Content | Position |
|------|--------|---------|-------------------|-------------------|----------|
| `ios::in` | 0b00001 | Read | No | Yes | Start |
| `ios::out` | 0b00010 | Write | Yes | No (truncates) | Start |
| `ios::app` | 0b00100 | Append | Yes | Yes | End |
| `ios::ate` | 0b01000 | At End | Depends | Yes | End |
| `ios::trunc` | 0b10000 | Truncate | Yes | No | Start |
| `ios::binary` | 0b100000 | Binary | Depends | Depends | Depends |

**Combining modes**:
```cpp
// Read + Write without truncate
fstream file("data.txt", ios::in | ios::out);

// Write at end (append)
ofstream file("log.txt", ios::out | ios::app);

// Read existing or create new binary file
fstream file("data.bin", ios::in | ios::out | ios::binary);
// If file missing, this FAILS without ios::trunc or ios::app

// Create if missing, read/write binary
fstream file("data.bin", ios::in | ios::out | ios::binary | ios::trunc);
```

**Inner mechanism**:
```cpp
// What happens in open()?
void open(const char* filename, ios::openmode mode) {
    1. Parse mode flags
    2. Call OS: open/CreateFile/fopen
    3. If ios::trunc: truncate file to 0 bytes
    4. If ios::ate: seek to end
    5. Allocate buffer (~8KB)
    6. Set stream state to goodbit
}
```

**Critical combinations**:
```cpp
// ✓ Read + Write, preserve content
ios::in | ios::out

// ✓ Write + Append (always writes at end)
ios::out | ios::app

// ✓ Binary read/write
ios::in | ios::out | ios::binary

// ✗ Contradictory (trunc removes content, but trying to read)
ios::in | ios::trunc  // Makes no sense!

// ✗ Append + ate (both seek to end, redundant)
ios::app | ios::ate  // ate is pointless here
```

---

## Text vs Binary Mode - Inner Workings

### Text Mode (default)
```cpp
ofstream file("data.txt");  // Text mode
```

**What happens**:
1. **Newline conversion**:
   - Linux: `'\n'` → `'\n'` (no change)
   - Windows: `'\n'` → `'\r\n'` (adds carriage return)
   - Mac (old): `'\n'` → `'\r'`

2. **End-of-file marker**:
   - Windows: `Ctrl+Z` (0x1A) treated as EOF
   - Linux: No special EOF character

3. **Character encoding**:
   - May perform locale-based conversions
   - UTF-8, Latin-1, etc.

**Why this matters**:
```cpp
// Text mode
ofstream file("data.txt");
file << "Line1\nLine2\n";  // Writes 12 chars on Linux, 14 on Windows

// File sizes differ across platforms!
```

### Binary Mode
```cpp
ofstream file("data.bin", ios::binary);
```

**What happens**:
1. **No transformations**: Byte-for-byte copy
2. **No newline conversion**: `'\n'` stays `'\n'`
3. **No EOF handling**: Reads entire file including 0x1A
4. **Platform-independent file sizes**

**When to use binary**:
```cpp
// 1. Structs/Classes
struct Data { int x; float y; };
Data d = {10, 3.14};
file.write((char*)&d, sizeof(d));  // Must use binary!

// 2. Images, audio, video
file.write(imageData, imageSize);

// 3. Exact byte operations
char byte = 0x1A;  // Ctrl+Z
file.write(&byte, 1);  // Text mode would treat as EOF!

// 4. Cross-platform files
// Binary ensures same byte count everywhere
```

---

## Buffer Mechanics - Deep Dive

### What is a buffer?

```
Program → Buffer (RAM) → OS → Disk
           ↑
        ~8KB default
```

**Why buffering?**
- **Disk I/O is SLOW**: ~100,000x slower than RAM
- **Writing 1 byte at a time = 1000s of disk operations**
- **Solution**: Collect bytes in buffer, write once

### Buffer lifecycle

```cpp
ofstream file("data.txt");

file << "A";     // ┐
file << "B";     // ├→ Goes to buffer (RAM)
file << "C";     // ┘

// Disk still empty!

file << endl;    // Flushes: Buffer → Disk
// OR
file.flush();    // Explicit flush
// OR
file.close();    // Flushes automatically
// OR
// Buffer full (8KB) → Auto flush
```

### Buffer strategies

```cpp
// 1. Unbuffered (immediate write)
file.rdbuf()->pubsetbuf(0, 0);  // No buffer
file << "X";  // Writes to disk immediately (slow!)

// 2. Line buffered
// Flushes on '\n' (default for console)

// 3. Full buffered (default for files)
// Flushes when full or manually
```

### Performance impact

```cpp
// ❌ Slow: 1000 disk writes
for (int i = 0; i < 1000; i++) {
    file << i << flush;  // Flushing each time!
}

// ✅ Fast: 1 disk write
for (int i = 0; i < 1000; i++) {
    file << i << '\n';
}
file.flush();  // Once at end
```

**Benchmark**: Flushing every write can be **100x slower**!

---

## Error Handling - Inner Mechanism

### Error flags deep dive

```cpp
// Internal state bits
goodbit = 0b0000  // All OK
eofbit  = 0b0001  // End of file
failbit = 0b0010  // Logical error
badbit  = 0b0100  // Serious error
```

**When each flag is set**:
```cpp
// eofbit
file >> x;  // Reads past end → eofbit set

// failbit
int x;
file >> x;  // File contains "abc" → failbit set

// badbit
// Disk full
// Hardware failure
// File permissions changed mid-operation
```

**Flag combinations**:
```cpp
// eof() + fail() both true
while (file >> x) {  // Last read fails at EOF
    // eofbit = 1, failbit = 1
}

// Checking properly:
if (file.fail()) {
    if (file.eof()) {
        // Normal end
    } else {
        // Actual error
    }
}
```

### Exception-based error handling

```cpp
// Enable exceptions
file.exceptions(ios::failbit | ios::badbit);

try {
    file >> x;  // Throws if fails
} catch (ios::failure& e) {
    cerr << "Error: " << e.what() << endl;
}

// eof doesn't throw by default (usually expected)
file.exceptions(ios::failbit | ios::badbit | ios::eofbit);
// Now EOF also throws
```

**When to use exceptions**:
- Critical files (config, database)
- Simplifies error propagation
- When failure is exceptional (not expected)

**When NOT to use**:
- Reading user input (errors expected)
- Performance-critical loops
- When EOF is normal condition

---

## Advanced Concepts

### 1. Atomic writes (crash-safe)

```cpp
// ❌ Risky: Partial write if crash
ofstream file("data.txt");
file << "Important data";
// Crash here = data lost or corrupted!

// ✅ Safe: Write to temp, then rename
ofstream temp("data.txt.tmp");
temp << "Important data";
temp.close();  // Ensure written
rename("data.txt.tmp", "data.txt");  // Atomic on POSIX
```

### 2. File locking (prevent concurrent access)

```cpp
#include <sys/file.h>  // POSIX
int fd = open("data.txt", O_RDWR);
flock(fd, LOCK_EX);  // Exclusive lock

// Critical section
// ...

flock(fd, LOCK_UN);  // Unlock
close(fd);
```

### 3. Memory-mapped files

```cpp
// Traditional: Read → Buffer → Process
char buffer[1000];
file.read(buffer, 1000);
process(buffer);

// Memory-mapped: File appears as RAM
// OS handles paging automatically
// Much faster for random access
```

### 4. Asynchronous I/O

```cpp
// Traditional: Blocking
file.read(buffer, size);  // Waits here
process(buffer);

// Async: Non-blocking
auto future = async(launch::async, [&]() {
    file.read(buffer, size);
});
doOtherWork();  // Runs while reading
future.wait();  // Wait when needed
```

---

## Performance Optimization Strategies

### 1. Buffer size tuning

```cpp
// Default: 8KB
// SSD: 64-128KB optimal
// Network drive: 256KB-1MB optimal

char buffer[128 * 1024];  // 128KB
file.rdbuf()->pubsetbuf(buffer, sizeof(buffer));
```

### 2. Batch operations

```cpp
// ❌ Slow: Many small reads
for (int i = 0; i < 1000; i++) {
    int x;
    file.read((char*)&x, sizeof(x));
}

// ✅ Fast: One large read
int data[1000];
file.read((char*)data, sizeof(data));
```

### 3. Disable sync for speed

```cpp
// Competitive programming trick
ios::sync_with_stdio(false);
cin.tie(nullptr);
// Can speed up I/O by 2-3x
```

### 4. Use binary for structured data

```cpp
// ❌ Slow: Text conversion
file << 123456789;  // Converts int → string → bytes

// ✅ Fast: Direct bytes
file.write((char*)&num, sizeof(num));
```

---

## Common Pitfalls - Root Causes

### 1. Mixed >> and getline()
```cpp
int age;
string name;
file >> age;        // Reads "25", leaves "\n"
getline(file, name); // Reads "" (empty line!)

// Root cause: >> stops at whitespace, doesn't consume it
// Fix:
file >> age;
file.ignore(numeric_limits<streamsize>::max(), '\n');
getline(file, name);
```

### 2. Forgetting to clear() after error
```cpp
file >> x;  // Fails, sets failbit
file >> y;  // FAILS IMMEDIATELY (failbit still set!)

// Root cause: Error flags persist until cleared
// Fix:
file.clear();  // Reset flags
file >> y;     // Now works
```

### 3. Text mode seeking
```cpp
// ❌ Unreliable on Windows
ifstream file("data.txt");  // Text mode
file.seekg(100, ios::beg);  // NOT byte 100!

// Root cause: '\n' → '\r\n' conversion makes positions unpredictable
// Fix: Always use binary for seeking
ifstream file("data.txt", ios::binary);
file.seekg(100, ios::beg);  // Reliable
```

### 4. Not checking open() success
```cpp
// ❌ Undefined behavior
ifstream file("missing.txt");
file >> x;  // File never opened!

// Root cause: Constructor doesn't throw, just sets failbit
// Fix: Always check
if (!file.is_open()) {
    // Handle error
}
```

### 5. Reading binary structs with padding
```cpp
struct Data {
    char c;     // 1 byte
    int x;      // 4 bytes, but padded to 8 total!
};

// ❌ Writing with padding
file.write((char*)&d, sizeof(d));  // Writes garbage padding bytes

// Root cause: Compiler adds padding for alignment
// Fix: Use #pragma pack or serialize members individually
#pragma pack(push, 1)
struct Data {
    char c;
    int x;
};
#pragma pack(pop)
```

---

## Real-World Patterns

### Pattern 1: Safe file update
```cpp
bool safeUpdate(const string& filename, const string& data) {
    string temp = filename + ".tmp";
    
    // Write to temp
    ofstream out(temp, ios::binary);
    if (!out) return false;
    out << data;
    out.close();
    
    // Verify written
    ifstream verify(temp, ios::binary);
    string written((istreambuf_iterator<char>(verify)), 
                   istreambuf_iterator<char>());
    if (written != data) {
        remove(temp.c_str());
        return false;
    }
    
    // Atomic replace
    remove(filename.c_str());
    rename(temp.c_str(), filename.c_str());
    return true;
}
```

### Pattern 2: Binary record database
```cpp
struct Record {
    int id;
    char name[50];
    float value;
};

class RecordDB {
    fstream file;
    
public:
    RecordDB(const string& filename) {
        file.open(filename, ios::in | ios::out | ios::binary);
        if (!file) {
            // Create new file
            file.open(filename, ios::out | ios::binary);
            file.close();
            file.open(filename, ios::in | ios::out | ios::binary);
        }
    }
    
    bool read(int index, Record& rec) {
        file.seekg(index * sizeof(Record), ios::beg);
        file.read((char*)&rec, sizeof(Record));
        return file.gcount() == sizeof(Record);
    }
    
    bool write(int index, const Record& rec) {
        file.seekp(index * sizeof(Record), ios::beg);
        file.write((char*)&rec, sizeof(Record));
        return !file.fail();
    }
    
    int count() {
        file.seekg(0, ios::end);
        return file.tellg() / sizeof(Record);
    }
};
```

### Pattern 3: Chunked file processing (memory-efficient)
```cpp
void processLargeFile(const string& input, const string& output) {
    ifstream in(input, ios::binary);
    ofstream out(output, ios::binary);
    
    const size_t CHUNK = 1024 * 1024;  // 1MB chunks
    char* buffer = new char[CHUNK];
    
    while (in.read(buffer, CHUNK) || in.gcount() > 0) {
        size_t bytes = in.gcount();
        
        // Process chunk
        for (size_t i = 0; i < bytes; i++) {
            buffer[i] = toupper(buffer[i]);
        }
        
        out.write(buffer, bytes);
    }
    
    delete[] buffer;
}
```

### Pattern 4: CSV with error recovery
```cpp
vector<vector<string>> parseCSV(const string& filename) {
    ifstream file(filename);
    vector<vector<string>> data;
    string line;
    int lineNum = 0;
    
    while (getline(file, line)) {
        lineNum++;
        vector<string> row;
        stringstream ss(line);
        string cell;
        
        while (getline(ss, cell, ',')) {
            // Trim whitespace
            cell.erase(0, cell.find_first_not_of(" \t"));
            cell.erase(cell.find_last_not_of(" \t") + 1);
            row.push_back(cell);
        }
        
        if (!row.empty()) {
            data.push_back(row);
        } else {
            cerr << "Warning: Empty line " << lineNum << endl;
        }
    }
    
    return data;
}
```

### Pattern 5: Binary file with header
```cpp
struct FileHeader {
    char magic[4] = {'M', 'Y', 'F', 'L'};
    uint32_t version = 1;
    uint32_t recordCount = 0;
};

void writeRecords(const string& filename, const vector<Record>& records) {
    ofstream file(filename, ios::binary);
    
    // Write header
    FileHeader header;
    header.recordCount = records.size();
    file.write((char*)&header, sizeof(header));
    
    // Write records
    file.write((char*)records.data(), records.size() * sizeof(Record));
}

vector<Record> readRecords(const string& filename) {
    ifstream file(filename, ios::binary);
    
    // Read and validate header
    FileHeader header;
    file.read((char*)&header, sizeof(header));
    
    if (strncmp(header.magic, "MYFL", 4) != 0) {
        throw runtime_error("Invalid file format");
    }
    
    if (header.version != 1) {
        throw runtime_error("Unsupported version");
    }
    
    // Read records
    vector<Record> records(header.recordCount);
    file.read((char*)records.data(), 
              header.recordCount * sizeof(Record));
    
    return records;
}
```

---

## File Operations with C++17 filesystem

```cpp
#include <filesystem>
namespace fs = std::filesystem;

// Check existence
if (fs::exists("data.txt")) { /* ... */ }

// File size
uintmax_t size = fs::file_size("data.txt");

// Copy/move/delete
fs::copy_file("src.txt", "dst.txt");
fs::copy_file("src.txt", "dst.txt", fs::copy_options::overwrite_existing);
fs::rename("old.txt", "new.txt");
fs::remove("file.txt");

// Directory operations
fs::create_directory("mydir");
fs::create_directories("path/to/nested");
fs::remove_all("directory");  // Recursive delete

// Iterate directory
for (const auto& entry : fs::directory_iterator(".")) {
    if (entry.is_regular_file()) {
        cout << entry.path() << " - " 
             << entry.file_size() << " bytes\n";
    }
}

// Recursive iteration
for (const auto& entry : fs::recursive_directory_iterator(".")) {
    cout << entry.path() << endl;
}

// File permissions
fs::permissions("file.txt", fs::perms::owner_all | fs::perms::group_read);

// Last write time
auto time = fs::last_write_time("data.txt");
auto sctp = chrono::time_point_cast<chrono::system_clock::duration>(
    time - fs::file_time_type::clock::now() + 
    chrono::system_clock::now()
);
time_t cftime = chrono::system_clock::to_time_t(sctp);
cout << ctime(&cftime);

// Path manipulation
fs::path p = "data.txt";
cout << p.filename() << endl;      // "data.txt"
cout << p.stem() << endl;          // "data"
cout << p.extension() << endl;     // ".txt"
cout << p.parent_path() << endl;   // ""

fs::path full = "/home/user/data.txt";
cout << full.root_path() << endl;      // "/"
cout << full.parent_path() << endl;    // "/home/user"

// Canonical path (resolve symlinks, .., .)
fs::path canonical = fs::canonical("../data.txt");

// Space info
fs::space_info space = fs::space(".");
cout << "Capacity: " << space.capacity << endl;
cout << "Free: " << space.free << endl;
cout << "Available: " << space.available << endl;
```

---

## Thread Safety Considerations

### File streams are NOT thread-safe

```cpp
// ❌ Race condition
ofstream logFile("app.log", ios::app);

void thread1() { logFile << "Thread 1\n"; }
void thread2() { logFile << "Thread 2\n"; }
// Output may be interleaved: "ThrTehared 21\n\n"

// ✅ Use mutex
mutex fileMutex;
ofstream logFile("app.log", ios::app);

void thread1() {
    lock_guard<mutex> lock(fileMutex);
    logFile << "Thread 1\n";
}

void thread2() {
    lock_guard<mutex> lock(fileMutex);
    logFile << "Thread 2\n";
}
```

### Better pattern: Message queue

```cpp
class ThreadSafeLogger {
    ofstream file;
    mutex mtx;
    condition_variable cv;
    queue<string> messages;
    thread writer;
    bool done = false;
    
public:
    ThreadSafeLogger(const string& filename) 
        : file(filename, ios::app) {
        writer = thread([this]() {
            while (true) {
                unique_lock<mutex> lock(mtx);
                cv.wait(lock, [this]() { 
                    return !messages.empty() || done; 
                });
                
                while (!messages.empty()) {
                    file << messages.front() << flush;
                    messages.pop();
                }
                
                if (done) break;
            }
        });
    }
    
    void log(const string& msg) {
        lock_guard<mutex> lock(mtx);
        messages.push(msg + "\n");
        cv.notify_one();
    }
    
    ~ThreadSafeLogger() {
        {
            lock_guard<mutex> lock(mtx);
            done = true;
        }
        cv.notify_one();
        writer.join();
    }
};
```

---

## Platform Differences

### Line endings
- **Linux/macOS**: `\n` (LF)
- **Windows**: `\r\n` (CRLF)
- **Old Mac**: `\r` (CR)

**Solution**: Always use binary mode for cross-platform files

### Path separators
```cpp
// ❌ Windows-only
string path = "data\\file.txt";

// ✅ Cross-platform
fs::path p = fs::path("data") / "file.txt";
// Or
string path = "data/file.txt";  // Works on Windows too!
```

### File permissions
- **POSIX (Linux/Mac)**: rwxrwxrwx (owner/group/other)
- **Windows**: ACLs (Access Control Lists)

Use `fs::permissions()` for cross-platform handling.

### Case sensitivity
- **Linux**: Case-sensitive (`file.txt` ≠ `File.txt`)
- **Windows**: Case-insensitive (`file.txt` = `File.txt`)
- **macOS**: Case-insensitive by default (configurable)

**Best practice**: Always use consistent casing

---

## Memory Management & RAII

### Why RAII matters

```cpp
// ❌ Manual management (error-prone)
ifstream* file = new ifstream("data.txt");
if (!file->is_open()) {
    delete file;  // Must remember!
    return;
}
// ... processing ...
delete file;  // Must remember!

// ✅ RAII (automatic)
{
    ifstream file("data.txt");
    if (!file.is_open()) return;
    // ... processing ...
}  // Automatically closed and destroyed
```

### Exception safety

```cpp
// ❌ Leak on exception
ifstream* file = new ifstream("data.txt");
processFile(*file);  // May throw
delete file;  // Never reached if exception!

// ✅ RAII guarantees cleanup
{
    ifstream file("data.txt");
    processFile(file);  // Even if throws, file closes
}

// ✅ Or use smart pointers
auto file = make_unique<ifstream>("data.txt");
processFile(*file);
// Automatically deleted even if exception
```

---

## Debugging File I/O

### 1. Check errno
```cpp
#include <cerrno>
#include <cstring>

ifstream file("data.txt");
if (!file) {
    cerr << "Error: " << strerror(errno) << endl;
    // ENOENT: No such file
    // EACCES: Permission denied
    // EMFILE: Too many open files
}
```

### 2. Verbose error checking
```cpp
void diagnoseFileError(const ifstream& file) {
    if (file.good()) {
        cout << "Stream is good\n";
    }
    if (file.eof()) {
        cout << "End of file reached\n";
    }
    if (file.fail()) {
        cout << "Logical error (failbit set)\n";
    }
    if (file.bad()) {
        cout << "Read/write error (badbit set)\n";
    }
}
```

### 3. Log file positions
```cpp
cout << "Read position: " << file.tellg() << endl;
cout << "Write position: " << file.tellp() << endl;
```

### 4. Hexdump utility
```cpp
void hexdump(const string& filename, size_t bytes = 256) {
    ifstream file(filename, ios::binary);
    char ch;
    size_t count = 0;
    
    while (file.get(ch) && count < bytes) {
        if (count % 16 == 0) cout << "\n" << hex << count << ": ";
        cout << setw(2) << setfill('0') << (int)(unsigned char)ch << " ";
        count++;
    }
    cout << dec << endl;
}
```

---

## Quick Reference by Use Case

### Reading entire file
```cpp
// Method 1: Line by line
ifstream file("data.txt");
string line;
while (getline(file, line)) {
    process(line);
}

// Method 2: All at once
ifstream file("data.txt");
stringstream buffer;
buffer << file.rdbuf();
string content = buffer.str();

// Method 3: Using iterators
ifstream file("data.txt");
string content((istreambuf_iterator<char>(file)),
               istreambuf_iterator<char>());
```

### Writing log file
```cpp
ofstream log("app.log", ios::app);
log << "[" << timestamp() << "] " << message << endl;
```

### Binary struct I/O
```cpp
// Write
ofstream out("data.bin", ios::binary);
out.write((char*)&obj, sizeof(obj));

// Read
ifstream in("data.bin", ios::binary);
in.read((char*)&obj, sizeof(obj));
```

### Random access
```cpp
fstream file("data.bin", ios::in | ios::out | ios::binary);
file.seekg(recordNum * sizeof(Record), ios::beg);
file.read((char*)&rec, sizeof(Record));
```

### File exists check
```cpp
// C++17
if (fs::exists("file.txt")) { /* ... */ }

// Pre-C++17
ifstream file("file.txt");
if (file.good()) { /* exists */ }
```

### Copy file
```cpp
// Method 1: Efficient
ifstream src("src.txt", ios::binary);
ofstream dst("dst.txt", ios::binary);
dst << src.rdbuf();

// Method 2: C++17
fs::copy_file("src.txt", "dst.txt");
```

---

## Advanced: Custom Stream Buffers

```cpp
// Custom streambuf that encrypts on write
class EncryptBuf : public streambuf {
    ofstream file;
    char buffer[1024];
    
protected:
    int overflow(int c) override {
        if (c != EOF) {
            char encrypted = c ^ 0xFF;  // Simple XOR
            file.put(encrypted);
        }
        return c;
    }
    
public:
    EncryptBuf(const string& filename) 
        : file(filename, ios::binary) {
        setp(buffer, buffer + sizeof(buffer));
    }
};

// Usage
EncryptBuf encBuf("encrypted.txt");
ostream encStream(&encBuf);
encStream << "Secret message";
```

---

## Critical Things to Remember

1. **Always check if file opened**: `if (!file.is_open())`
2. **Use binary mode for seeking**: Text mode positions are unreliable
3. **Clear error flags after errors**: `file.clear()`
4. **Use `ignore()` after `>>` before `getline()`**
5. **Flush critical data**: `file.flush()` or `endl`
6. **Check `gcount()` after `read()`**: May read less than requested
7. **Text mode converts newlines**: Use binary for exact bytes
8. **fstream needs explicit mode**: No defaults, specify `ios::in | ios::out`
9. **Buffers persist until flush**: Data may be lost on crash
10. **RAII handles cleanup**: Files auto-close when out of scope
11. **One error flag blocks all I/O**: Must clear before retrying
12. **Struct padding affects binary I/O**: Use `#pragma pack` or serialize manually
13. **Streams aren't thread-safe**: Use mutexes for concurrent access
14. **EOF sets both eofbit and failbit**: Check both in read loops
15. **`operator bool()` checks `!fail()`**: Not the same as `good()`

---

## When to Use What

| Scenario | Use | Why |
|----------|-----|-----|
| Read-only access | `ifstream` | Safety, clarity |
| Write-only access | `ofstream` | Safety, auto-truncate |
| Read + Write (random access) | `fstream` | Need seeking both ways |
| Appending logs | `ofstream(file, ios::app)` | Preserves existing content |
| Binary files | `ios::binary` mode | No transformations |
| Text config files | Text mode | Platform-appropriate newlines |
| Large files | Chunked reading | Memory efficient |
| Performance-critical | Binary + large buffer | Minimize conversions |
| Thread-safe logging | Mutex or queue pattern | Prevent corruption |
| Cross-platform | `fs::path` + binary mode | Consistent behavior |

---

## Conclusion

File handling in C++ provides:
- **Three specialized classes** (ifstream/ofstream/fstream) for clarity and safety
- **Complete control** over buffering, positioning, and formatting
- **Rich method set** inherited from stream hierarchy
- **RAII guarantees** automatic resource cleanup
- **Performance options** from buffering to memory-mapping

Master the core concepts:
- **Stream state** (goodbit/eofbit/failbit/badbit)
- **Buffer mechanics** (when data actually writes to disk)
- **Text vs binary** (transformations vs exact bytes)
- **Position management** (separate get/put pointers)
- **Error recovery** (clear flags, check operations)

The key is understanding **when to use which tool** and **why certain operations behave as they do**. With this knowledge, you can handle any file I/O scenario efficiently and correctly!
