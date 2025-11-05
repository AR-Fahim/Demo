# C++ File Handling Complete Guide

## Core Concepts

**File**: Persistent storage on disk. Data remains after program ends.

**Stream**: Flow of data between program and file. Think of it as a pipeline.

**Buffer**: Temporary memory holding data before writing to disk (improves performance).

---

## Three Main Classes

```cpp
#include <fstream>

ifstream  // Input File Stream (reading)
ofstream  // Output File Stream (writing)
fstream   // Both read and write
```

---

## Opening Files

### Method 1: Constructor
```cpp
ifstream inFile("data.txt");
ofstream outFile("output.txt");
fstream file("data.txt", ios::in | ios::out);
```

### Method 2: open() function
```cpp
ifstream inFile;
inFile.open("data.txt");
```

### File Modes (ios flags)

| Mode | Purpose | Details |
|------|---------|---------|
| `ios::in` | Read | File must exist |
| `ios::out` | Write | Creates/truncates file |
| `ios::app` | Append | Writes at end, creates if missing |
| `ios::ate` | At End | Opens and moves to end |
| `ios::trunc` | Truncate | Deletes existing content |
| `ios::binary` | Binary mode | No text transformations |

**Combine modes with `|` operator:**
```cpp
fstream file("data.txt", ios::in | ios::out | ios::binary);
```

---

## Checking File Status

```cpp
// Always check if file opened successfully
if (!inFile.is_open()) {
    cerr << "Error opening file!" << endl;
    return 1;
}

// Alternative
if (inFile.fail()) {
    // Handle error
}

// Check if file exists before opening
#include <fstream>
bool fileExists(const string& name) {
    ifstream f(name);
    return f.good();
}
```

---

## Closing Files

```cpp
inFile.close();  // Flushes buffer and releases file
```

**Note**: Files auto-close when object goes out of scope (RAII), but explicit closing is good practice.

---

## Reading Files

### 1. Character by Character
```cpp
char ch;
while (inFile.get(ch)) {  // Better than >> for whitespace
    cout << ch;
}
```

### 2. Word by Word
```cpp
string word;
while (inFile >> word) {  // Skips whitespace
    cout << word << endl;
}
```

### 3. Line by Line
```cpp
string line;
while (getline(inFile, line)) {  // Reads entire line
    cout << line << endl;
}

// With custom delimiter
while (getline(inFile, line, ';')) {  // Reads until semicolon
    cout << line << endl;
}
```

### 4. Fixed-size block
```cpp
char buffer[100];
inFile.read(buffer, 100);  // Reads 100 bytes
int bytesRead = inFile.gcount();  // Actual bytes read
```

### 5. Entire file at once
```cpp
ifstream inFile("data.txt");
stringstream buffer;
buffer << inFile.rdbuf();  // Read entire file
string content = buffer.str();
```

---

## Writing Files

### 1. Basic writing
```cpp
ofstream outFile("output.txt");
outFile << "Hello World" << endl;
outFile << 42 << " " << 3.14 << endl;
```

### 2. Writing binary data
```cpp
int num = 100;
outFile.write((char*)&num, sizeof(num));
```

### 3. Formatted output
```cpp
#include <iomanip>
outFile << setw(10) << setfill('0') << 42;  // Output: 0000000042
outFile << fixed << setprecision(2) << 3.14159;  // Output: 3.14
```

---

## File Pointers (Navigation)

Two pointers: **get pointer** (read) and **put pointer** (write)

### tellg() / tellp() - Get current position
```cpp
streampos pos = inFile.tellg();  // Get position (reading)
streampos pos = outFile.tellp(); // Put position (writing)
```

### seekg() / seekp() - Move position
```cpp
// From beginning
inFile.seekg(10, ios::beg);  // Move to byte 10

// From current position
inFile.seekg(5, ios::cur);   // Move 5 bytes forward

// From end
inFile.seekg(-10, ios::end); // 10 bytes before end

// Absolute position
inFile.seekg(0, ios::beg);   // Move to start
inFile.seekg(0, ios::end);   // Move to end
```

### Getting file size
```cpp
inFile.seekg(0, ios::end);
streampos size = inFile.tellg();
inFile.seekg(0, ios::beg);  // Reset to beginning
```

---

## Complete Methods Reference

### Stream State Functions
```cpp
bool good()      // No errors
bool eof()       // End of file reached
bool fail()      // Operation failed
bool bad()       // Read/write error
void clear()     // Reset error flags
```

### Input Methods (ifstream/fstream)
```cpp
get(char& ch)                    // Read single character
get(char* buffer, int n)         // Read n-1 chars or until newline
getline(char* buffer, int n)     // Read line (n-1 chars max)
read(char* buffer, int n)        // Read n bytes (binary)
gcount()                         // Get number of chars from last read
peek()                           // Look at next char without extracting
putback(char ch)                 // Put character back
unget()                          // Put last read char back
ignore(int n = 1, int delim = EOF) // Skip n chars or until delim
```

### Output Methods (ofstream/fstream)
```cpp
put(char ch)                     // Write single character
write(const char* buffer, int n) // Write n bytes (binary)
flush()                          // Force buffer write to disk
```

### File Operations
```cpp
open(filename, mode)             // Open file
close()                          // Close file
is_open()                        // Check if file is open
```

### Position Methods
```cpp
tellg()                          // Get read position
tellp()                          // Get write position
seekg(pos)                       // Set read position (absolute)
seekg(offset, direction)         // Set read position (relative)
seekp(pos)                       // Set write position (absolute)
seekp(offset, direction)         // Set write position (relative)
```

---

## Practical Examples

### Example 1: Copy file
```cpp
void copyFile(const string& source, const string& dest) {
    ifstream src(source, ios::binary);
    ofstream dst(dest, ios::binary);
    
    if (!src || !dst) {
        cerr << "Error opening files!" << endl;
        return;
    }
    
    dst << src.rdbuf();  // Efficient copy
}
```

### Example 2: Count lines, words, characters
```cpp
void analyzeFile(const string& filename) {
    ifstream file(filename);
    int lines = 0, words = 0, chars = 0;
    string line, word;
    
    while (getline(file, line)) {
        lines++;
        chars += line.length();
        istringstream iss(line);
        while (iss >> word) words++;
    }
    
    cout << "Lines: " << lines << ", Words: " << words 
         << ", Chars: " << chars << endl;
}
```

### Example 3: CSV file handling
```cpp
vector<vector<string>> readCSV(const string& filename) {
    ifstream file(filename);
    vector<vector<string>> data;
    string line, cell;
    
    while (getline(file, line)) {
        vector<string> row;
        istringstream lineStream(line);
        while (getline(lineStream, cell, ',')) {
            row.push_back(cell);
        }
        data.push_back(row);
    }
    return data;
}
```

### Example 4: Binary file (struct)
```cpp
struct Student {
    char name[50];
    int age;
    float gpa;
};

// Writing
ofstream out("students.dat", ios::binary);
Student s = {"Alice", 20, 3.8};
out.write((char*)&s, sizeof(Student));

// Reading
ifstream in("students.dat", ios::binary);
Student s;
in.read((char*)&s, sizeof(Student));
```

### Example 5: Random access file
```cpp
void updateRecord(const string& filename, int recordNum, const Student& s) {
    fstream file(filename, ios::in | ios::out | ios::binary);
    
    // Move to specific record
    file.seekp(recordNum * sizeof(Student), ios::beg);
    file.write((char*)&s, sizeof(Student));
}
```

### Example 6: Append to file
```cpp
void logMessage(const string& message) {
    ofstream logFile("app.log", ios::app);
    time_t now = time(0);
    logFile << ctime(&now) << ": " << message << endl;
}
```

---

## Text vs Binary Mode

### Text Mode (default)
- Platform-specific newline conversion (`\n` ↔ `\r\n` on Windows)
- Can't handle null bytes reliably
- Good for human-readable files

### Binary Mode (`ios::binary`)
- No transformations
- Exact byte-for-byte operations
- Required for non-text files (images, executables)
- **Always use for struct/class serialization**

```cpp
// Text mode
ofstream textFile("data.txt");
textFile << "Hello\n";  // May write "Hello\r\n" on Windows

// Binary mode
ofstream binFile("data.bin", ios::binary);
binFile.write("Hello\n", 6);  // Writes exactly "Hello\n"
```

---

## Error Handling Patterns

### Pattern 1: Basic check
```cpp
ifstream file("data.txt");
if (!file) {
    cerr << "Cannot open file!" << endl;
    return;
}
```

### Pattern 2: Detailed error checking
```cpp
ifstream file("data.txt");
if (file.fail()) {
    if (errno == ENOENT) cerr << "File not found" << endl;
    else if (errno == EACCES) cerr << "Permission denied" << endl;
    else cerr << "Unknown error: " << errno << endl;
}
```

### Pattern 3: RAII wrapper
```cpp
class FileGuard {
    fstream file;
public:
    FileGuard(const string& name, ios::openmode mode) 
        : file(name, mode) {
        if (!file) throw runtime_error("Cannot open file");
    }
    ~FileGuard() { if (file.is_open()) file.close(); }
    fstream& get() { return file; }
};

// Usage
try {
    FileGuard fg("data.txt", ios::in);
    // Use fg.get() ...
} catch (const exception& e) {
    cerr << e.what() << endl;
}
```

---

## Common Pitfalls & Solutions

### 1. Forgetting to check file status
```cpp
// ❌ Bad
ifstream file("data.txt");
file >> data;  // Might fail silently

// ✅ Good
ifstream file("data.txt");
if (!file) { /* handle error */ }
file >> data;
```

### 2. Not clearing error flags
```cpp
// After EOF or error, stream is in bad state
file.clear();  // Reset flags before reusing
file.seekg(0, ios::beg);  // Rewind
```

### 3. Mixing >> and getline()
```cpp
int num;
string line;
file >> num;        // Leaves '\n' in buffer
getline(file, line); // Reads empty line!

// Fix: Use ignore()
file >> num;
file.ignore(numeric_limits<streamsize>::max(), '\n');
getline(file, line);
```

### 4. Buffer not flushed
```cpp
outFile << "Important data";
// Program crashes before close() → data lost!

// Fix: Flush explicitly
outFile << "Important data" << flush;
// Or use endl (writes '\n' + flush)
outFile << "Important data" << endl;
```

### 5. Wrong binary mode
```cpp
// ❌ Reading struct in text mode
ifstream file("data.dat");  // Missing ios::binary
Student s;
file.read((char*)&s, sizeof(s));  // Corrupted data!

// ✅ Use binary mode
ifstream file("data.dat", ios::binary);
```

### 6. Not checking read success
```cpp
// ❌ Assuming read succeeded
file.read(buffer, 100);
processData(buffer, 100);  // Might use garbage data

// ✅ Check gcount()
file.read(buffer, 100);
int bytesRead = file.gcount();
processData(buffer, bytesRead);
```

---

## Performance Tips

### 1. Use buffering wisely
```cpp
// Set custom buffer size (8KB)
char buffer[8192];
inFile.rdbuf()->pubsetbuf(buffer, sizeof(buffer));
```

### 2. Read in chunks (faster than char-by-char)
```cpp
// ❌ Slow
while (file.get(ch)) { /* process */ }

// ✅ Fast
char buffer[4096];
while (file.read(buffer, sizeof(buffer)) || file.gcount() > 0) {
    process(buffer, file.gcount());
}
```

### 3. Use rdbuf() for file copying
```cpp
// Fastest way to copy
dst << src.rdbuf();
```

### 4. Avoid frequent flush()
```cpp
// ❌ Slow (flushes 1000 times)
for (int i = 0; i < 1000; i++)
    file << i << flush;

// ✅ Fast (flushes once)
for (int i = 0; i < 1000; i++)
    file << i << '\n';
file.flush();
```

### 5. Use binary mode for large data
```cpp
// Binary is faster (no conversions)
fstream file("big.dat", ios::binary);
```

---

## Working with Different File Types

### JSON (manual parsing)
```cpp
#include <nlohmann/json.hpp>  // Popular library
ifstream file("data.json");
json j = json::parse(file);
string name = j["name"];
```

### INI/Config files
```cpp
map<string, string> readConfig(const string& file) {
    ifstream in(file);
    map<string, string> config;
    string line;
    while (getline(in, line)) {
        size_t pos = line.find('=');
        if (pos != string::npos) {
            string key = line.substr(0, pos);
            string val = line.substr(pos + 1);
            config[key] = val;
        }
    }
    return config;
}
```

### Large files (chunked processing)
```cpp
void processLargeFile(const string& filename) {
    ifstream file(filename, ios::binary);
    const size_t CHUNK_SIZE = 1024 * 1024;  // 1MB
    char* buffer = new char[CHUNK_SIZE];
    
    while (file.read(buffer, CHUNK_SIZE) || file.gcount() > 0) {
        processChunk(buffer, file.gcount());
    }
    delete[] buffer;
}
```

---

## Advanced Techniques

### 1. Memory-mapped files (C++17)
```cpp
#include <fstream>
#include <sys/mman.h>  // POSIX only
// Faster for random access, but platform-specific
```

### 2. Asynchronous I/O
```cpp
#include <future>
auto future = async(launch::async, [](){ 
    return readFile("data.txt"); 
});
// Do other work...
string data = future.get();
```

### 3. File locking (prevent concurrent access)
```cpp
#include <fcntl.h>  // POSIX
// Platform-specific, prevents race conditions
```

### 4. Temporary files
```cpp
#include <cstdio>
char tmp[L_tmpnam];
tmpnam(tmp);  // Generate temp filename
ofstream tempFile(tmp);
// ... use file ...
remove(tmp);  // Delete when done
```

---

## File System Operations (C++17)

```cpp
#include <filesystem>
namespace fs = std::filesystem;

// Check if file exists
if (fs::exists("data.txt")) { /* ... */ }

// Get file size
uintmax_t size = fs::file_size("data.txt");

// Copy/move/delete
fs::copy("src.txt", "dst.txt");
fs::rename("old.txt", "new.txt");
fs::remove("file.txt");

// Iterate directory
for (auto& entry : fs::directory_iterator(".")) {
    cout << entry.path() << endl;
}

// Create directories
fs::create_directory("mydir");
fs::create_directories("path/to/nested/dir");

// Get file info
auto time = fs::last_write_time("data.txt");
auto perms = fs::status("data.txt").permissions();
```

---

## Quick Reference Cheatsheet

```cpp
// OPEN
ifstream in("r.txt");                    // Read
ofstream out("w.txt");                   // Write (truncate)
ofstream out("a.txt", ios::app);         // Append
fstream f("rw.txt", ios::in|ios::out);   // Read+Write

// CHECK
if (!file) { /* error */ }
if (file.is_open()) { /* ok */ }

// READ
char ch; file.get(ch);                   // One char
string s; file >> s;                     // One word
string line; getline(file, line);        // One line
file.read(buf, n);                       // n bytes

// WRITE
file << "text" << 42 << endl;            // Formatted
file.put('A');                           // One char
file.write(buf, n);                      // n bytes

// POSITION
file.seekg(0, ios::beg);                 // Start
file.seekg(0, ios::end);                 // End
file.seekg(10, ios::cur);                // +10 from current
streampos pos = file.tellg();            // Get position

// CLOSE
file.close();
```

---

## Things to Remember

1. **Always check if file opened successfully** before operations
2. **Use binary mode** for non-text files and structs
3. **Clear error flags** with `clear()` after errors/EOF
4. **Use `ignore()`** after `>>` before `getline()`
5. **Flush critical data** with `flush()` or `endl`
6. **Check `gcount()`** after `read()` for actual bytes read
7. **Close files explicitly** in long-running programs
8. **Use `const string&`** for filename parameters (not `char*`)
9. **Text mode converts newlines**, binary mode doesn't
10. **RAII ensures cleanup** - objects auto-close when out of scope

---

## Security Considerations

```cpp
// 1. Validate filenames (prevent path traversal)
if (filename.find("..") != string::npos) {
    // Reject - potential attack
}

// 2. Limit file size when reading
const size_t MAX_SIZE = 10 * 1024 * 1024;  // 10MB
if (fs::file_size(filename) > MAX_SIZE) {
    // Reject - too large
}

// 3. Use exceptions for critical errors
try {
    ifstream file(filename);
    if (!file) throw runtime_error("Cannot open");
    // ...
} catch (const exception& e) {
    logError(e.what());
}

// 4. Don't trust external data
// Always validate before parsing
```

---

## Conclusion

File handling in C++ is powerful but requires attention to:
- **Error checking** (always verify operations)
- **Mode selection** (text vs binary)
- **Resource management** (close files, RAII)
- **Performance** (buffering, chunking)
- **Platform differences** (newlines, paths)

Master these concepts, and you'll handle any file operation confidently!
