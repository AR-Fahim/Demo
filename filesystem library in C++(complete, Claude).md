# C++ Filesystem Library: Complete 

## 1. Core Philosophy & Design

**Why It Exists**: Before C++17, filesystem operations were OS-specific (POSIX vs Windows API). The `<filesystem>` library abstracts these differences into a portable interface.

**Mental Model**: Think of it as three layers:
1. **Path representation** - how to describe locations
2. **Query operations** - inspecting without modification
3. **Modification operations** - changing the filesystem

**Header & Namespace**:
```cpp
#include <filesystem>
namespace fs = std::filesystem; // Common alias
```

---

## 2. Path: The Foundation

### Conceptual Understanding
A `path` is NOT just a string—it's a parsed, platform-aware representation. Internally:
- Stores components (directories, filename, extension)
- Handles platform separators automatically (`/` vs `\`)
- Manages relative vs absolute resolution

### Construction
```cpp
fs::path p1("dir/file.txt");           // From string
fs::path p2 = "dir" / "file.txt";      // Operator overload (portable)
fs::path p3(L"unicode/path");          // Wide string support
fs::path p4 = fs::current_path();      // From function
```

**Key Insight**: Use `/` operator for portability—it handles OS differences automatically.

### Path Decomposition
```cpp
fs::path p = "/home/user/docs/report.tar.gz";

p.root_name()         // "" (Linux) or "C:" (Windows)
p.root_directory()    // "/"
p.root_path()         // "/" (combines name + directory)
p.relative_path()     // "home/user/docs/report.tar.gz"
p.parent_path()       // "/home/user/docs"
p.filename()          // "report.tar.gz"
p.stem()              // "report.tar"
p.extension()         // ".gz"
```

**Critical Edge Case**: Extensions are tricky
```cpp
fs::path p1("file.tar.gz");
p1.extension();  // ".gz" (only last!)
p1.stem();       // "file.tar"

// To get full extension manually:
std::string fullExt;
auto temp = p1;
while (temp.has_extension()) {
    fullExt = temp.extension().string() + fullExt;
    temp = temp.stem();
}
// fullExt = ".tar.gz"
```

### Path Manipulation
```cpp
fs::path p = "dir/file.txt";

p.replace_filename("new.txt");    // "dir/new.txt"
p.replace_extension(".md");       // "dir/new.md"
p.remove_filename();              // "dir/"
p.clear();                        // Empty path

// Concatenation
fs::path base = "dir";
base /= "subdir";                 // "dir/subdir" (adds separator)
base += ".txt";                   // "dir/subdir.txt" (no separator)
```

**Important**: `/=` adds separator, `+=` doesn't.

### Path Conversion
```cpp
fs::path p = "dir/file.txt";

p.string()          // Returns std::string
p.wstring()         // Returns std::wstring
p.u8string()        // Returns std::u8string (UTF-8)
p.generic_string()  // Always uses '/' (portable output)
p.native()          // Platform-specific native format
p.c_str()           // C-style string
```

**Use Case**: Use `.generic_string()` for logging/config files to ensure portability.

### Path Comparison
```cpp
fs::path p1 = "dir/file.txt";
fs::path p2 = "DIR/FILE.TXT";

p1 == p2;  // false (case-sensitive on most systems)
p1 < p2;   // Lexicographical comparison

// Case-insensitive comparison (manual)
auto compare_ci = [](const fs::path& a, const fs::path& b) {
    return fs::equivalent(fs::absolute(a), fs::absolute(b));
};
```

**Platform Behavior**: Windows paths are case-insensitive (NTFS), Linux is case-sensitive (ext4).

### Normalization & Resolution
```cpp
fs::path p = "/home/user/../user/./docs/../docs/file.txt";

p.lexically_normal();    // "/home/user/docs/file.txt" (doesn't touch filesystem)
fs::canonical(p);        // Resolves symlinks + normalizes (requires path exists)
fs::weakly_canonical(p); // Normalizes existing parts, keeps non-existing
fs::absolute(p);         // Makes absolute (prepends current_path if relative)
fs::relative(p, base);   // Finds relative path from base to p
fs::proximate(p, base);  // Like relative, but returns p if no relative path exists
```

**Critical Difference**:
- `lexically_normal()` - string manipulation only
- `canonical()` - filesystem access, resolves symlinks, **throws if path doesn't exist**
- `weakly_canonical()` - best of both (resolves what exists, normalizes rest)

---

## 3. Directory Iteration

### Basic Iteration
```cpp
for (const auto& entry : fs::directory_iterator("path")) {
    std::cout << entry.path() << '\n';
}
```

**Behavior**: 
- Non-recursive
- Order is **undefined** (filesystem-dependent)
- Skips `.` and `..` automatically

### Recursive Iteration
```cpp
for (const auto& entry : fs::recursive_directory_iterator("path")) {
    std::cout << entry.path() << '\n';
}
```

**Control Options**:
```cpp
fs::directory_options opts = 
    fs::directory_options::skip_permission_denied |
    fs::directory_options::follow_directory_symlink;

for (const auto& entry : fs::recursive_directory_iterator("path", opts)) {
    // Process entry
}
```

**Options**:
- `none` - Default (throw on permission denied)
- `follow_directory_symlink` - Follow symlinks to directories
- `skip_permission_denied` - Skip inaccessible directories instead of throwing

### Directory Entry Optimization

**Key Insight**: `directory_entry` caches attributes to avoid repeated syscalls.

```cpp
for (const auto& entry : fs::directory_iterator("path")) {
    // Efficient (cached)
    if (entry.is_regular_file() && entry.file_size() > 1024) {
        // Process
    }
}

// vs Inefficient
for (const auto& entry : fs::directory_iterator("path")) {
    fs::path p = entry.path();
    if (fs::is_regular_file(p) && fs::file_size(p) > 1024) {
        // Each free function call = new syscall!
    }
}
```

**Available Methods**:
```cpp
entry.exists();              // Cached
entry.is_regular_file();     // Cached
entry.is_directory();        // Cached
entry.is_symlink();          // Cached
entry.file_size();           // Cached
entry.last_write_time();     // Cached
entry.status();              // Gets full status
entry.refresh();             // Updates cache
```

### Iteration Depth Control
```cpp
fs::recursive_directory_iterator it("path");
for (auto& entry : it) {
    if (entry.path().filename() == "skip") {
        it.disable_recursion_pending(); // Don't enter this directory
    }
    
    std::cout << "Depth: " << it.depth() << " - " << entry.path() << '\n';
    
    if (it.depth() >= 3) {
        it.pop(); // Go back up one level
    }
}
```

**Real-World Use Case**: Implementing `.gitignore` logic while traversing.

---

## 4. File Status & Attributes

### File Types
```cpp
enum class file_type {
    none,       // Unknown/error
    not_found,  // Doesn't exist
    regular,    // Regular file
    directory,  // Directory
    symlink,    // Symbolic link
    block,      // Block device
    character,  // Character device
    fifo,       // Named pipe
    socket,     // Socket
    unknown     // Exists but type unknown
};
```

### Status Inspection
```cpp
fs::file_status s = fs::status("path");
s.type();          // file_type enum
s.permissions();   // perms enum

// Convenience functions
fs::is_regular_file("path");
fs::is_directory("path");
fs::is_symlink("path");       // Only checks if it's a symlink
fs::is_empty("path");         // Empty file or empty directory
fs::is_block_file("path");
fs::is_character_file("path");
fs::is_fifo("path");
fs::is_socket("path");
fs::is_other("path");         // Block, char, fifo, socket, or unknown
```

**Critical Distinction**:
```cpp
fs::status("symlink");          // Follows symlink (gets target status)
fs::symlink_status("symlink");  // Gets symlink status itself
```

### Permissions
```cpp
enum class perms {
    none         = 0,
    owner_read   = 0400,
    owner_write  = 0200,
    owner_exec   = 0100,
    owner_all    = 0700,
    
    group_read   = 040,
    group_write  = 020,
    group_exec   = 010,
    group_all    = 070,
    
    others_read  = 04,
    others_write = 02,
    others_exec  = 01,
    others_all   = 07,
    
    all          = 0777,
    
    set_uid      = 04000,
    set_gid      = 02000,
    sticky_bit   = 01000,
    
    mask         = 07777,
    unknown      = 0xFFFF
};

// Bitwise operations
perms p = perms::owner_read | perms::owner_write;

// Checking permissions
if ((s.permissions() & perms::owner_write) != perms::none) {
    // Owner has write permission
}
```

### Modifying Permissions
```cpp
enum class perm_options {
    replace,    // Set exactly these permissions
    add,        // Add these permissions
    remove,     // Remove these permissions
    nofollow    // Don't follow symlinks
};

fs::permissions("file.txt", 
    perms::owner_all | perms::group_read, 
    perm_options::replace);

fs::permissions("file.txt",
    perms::others_write,
    perm_options::remove);
```

**Windows Note**: Windows permissions are simplified. Most `perms` values map to read-only/not read-only.

### File Size & Space
```cpp
std::uintmax_t size = fs::file_size("file.txt");  // Bytes
// Throws if not a regular file or symlink to regular file

fs::space_info space = fs::space("/mount/point");
space.capacity;    // Total bytes
space.free;        // Free bytes (including privileged)
space.available;   // Available to non-privileged user

// Safe size checking
auto safe_size = [](const fs::path& p) -> std::uintmax_t {
    std::error_code ec;
    auto size = fs::file_size(p, ec);
    return ec ? 0 : size;
};
```

### File Times
```cpp
using file_time = fs::file_time_type;  // std::chrono based

file_time ftime = fs::last_write_time("file.txt");

// Modify
fs::last_write_time("file.txt", ftime);

// Convert to system time
auto sctp = std::chrono::time_point_cast<std::chrono::system_clock::duration>(
    ftime - file_time::clock::now() + std::chrono::system_clock::now()
);
std::time_t cftime = std::chrono::system_clock::to_time_t(sctp);
```

**Platform Differences**:
- Unix: Access time, modification time, change time (ctime)
- Windows: Creation time, last access, last write
- C++17 only exposes modification time portably

---

## 5. Filesystem Modifications

### Creating Directories
```cpp
bool created = fs::create_directory("newdir");       // false if exists
bool created = fs::create_directories("a/b/c");      // Creates full path, false if exists

// With error handling
std::error_code ec;
fs::create_directories("path", ec);
if (ec) {
    std::cerr << "Error: " << ec.message() << '\n';
}
```

**Atomicity**: `create_directory` is atomic. `create_directories` is not—if it fails midway, some directories remain.

### Copying
```cpp
enum class copy_options {
    none,
    skip_existing,           // Don't overwrite
    overwrite_existing,      // Overwrite files
    update_existing,         // Overwrite if source is newer
    
    recursive,               // Copy directories recursively
    copy_symlinks,           // Copy symlinks as symlinks
    skip_symlinks,           // Skip symlinks
    
    directories_only,        // Copy directory structure only
    create_symlinks,         // Create symlinks instead of copying
    create_hard_links        // Create hard links instead of copying
};

fs::copy("source", "dest", fs::copy_options::recursive);

// Fine-grained control
fs::copy_file("src.txt", "dst.txt", 
    fs::copy_options::overwrite_existing);

fs::copy_symlink("link_src", "link_dst");
```

**Edge Cases**:
```cpp
// Copying directory to existing directory
fs::create_directories("dst/subdir");
fs::copy("src", "dst", fs::copy_options::recursive);
// Result: Contents merged (files from src added to dst)

// Copying file to directory
fs::copy("file.txt", "existing_dir/");  // Error! Must specify filename
fs::copy("file.txt", "existing_dir/file.txt");  // Correct
```

### Moving/Renaming
```cpp
fs::rename("old.txt", "new.txt");

// Atomic on same filesystem
// May fail across different filesystems (mount points)
// If dest exists: behavior is platform-dependent
```

**Cross-Filesystem Move**:
```cpp
void safe_move(const fs::path& src, const fs::path& dst) {
    std::error_code ec;
    fs::rename(src, dst, ec);
    if (ec) {
        // Failed (likely cross-filesystem), fallback to copy+delete
        fs::copy(src, dst, fs::copy_options::recursive, ec);
        if (!ec) {
            fs::remove_all(src);
        }
    }
}
```

### Removing
```cpp
bool removed = fs::remove("file.txt");       // false if doesn't exist
std::uintmax_t count = fs::remove_all("dir"); // Returns number of removed items

// Safe removal
std::error_code ec;
fs::remove_all("path", ec);
if (ec) {
    // Handle error (permission denied, in use, etc.)
}
```

**Race Condition Warning**:
```cpp
// WRONG: Race condition
if (fs::exists("file.txt")) {
    fs::remove("file.txt");  // File might be deleted between check and remove
}

// RIGHT: Just try to remove
std::error_code ec;
fs::remove("file.txt", ec);
// Check ec if you care about errors
```

### Symlinks & Hard Links
```cpp
fs::create_symlink("target.txt", "link.txt");          // File symlink
fs::create_directory_symlink("target_dir", "link_dir"); // Directory symlink
fs::create_hard_link("target.txt", "hard_link.txt");

fs::path target = fs::read_symlink("link.txt");

// Resolving symlinks
fs::path resolved = fs::canonical("link.txt");  // Gets final target
```

**Hard Link Restrictions**:
- Must be on same filesystem
- Cannot link directories (on most systems)
- Links share inode (deleting one doesn't affect others until all are gone)

**Symlink Portability**:
- Unix: Full support
- Windows: Requires admin privileges (or Developer Mode in Win10+)

---

## 6. Error Handling Strategies

### Two Paradigms
```cpp
// 1. Exception-based (default)
try {
    fs::remove("file.txt");
} catch (const fs::filesystem_error& e) {
    std::cerr << "Error: " << e.what() << '\n';
    std::cerr << "Path1: " << e.path1() << '\n';
    std::cerr << "Path2: " << e.path2() << '\n';  // For two-path operations
    std::cerr << "Code: " << e.code() << '\n';
}

// 2. Error code (no exceptions)
std::error_code ec;
fs::remove("file.txt", ec);
if (ec) {
    std::cerr << "Error: " << ec.message() << '\n';
}
```

**Best Practice**: Use error codes in production for:
- Performance-critical code
- Codebases that disable exceptions
- Expected failures (e.g., checking if file exists by trying to open it)

### Common Error Conditions
```cpp
ec == std::errc::no_such_file_or_directory   // File not found
ec == std::errc::permission_denied            // Access denied
ec == std::errc::file_exists                  // File already exists
ec == std::errc::not_a_directory              // Expected directory
ec == std::errc::is_a_directory               // Expected file
ec == std::errc::directory_not_empty          // Can't remove non-empty dir
ec == std::errc::cross_device_link            // Rename across filesystems
ec == std::errc::filename_too_long            // Path exceeds system limit
```

---

## 7. Advanced Patterns & Real-World Usage

### Safe File Replacement (Atomic Write)
```cpp
void safe_write(const fs::path& target, const std::string& data) {
    fs::path temp = target;
    temp += ".tmp";
    
    std::ofstream(temp) << data;  // Write to temp
    
    fs::rename(temp, target);  // Atomic on same filesystem
    // If rename fails, original file is untouched
}
```

### Recursive File Search
```cpp
std::vector<fs::path> find_files(const fs::path& root, const std::string& pattern) {
    std::vector<fs::path> results;
    std::error_code ec;
    
    for (const auto& entry : fs::recursive_directory_iterator(root, 
        fs::directory_options::skip_permission_denied, ec)) {
        
        if (entry.path().filename().string().find(pattern) != std::string::npos) {
            results.push_back(entry.path());
        }
    }
    
    return results;
}
```

### Directory Size Calculation
```cpp
std::uintmax_t directory_size(const fs::path& dir) {
    std::uintmax_t size = 0;
    std::error_code ec;
    
    for (const auto& entry : fs::recursive_directory_iterator(dir,
        fs::directory_options::skip_permission_denied, ec)) {
        
        if (entry.is_regular_file(ec)) {
            size += entry.file_size(ec);
        }
    }
    
    return size;
}
```

### Temporary File Creation
```cpp
fs::path create_temp_file() {
    auto temp_dir = fs::temp_directory_path();
    fs::path temp_file;
    
    do {
        std::string random_name = "app_" + std::to_string(std::rand()) + ".tmp";
        temp_file = temp_dir / random_name;
    } while (fs::exists(temp_file));
    
    std::ofstream(temp_file);  // Create empty file
    return temp_file;
}

// RAII cleanup
class TempFile {
    fs::path path_;
public:
    TempFile() : path_(create_temp_file()) {}
    ~TempFile() { std::error_code ec; fs::remove(path_, ec); }
    
    const fs::path& path() const { return path_; }
    
    TempFile(const TempFile&) = delete;
    TempFile& operator=(const TempFile&) = delete;
};
```

### Directory Watcher Pattern (Manual)
```cpp
// C++ filesystem doesn't have built-in watching
// Manual polling approach:

struct FileSnapshot {
    std::map<fs::path, fs::file_time_type> files;
};

FileSnapshot snapshot_directory(const fs::path& dir) {
    FileSnapshot snap;
    for (const auto& entry : fs::recursive_directory_iterator(dir)) {
        if (entry.is_regular_file()) {
            snap.files[entry.path()] = fs::last_write_time(entry);
        }
    }
    return snap;
}

void check_changes(const FileSnapshot& before, const FileSnapshot& after) {
    // New/modified files
    for (const auto& [path, time] : after.files) {
        auto it = before.files.find(path);
        if (it == before.files.end()) {
            std::cout << "Added: " << path << '\n';
        } else if (it->second != time) {
            std::cout << "Modified: " << path << '\n';
        }
    }
    
    // Deleted files
    for (const auto& [path, time] : before.files) {
        if (after.files.find(path) == after.files.end()) {
            std::cout << "Deleted: " << path << '\n';
        }
    }
}
```

### Safe Recursion with Cycle Detection
```cpp
void traverse_safe(const fs::path& path, std::set<fs::path>& visited) {
    fs::path canonical_path;
    try {
        canonical_path = fs::canonical(path);
    } catch (...) {
        return;  // Invalid path
    }
    
    if (!visited.insert(canonical_path).second) {
        return;  // Already visited (cycle detected)
    }
    
    for (const auto& entry : fs::directory_iterator(path)) {
        if (entry.is_directory()) {
            traverse_safe(entry.path(), visited);
        }
    }
}
```

---

## 8. Performance Considerations

### Syscall Overhead
```cpp
// BAD: Multiple syscalls
for (const auto& entry : fs::directory_iterator("dir")) {
    if (fs::is_regular_file(entry.path())) {           // Syscall
        auto size = fs::file_size(entry.path());       // Syscall
        auto time = fs::last_write_time(entry.path()); // Syscall
    }
}

// GOOD: Use cached directory_entry
for (const auto& entry : fs::directory_iterator("dir")) {
    if (entry.is_regular_file()) {    // Cached
        auto size = entry.file_size(); // Cached
        auto time = entry.last_write_time(); // Cached
    }
}
```

### Batch Operations
```cpp
// When processing many files, minimize exception handling overhead
std::error_code ec;
for (const auto& entry : fs::directory_iterator("dir", ec)) {
    // Process with ec versions
}
```

### Avoiding Redundant Canonicalization
```cpp
// BAD: Canonicalize inside loop
for (const auto& path : paths) {
    auto canonical = fs::canonical(path);  // Expensive
    process(canonical);
}

// GOOD: Canonicalize once if base is same
auto base = fs::canonical("common/base");
for (const auto& path : paths) {
    auto full = base / path;
    process(full);
}
```

---

## 9. Platform-Specific Gotchas

### Path Length Limits
- **Windows**: 260 characters (MAX_PATH) by default
  - Enable long paths: Registry + manifest + Windows 10+
  - UNC paths (`\\?\C:\...`) bypass limit
- **Linux**: 4096 characters (PATH_MAX)

### Path Separators
```cpp
// Portable separator access
char sep = fs::path::preferred_separator;  // '/' on Unix, '\' on Windows

// Always use operator/ for portability
fs::path p = "dir" / "subdir" / "file.txt";  // Works everywhere
```

### Case Sensitivity
```cpp
// Windows: "File.txt" == "file.txt"
// Linux: "File.txt" != "file.txt"

// Portable case-insensitive comparison
bool equal_ci(const fs::path& a, const fs::path& b) {
    std::error_code ec;
    return fs::equivalent(a, b, ec);  // Checks if same inode/filesystem object
}
```

### Symlink Permissions
- **Unix**: Full support, permissions on symlink vs target
- **Windows**: Requires privileges, limited support

### Unicode
```cpp
// Windows: Native UTF-16
// Unix: Usually UTF-8

// Safe handling
fs::path p = fs::u8path(u8"unicode_文件.txt");  // C++17
// C++20: fs::path p{u8"unicode_文件.txt"};
```

---

## 10. Common Pitfalls & Solutions

### Pitfall 1: Checking Existence Before Operations
```cpp
// WRONG: Race condition
if (fs::exists("file.txt")) {
    fs::remove("file.txt");  // Might fail if file deleted between check and remove
}

// RIGHT: Try the operation
std::error_code ec;
fs::remove("file.txt", ec);
if (ec && ec != std::errc::no_such_file_or_directory) {
    // Handle real error
}
```

### Pitfall 2: Exception in Destructor
```cpp
// WRONG: May throw in destructor
~TempFile() {
    fs::remove(path_);  // Throws on error
}

// RIGHT: Suppress exceptions
~TempFile() {
    std::error_code ec;
    fs::remove(path_, ec);
}
```

### Pitfall 3: Forgetting to Follow Symlinks
```cpp
// Be explicit about symlink handling
auto status = fs::status("path");          // Follows symlinks
auto symlink_status = fs::symlink_status("path");  // Doesn't follow

if (fs::is_symlink(symlink_status)) {
    auto target = fs::read_symlink("path");
    // Process target
}
```

### Pitfall 4: Assuming Directory Order
```cpp
// WRONG: Assuming alphabetical order
for (const auto& entry : fs::directory_iterator("dir")) {
    // Order is undefined!
}

// RIGHT: Sort if needed
std::vector<fs::path> paths;
for (const auto& entry : fs::directory_iterator("dir")) {
    paths.push_back(entry.path());
}
std::sort(paths.begin(), paths.end());
```

### Pitfall 5: Recursive Removal Without Confirmation
```cpp
// DANGEROUS: No confirmation
fs::remove_all(user_input);  // Never do this directly!

// SAFE: Validate and confirm
auto path = fs::canonical(user_input);  // Resolve symlinks
if (path.string().find("/safe/directory/") == 0) {  // Whitelist check
    std::cout << "Remove " << path << "? (y/n): ";
    char confirm;
    std::cin >> confirm;
    if (confirm == 'y') {
        fs::remove_all(path);
    }
}
```

---

## 11. Complete API Reference

### Path Operations
| Method | Description | Notes |
|--------|-------------|-------|
| `path()` | Constructor | Multiple overloads |
| `assign()` | Assign new path | |
| `append()` / `/=` | Append with separator | Portable |
| `concat()` / `+=` | Append without separator | |
| `clear()` | Make empty | |
| `make_preferred()` | Convert separators to native | |
| `remove_filename()` | Remove filename component | |
| `replace_filename()` | Change filename | |
| `replace_extension()` | Change extension | |
| `swap()` | Swap paths | |
| `compare()` | Lexicographical comparison | |
| `root_name()` | Get root name | Empty on Unix |
| `root_directory()` | Get root directory | |
| `root_path()` | Get root path | |
| `relative_path()` | Get relative path | |
| `parent_path()` | Get parent | |
| `filename()` | Get filename | |
| `stem()` | Get stem | Without extension |
| `extension()` | Get extension | With dot |
| `empty()` | Check if empty | |
| `has_*()` | Check component presence | Multiple variants |
| `is_absolute()` | Check if absolute | |
| `is_relative()` | Check if relative | |
| `lexically_normal()` | Normalize lexically | No filesystem access |
| `lexically_relative()` | Compute relative | |
| `lexically_proximate()` | Compute proximate | |

### Filesystem Operations
| Function | Description | Throws on Error |
|----------|-------------|-----------------|
| `absolute()` | Make absolute | Yes |
| `canonical()` | Resolve to canonical | Yes |
| `weakly_canonical()` | Partial canonical | Yes |
| `relative()` | Compute relative path | Yes |
| `proximate()` | Compute proximate path | Yes |
| `copy()` | Copy files/directories | Yes |
| `copy_file()` | Copy single file | Yes |
| `copy_symlink()` | Copy symlink | Yes |
| `create_directory()` | Create directory | Yes |
| `create_directories()` | Create directory tree | Yes |
| `create_hard_link()` | Create hard link | Yes |
| `create_symlink()` | Create symlink | Yes |
| `create_directory_symlink()` | Create directory symlink | Yes |
| `current_path()` | Get/set current directory | Yes |
| `exists()` | Check existence | No (returns false) |
| `equivalent()` | Check if same object | Yes |
| `file_size()` | Get file size | Yes |
| `hard_link_count()` | Get hard link count | Yes |
| `last_write_time()` | Get/set modification time | Yes |
| `permissions()` | Get/set permissions | Yes |
| `read_symlink()` | Read symlink target | Yes |
| `remove()` | Remove file | Yes |
| `remove_all()` | Remove recursively | Yes |
| `rename()` | Rename/move | Yes |
| `resize_file()` | Change file size | Yes |
| `space()` | Get space info | Yes |
| `status()` | Get file status | No (returns status) |
| `symlink_status()` | Get symlink status | No (returns status) |
| `temp_directory_path()` | Get temp directory | Yes |
| `is_block_file()` | Check if block device | No |
| `is_character_file()` | Check if character device | No |
| `is_directory()` | Check if directory | No |
| `is_empty()` | Check if empty | Yes |
| `is_fifo()` | Check if FIFO | No |
| `is_other()` | Check if other type | No |
| `is_regular_file()` | Check if regular file | No |
| `is_socket()` | Check if socket | No |
|

# C++ Filesystem Library: Advanced Concepts & Patterns (Part 2)

## 12. Directory Entry

### Internal Caching Mechanism

**Key Concept**: `directory_entry` acts as a cache for file attributes to minimize syscalls.

```cpp
// When you iterate:
for (const auto& entry : fs::directory_iterator("dir")) {
    // entry performs ONE syscall during construction to get:
    // - File type (regular, directory, symlink, etc.)
    // - File size (if regular file)
    // - Permissions
    // - Modification time
    // All stored internally in the entry object
}
```

**Performance Impact**:
```cpp
// Scenario: Check 10,000 files for size > 1MB

// Slow version: 30,000+ syscalls
for (const auto& entry : fs::directory_iterator("dir")) {
    auto p = entry.path();
    if (fs::is_regular_file(p) &&        // syscall
        fs::file_size(p) > 1'000'000) {   // syscall
        process(p);
    }
}

// Fast version: ~10,000 syscalls
for (const auto& entry : fs::directory_iterator("dir")) {
    if (entry.is_regular_file() &&         // cached
        entry.file_size() > 1'000'000) {   // cached
        process(entry.path());
    }
}
```

### Cache Invalidation
```cpp
fs::directory_iterator it("dir");
auto entry = *it;

// Entry is cached at construction time
auto size1 = entry.file_size();  // Returns cached value

// If file is modified externally...
modify_file_externally(entry.path());

auto size2 = entry.file_size();  // Still returns OLD cached value!

// Must refresh explicitly
entry.refresh();
auto size3 = entry.file_size();  // Now returns updated value
```

**Real-World Scenario**: File monitoring
```cpp
void monitor_directory(const fs::path& dir, std::chrono::seconds interval) {
    std::map<fs::path, std::uintmax_t> last_sizes;
    
    while (true) {
        for (auto& entry : fs::directory_iterator(dir)) {
            entry.refresh();  // Critical: Get current state
            
            if (entry.is_regular_file()) {
                auto current_size = entry.file_size();
                auto it = last_sizes.find(entry.path());
                
                if (it != last_sizes.end() && it->second != current_size) {
                    std::cout << entry.path() << " changed\n";
                }
                last_sizes[entry.path()] = current_size;
            }
        }
        std::this_thread::sleep_for(interval);
    }
}
```

---

## 13. Memory & Resource Management

### Iterator Lifetime
```cpp
// WRONG: Dangling reference
fs::path get_first_file(const fs::path& dir) {
    for (const auto& entry : fs::directory_iterator(dir)) {
        return entry.path();  // Copies path - SAFE
    }
    return {};
}

// WRONG: Dangling reference
const fs::directory_entry& get_first_entry(const fs::path& dir) {
    for (const auto& entry : fs::directory_iterator(dir)) {
        return entry;  // Returns reference to local - UNDEFINED BEHAVIOR!
    }
    throw std::runtime_error("No files");
}

// RIGHT: Return by value
fs::directory_entry get_first_entry_safe(const fs::path& dir) {
    for (const auto& entry : fs::directory_iterator(dir)) {
        return entry;  // Copies entry - SAFE
    }
    throw std::runtime_error("No files");
}
```

### Large Directory Handling
```cpp
// Problem: Loading all paths into memory
std::vector<fs::path> all_files(const fs::path& dir) {
    std::vector<fs::path> result;
    for (const auto& entry : fs::recursive_directory_iterator(dir)) {
        result.push_back(entry.path());
    }
    return result;  // Could be gigabytes of memory!
}

// Solution 1: Process on-the-fly
void process_large_directory(const fs::path& dir) {
    for (const auto& entry : fs::recursive_directory_iterator(dir)) {
        process_immediately(entry);  // No storage
    }
}

// Solution 2: Lazy evaluation with generator pattern
class FileGenerator {
    fs::recursive_directory_iterator it_;
    fs::recursive_directory_iterator end_;
    
public:
    FileGenerator(const fs::path& dir) : it_(dir) {}
    
    std::optional<fs::path> next() {
        if (it_ != end_) {
            auto path = it_->path();
            ++it_;
            return path;
        }
        return std::nullopt;
    }
};

// Usage
FileGenerator gen("huge_dir");
while (auto path = gen.next()) {
    process(*path);
    // Only one path in memory at a time
}
```

### File Handle Management
```cpp
// Filesystem library doesn't manage file handles
// But affects them indirectly:

// WRONG: Too many open handles
std::vector<std::ifstream> files;
for (const auto& entry : fs::recursive_directory_iterator("dir")) {
    if (entry.is_regular_file()) {
        files.emplace_back(entry.path());  // Might exceed OS limit!
    }
}

// RIGHT: Process and close
for (const auto& entry : fs::recursive_directory_iterator("dir")) {
    if (entry.is_regular_file()) {
        std::ifstream file(entry.path());
        process(file);
        // file closed automatically
    }
}
```

---

## 14. Concurrency & Thread Safety

### Thread Safety Guarantees

**General Rule**: Different objects = safe, same object = unsafe

```cpp
// SAFE: Different directories
std::thread t1([]{ fs::directory_iterator("dir1"); });
std::thread t2([]{ fs::directory_iterator("dir2"); });

// SAFE: Different paths
fs::path p1 = "path1";
fs::path p2 = "path2";
std::thread t1([&]{ fs::file_size(p1); });
std::thread t2([&]{ fs::file_size(p2); });

// UNSAFE: Shared iterator
fs::directory_iterator it("dir");
std::thread t1([&]{ ++it; });  // Race condition!
std::thread t2([&]{ ++it; });

// UNSAFE: Modifying same path object
fs::path p = "dir";
std::thread t1([&]{ p /= "sub1"; });  // Race condition!
std::thread t2([&]{ p /= "sub2"; });
```

### Concurrent Directory Traversal
```cpp
#include <thread>
#include <mutex>
#include <queue>

class ParallelDirectoryProcessor {
    std::queue<fs::path> queue_;
    std::mutex mutex_;
    std::atomic<bool> done_{false};
    
public:
    void scan(const fs::path& root) {
        for (const auto& entry : fs::recursive_directory_iterator(root)) {
            std::lock_guard lock(mutex_);
            queue_.push(entry.path());
        }
        done_ = true;
    }
    
    void worker() {
        while (!done_ || !queue_.empty()) {
            fs::path path;
            {
                std::lock_guard lock(mutex_);
                if (queue_.empty()) continue;
                path = queue_.front();
                queue_.pop();
            }
            process(path);
        }
    }
    
    void run(const fs::path& root, int num_threads) {
        std::thread scanner(&ParallelDirectoryProcessor::scan, this, root);
        
        std::vector<std::thread> workers;
        for (int i = 0; i < num_threads; ++i) {
            workers.emplace_back(&ParallelDirectoryProcessor::worker, this);
        }
        
        scanner.join();
        for (auto& w : workers) w.join();
    }
};
```

### Filesystem Races (TOCTOU)

**Time-Of-Check-Time-Of-Use** vulnerabilities:

```cpp
// VULNERABLE: Classic TOCTOU bug
void unsafe_write(const fs::path& path) {
    if (!fs::exists(path)) {              // Check
        std::ofstream file(path);         // Use
        file << "data";
        // Between check and use, attacker could:
        // 1. Create symlink to sensitive file
        // 2. Create directory (causing open to fail)
        // 3. Create file with different permissions
    }
}

// SAFER: Don't check, handle errors
void safer_write(const fs::path& path) {
    std::ofstream file(path, std::ios::out | std::ios::excl);  // Fail if exists
    if (!file) {
        throw std::runtime_error("File exists or can't be created");
    }
    file << "data";
}

// SAFEST: Use OS-level atomic operations
void safest_write(const fs::path& path) {
    // On POSIX: open() with O_CREAT | O_EXCL is atomic
    int fd = open(path.c_str(), O_CREAT | O_EXCL | O_WRONLY, 0600);
    if (fd == -1) {
        throw std::system_error(errno, std::system_category());
    }
    write(fd, "data", 4);
    close(fd);
}
```

### Safe Temporary Files in Concurrent Environment
```cpp
class SafeTempFile {
    fs::path path_;
    int fd_ = -1;
    
public:
    SafeTempFile() {
        auto temp_dir = fs::temp_directory_path();
        
        // Use mkstemp for atomic creation (POSIX)
        std::string template_path = (temp_dir / "tmpXXXXXX").string();
        fd_ = mkstemp(&template_path[0]);
        
        if (fd_ == -1) {
            throw std::system_error(errno, std::system_category());
        }
        
        path_ = template_path;
    }
    
    ~SafeTempFile() {
        if (fd_ != -1) {
            close(fd_);
            std::error_code ec;
            fs::remove(path_, ec);
        }
    }
    
    const fs::path& path() const { return path_; }
    int fd() const { return fd_; }
};
```

---

## 15. Cross-Platform Abstractions

### Path Normalization Across Platforms
```cpp
class PortablePath {
    fs::path path_;
    
public:
    PortablePath(const std::string& path) {
        // Always use forward slashes internally
        std::string normalized = path;
        std::replace(normalized.begin(), normalized.end(), '\\', '/');
        path_ = fs::path(normalized);
    }
    
    fs::path native() const {
        return path_;  // Automatically converted by filesystem library
    }
    
    std::string portable() const {
        return path_.generic_string();  // Always forward slashes
    }
};

// Use case: Configuration files
void save_config(const std::map<std::string, fs::path>& paths) {
    std::ofstream config("config.json");
    config << "{\n";
    for (const auto& [key, path] : paths) {
        // Always save with forward slashes for portability
        config << "  \"" << key << "\": \"" 
               << path.generic_string() << "\",\n";
    }
    config << "}\n";
}
```

### Handling Case Sensitivity
```cpp
class CaseInsensitiveMap {
    std::map<std::string, fs::path> map_;
    
    static std::string normalize(const fs::path& p) {
        std::string s = p.string();
        #ifdef _WIN32
        std::transform(s.begin(), s.end(), s.begin(), ::tolower);
        #endif
        return s;
    }
    
public:
    void insert(const fs::path& path) {
        map_[normalize(path)] = path;
    }
    
    bool contains(const fs::path& path) const {
        return map_.find(normalize(path)) != map_.end();
    }
};
```

### Long Path Support (Windows)
```cpp
#ifdef _WIN32
fs::path enable_long_paths(const fs::path& path) {
    // Windows long path prefix
    if (path.is_absolute() && !path.string().starts_with("\\\\?\\")) {
        return fs::path("\\\\?\\" + path.string());
    }
    return path;
}
#else
fs::path enable_long_paths(const fs::path& path) {
    return path;  // Not needed on Unix
}
#endif

// Usage
void process_long_path(const fs::path& user_path) {
    auto path = enable_long_paths(fs::absolute(user_path));
    // Now can handle paths > 260 chars on Windows
}
```

---

## 16. Performance Optimization Patterns

### Bulk Operations Batching
```cpp
// Scenario: Delete thousands of files

// SLOW: Individual operations
for (const auto& path : paths) {
    fs::remove(path);  // Many syscalls + error handling overhead
}

// FASTER: Batch with error codes
std::error_code ec;
for (const auto& path : paths) {
    fs::remove(path, ec);  // Less exception overhead
    if (ec) {
        // Log or handle
    }
}

// FASTEST: Group by directory (locality of reference)
std::map<fs::path, std::vector<fs::path>> by_dir;
for (const auto& path : paths) {
    by_dir[path.parent_path()].push_back(path);
}
for (const auto& [dir, files] : by_dir) {
    std::error_code ec;
    for (const auto& file : files) {
        fs::remove(file, ec);
        // Better cache performance - same directory
    }
}
```

### Parallel Directory Tree Walking
```cpp
#include <execution>
#include <algorithm>

std::vector<fs::path> parallel_find(const fs::path& root, 
                                      const std::string& extension) {
    // Collect directories first
    std::vector<fs::path> dirs;
    for (const auto& entry : fs::directory_iterator(root)) {
        if (entry.is_directory()) {
            dirs.push_back(entry.path());
        }
    }
    
    // Process directories in parallel
    std::vector<fs::path> results;
    std::mutex results_mutex;
    
    std::for_each(std::execution::par, dirs.begin(), dirs.end(),
        [&](const fs::path& dir) {
            std::vector<fs::path> local_results;
            for (const auto& entry : fs::recursive_directory_iterator(dir)) {
                if (entry.path().extension() == extension) {
                    local_results.push_back(entry.path());
                }
            }
            
            std::lock_guard lock(results_mutex);
            results.insert(results.end(), 
                          local_results.begin(), local_results.end());
        });
    
    return results;
}
```

### Metadata Caching Strategy
```cpp
class FileCache {
    struct CachedInfo {
        std::uintmax_t size;
        fs::file_time_type mtime;
        fs::file_status status;
        std::chrono::steady_clock::time_point cached_at;
    };
    
    std::unordered_map<fs::path, CachedInfo> cache_;
    std::chrono::seconds ttl_;
    
    bool is_stale(const CachedInfo& info) const {
        auto now = std::chrono::steady_clock::now();
        return (now - info.cached_at) > ttl_;
    }
    
public:
    FileCache(std::chrono::seconds ttl = std::chrono::seconds(5)) 
        : ttl_(ttl) {}
    
    std::uintmax_t file_size(const fs::path& path) {
        auto it = cache_.find(path);
        if (it != cache_.end() && !is_stale(it->second)) {
            return it->second.size;  // Return cached
        }
        
        // Fetch and cache
        CachedInfo info;
        info.size = fs::file_size(path);
        info.mtime = fs::last_write_time(path);
        info.status = fs::status(path);
        info.cached_at = std::chrono::steady_clock::now();
        
        cache_[path] = info;
        return info.size;
    }
    
    void invalidate(const fs::path& path) {
        cache_.erase(path);
    }
};
```

---

## 17. Advanced Error Recovery

### Retry Logic for Transient Failures
```cpp
template<typename Func>
bool retry_operation(Func&& op, int max_attempts = 3, 
                     std::chrono::milliseconds delay = std::chrono::milliseconds(100)) {
    for (int attempt = 1; attempt <= max_attempts; ++attempt) {
        std::error_code ec;
        op(ec);
        
        if (!ec) return true;
        
        // Check if error is transient
        if (ec == std::errc::resource_unavailable_try_again ||
            ec == std::errc::no_lock_available) {
            
            if (attempt < max_attempts) {
                std::this_thread::sleep_for(delay * attempt);  // Exponential backoff
                continue;
            }
        }
        
        return false;  // Permanent error or max attempts reached
    }
    return false;
}

// Usage
bool success = retry_operation([&](std::error_code& ec) {
    fs::remove("locked_file.txt", ec);
});
```

### Graceful Degradation
```cpp
fs::path find_config_file() {
    // Try multiple locations in order of preference
    std::vector<fs::path> candidates = {
        fs::current_path() / "config.json",
        fs::path(std::getenv("HOME")) / ".myapp" / "config.json",
        "/etc/myapp/config.json"
    };
    
    for (const auto& path : candidates) {
        std::error_code ec;
        if (fs::exists(path, ec) && fs::is_regular_file(path, ec)) {
            return path;
        }
    }
    
    // Fallback: Create default config
    auto default_path = fs::current_path() / "config.json";
    create_default_config(default_path);
    return default_path;
}
```

### Transactional Filesystem Operations
```cpp
class FilesystemTransaction {
    struct Operation {
        enum Type { CREATE, REMOVE, RENAME } type;
        fs::path path1, path2;
        std::vector<char> backup_data;
    };
    
    std::vector<Operation> operations_;
    std::vector<Operation> undo_log_;
    
public:
    void create_file(const fs::path& path, const std::string& content) {
        operations_.push_back({Operation::CREATE, path, {}, {}});
        // Store operation
    }
    
    void remove_file(const fs::path& path) {
        // Backup content before removing
        std::ifstream file(path, std::ios::binary);
        std::vector<char> backup((std::istreambuf_iterator<char>(file)),
                                  std::istreambuf_iterator<char>());
        operations_.push_back({Operation::REMOVE, path, {}, std::move(backup)});
    }
    
    void rename_file(const fs::path& from, const fs::path& to) {
        operations_.push_back({Operation::RENAME, from, to, {}});
    }
    
    bool commit() {
        undo_log_.clear();
        
        for (const auto& op : operations_) {
            try {
                switch (op.type) {
                    case Operation::CREATE:
                        std::ofstream(op.path1) << "content";
                        undo_log_.push_back({Operation::REMOVE, op.path1, {}, {}});
                        break;
                    
                    case Operation::REMOVE:
                        fs::remove(op.path1);
                        undo_log_.push_back({Operation::CREATE, op.path1, {}, op.backup_data});
                        break;
                    
                    case Operation::RENAME:
                        fs::rename(op.path1, op.path2);
                        undo_log_.push_back({Operation::RENAME, op.path2, op.path1, {}});
                        break;
                }
            } catch (...) {
                rollback();
                return false;
            }
        }
        
        operations_.clear();
        return true;
    }
    
    void rollback() {
        for (auto it = undo_log_.rbegin(); it != undo_log_.rend(); ++it) {
            try {
                switch (it->type) {
                    case Operation::CREATE:
                        if (!it->backup_data.empty()) {
                            std::ofstream file(it->path1, std::ios::binary);
                            file.write(it->backup_data.data(), it->backup_data.size());
                        }
                        break;
                    
                    case Operation::REMOVE:
                        std::error_code ec;
                        fs::remove(it->path1, ec);
                        break;
                    
                    case Operation::RENAME:
                        std::error_code ec;
                        fs::rename(it->path1, it->path2, ec);
                        break;
                }
            } catch (...) {
                // Best effort rollback
            }
        }
        
        undo_log_.clear();
        operations_.clear();
    }
};

// Usage
FilesystemTransaction txn;
txn.create_file("file1.txt", "data");
txn.rename_file("old.txt", "new.txt");
txn.remove_file("temp.txt");

if (!txn.commit()) {
    // Transaction failed and was rolled back
}
```

---

## 18. Security Considerations

### Path Traversal Prevention
```cpp
bool is_safe_path(const fs::path& base, const fs::path& user_input) {
    try {
        auto abs_base = fs::canonical(base);
        auto abs_path = fs::weakly_canonical(abs_base / user_input);
        
        // Check if resolved path is within base
        auto rel = abs_path.lexically_relative(abs_base);
        
        // If relative path goes up beyond base, it's unsafe
        return !rel.empty() && *rel.begin() != "..";
        
    } catch (const fs::filesystem_error&) {
        return false;  // Error = unsafe
    }
}

// Usage
fs::path base = "/var/www/uploads";
std::string user_file = "../../etc/passwd";  // Attack attempt

if (is_safe_path(base, user_file)) {
    // Safe to access
    auto full_path = base / user_file;
    process(full_path);
} else {
    throw std::runtime_error("Path traversal attempt detected");
}
```

### Symlink Attack Prevention
```cpp
void safe_create_file(const fs::path& path) {
    auto parent = path.parent_path();
    
    // Ensure parent exists and is not a symlink
    if (fs::exists(parent)) {
        auto status = fs::symlink_status(parent);
        if (fs::is_symlink(status)) {
            throw std::runtime_error("Parent is a symlink - potential attack");
        }
    }
    
    // Check if target exists and is a symlink
    if (fs::exists(path)) {
        auto status = fs::symlink_status(path);
        if (fs::is_symlink(status)) {
            throw std::runtime_error("Target is a symlink - potential attack");
        }
    }
    
    // Safe to create
    std::ofstream file(path);
}
```

### Permission Verification
```cpp
bool has_safe_permissions(const fs::path& path) {
    auto status = fs::status(path);
    auto perms = status.permissions();
    
    // Check if world-writable (security risk)
    if ((perms & fs::perms::others_write) != fs::perms::none) {
        return false;
    }
    
    // Check if group-writable (may be risk depending on context)
    if ((perms & fs::perms::group_write) != fs::perms::none) {
        // Additional checks needed
    }
    
    return true;
}

void secure_config_file(const fs::path& path) {
    // Set restrictive permissions: owner read/write only
    fs::permissions(path, 
        fs::perms::owner_read | fs::perms::owner_write,
        fs::perm_options::replace);
}
```

---

## 19. Testing Filesystem Code

### Mock Filesystem for Unit Tests
```cpp
class IFilesystem {
public:
    virtual ~IFilesystem() = default;
    virtual bool exists(const fs::path& p) const = 0;
    virtual std::uintmax_t file_size(const fs::path& p) const = 0;
    virtual void create_directory(const fs::path& p) = 0;
    // ... other operations
};

class RealFilesystem : public IFilesystem {
public:
    bool exists(const fs::path& p) const override {
        return fs::exists(p);
    }
    
    std::uintmax_t file_size(const fs::path& p) const override {
        return fs::file_size(p);
    }
    
    void create_directory(const fs::path& p) override {
        fs::create_directory(p);
    }
};

class MockFilesystem : public IFilesystem {
    std::map<fs::path, std::uintmax_t> files_;
    std::set<fs::path> directories_;
    
public:
    bool exists(const fs::path& p) const override {
        return files_.count(p) || directories_.count(p);
    }
    
    std::uintmax_t file_size(const fs::path& p) const override {
        auto it = files_.find(p);
        if (it == files_.end()) throw fs::filesystem_error("Not found", p, {});
        return it->second;
    }
    
    void create_directory(const fs::path& p) override {
        directories_.insert(p);
    }
    
    void add_file(const fs::path& p, std::uintmax_t size) {
        files_[p] = size;
    }
};

// Usage in tests
void test_function() {
    MockFilesystem mock_fs;
    mock_fs.add_file("test.txt", 1024);
    
    // Test your code with mock_fs instead of real filesystem
}
```

### Temporary Test Directory Pattern
```cpp
class TempTestDir {
    fs::path path_;
    
public:
    TempTestDir() {
        path_ = fs::temp_directory_path() / 
                ("test_" + std::to_string(std::rand()));
        fs::create_directories(path_);
    }
    
    ~TempTestDir() {
        std::error_code ec;
        fs::remove_all(path_, ec);
    }
    
    fs::path path() const { return path_; }
    
    fs::path create_file(const std::string& name, const std::string& content = "") {
        auto file_path = path_ / name;
        std::ofstream(file_path) << content;
        return file_path;
    }
    
    fs::path create_subdir(const std::string& name) {
        auto dir_path = path_ / name;
        fs::create_directories(dir_path);
        return dir_path;
    }
};

// Usage
void test_my_function() {
    TempTestDir test_dir;
    auto file1 = test_dir.create_file("test1.txt", "content");
    auto subdir = test_dir.create_subdir("subdir");
    
    // Run tests
    my_function(test_dir.path());
    
    // Cleanup automatic
}
```

---

## 20. Real-World Complete Examples

### Project File Organizer
```cpp
class FileOrganizer {
    fs::path source_dir_;
    fs::path target_dir_;
    
    std::map<std::string, std::string> extension_to_folder_ = {
        {".jpg", "Images"}, {".png", "Images"}, {".gif", "Images"},
        {".mp4", "Videos"}, {".avi", "Videos"},
        {".pdf", "Documents"}, {".docx", "Documents"},
        {".zip", "Archives"}, {".tar", "Archives"}
    };
    
public:
    FileOrganizer(const fs::path& source, const fs::path& target)
        : source_dir_(source), target_dir_(target) {}
    
    void organize() {
        for (const auto& entry : fs::directory_iterator(source_dir_)) {
            if (!entry.is_regular_file()) continue;
            
            auto ext = entry.path().extension().string();
            auto it = extension_to_folder_.find(ext);
            
            fs::path target_folder = target_dir_;
            if (it != extension_to_folder_.end()) {
                target_folder /= it->second;
            } else {
                target_folder /= "Others";
            }
            
            std::error_code ec;
            fs::create_directories(target_folder, ec);
            
            auto target_file = target_folder / entry.path().filename();
            
            // Handle name conflicts
            if (fs::exists(target_file)) {
                target_file = find_unique_name(target_file);
            }
            
            fs::rename(entry.path(), target_file, ec);
            if (ec) {
                std::cerr << "Failed to move " << entry.path() 
                         << ": " << ec.message() << '\n';
            }
        }
    }
    
private:
    fs::path find_unique_name(const fs::path& path) {
        auto stem = path.stem();
        auto ext = path.extension();
        auto parent = path.parent_path();
        
        for (int i = 1; ; ++i) {
            auto new_path = parent / (stem.string() + "_" + std::to_string(i) + ext.string());
            if (!fs::exists(new_path)) {
                return new_path;
            }
        }
    }
};
```

### Backup System with Incremental Support
```cpp
class BackupSystem {
    fs::path source_;
    fs::path backup_root_;
    fs::path manifest_file_;
    
    struct FileInfo {
        std::uintmax_t size;
        fs::file_time_type mtime;
    };
    
    std::map<fs::path, FileInfo> last_manifest_;
    
public:
    BackupSystem(const fs::path& source, const fs::path& backup_root)
        : source_(source), backup_root_(backup...
...
};

class BackupSystem {
    fs::path source_;
    fs::path backup_root_;
    fs::path manifest_file_;
    
    struct FileInfo {
        std::uintmax_t size;
        fs::file_time_type mtime;
        std::string hash;  // Optional: for content verification
    };
    
    std::map<fs::path, FileInfo> last_manifest_;
    
public:
    BackupSystem(const fs::path& source, const fs::path& backup_root)
        : source_(source), 
          backup_root_(backup_root),
          manifest_file_(backup_root / "manifest.json") {
        
        fs::create_directories(backup_root_);
        load_manifest();
    }
    
    void perform_backup() {
        auto timestamp = std::time(nullptr);
        auto backup_dir = backup_root_ / std::to_string(timestamp);
        fs::create_directories(backup_dir);
        
        std::map<fs::path, FileInfo> current_manifest;
        std::vector<fs::path> changed_files;
        
        // Scan source directory
        for (const auto& entry : fs::recursive_directory_iterator(source_)) {
            if (!entry.is_regular_file()) continue;
            
            auto rel_path = fs::relative(entry.path(), source_);
            FileInfo info{
                entry.file_size(),
                entry.last_write_time(),
                ""
            };
            
            current_manifest[rel_path] = info;
            
            // Check if file changed
            auto it = last_manifest_.find(rel_path);
            if (it == last_manifest_.end() ||
                it->second.size != info.size ||
                it->second.mtime != info.mtime) {
                
                changed_files.push_back(rel_path);
            }
        }
        
        // Copy only changed files
        std::cout << "Backing up " << changed_files.size() << " changed files...\n";
        for (const auto& rel_path : changed_files) {
            auto src = source_ / rel_path;
            auto dst = backup_dir / rel_path;
            
            fs::create_directories(dst.parent_path());
            
            std::error_code ec;
            fs::copy_file(src, dst, fs::copy_options::overwrite_existing, ec);
            
            if (ec) {
                std::cerr << "Failed to backup " << rel_path 
                         << ": " << ec.message() << '\n';
            }
        }
        
        // Save manifest
        last_manifest_ = current_manifest;
        save_manifest();
        
        // Create symlink to latest backup
        auto latest_link = backup_root_ / "latest";
        std::error_code ec;
        fs::remove(latest_link, ec);
        fs::create_directory_symlink(backup_dir, latest_link, ec);
    }
    
    void restore(const fs::path& backup_name, const fs::path& restore_to) {
        auto backup_dir = backup_root_ / backup_name;
        
        if (!fs::exists(backup_dir)) {
            throw std::runtime_error("Backup not found");
        }
        
        fs::create_directories(restore_to);
        
        for (const auto& entry : fs::recursive_directory_iterator(backup_dir)) {
            if (!entry.is_regular_file()) continue;
            
            auto rel_path = fs::relative(entry.path(), backup_dir);
            auto dst = restore_to / rel_path;
            
            fs::create_directories(dst.parent_path());
            fs::copy_file(entry.path(), dst, fs::copy_options::overwrite_existing);
        }
        
        std::cout << "Restore completed to " << restore_to << '\n';
    }
    
    void list_backups() {
        std::vector<std::pair<std::time_t, fs::path>> backups;
        
        for (const auto& entry : fs::directory_iterator(backup_root_)) {
            if (entry.is_directory() && entry.path().filename() != "latest") {
                try {
                    auto timestamp = std::stoll(entry.path().filename().string());
                    backups.emplace_back(timestamp, entry.path());
                } catch (...) {
                    // Skip non-timestamp directories
                }
            }
        }
        
        std::sort(backups.begin(), backups.end());
        
        std::cout << "Available backups:\n";
        for (const auto& [timestamp, path] : backups) {
            std::cout << "  " << std::ctime(&timestamp) 
                     << "  (" << path.filename() << ")\n";
        }
    }
    
private:
    void load_manifest() {
        if (!fs::exists(manifest_file_)) return;
        
        std::ifstream file(manifest_file_);
        // Parse JSON manifest (simplified - use real JSON library in production)
        // last_manifest_ = parse_json(file);
    }
    
    void save_manifest() {
        std::ofstream file(manifest_file_);
        // Write JSON manifest (simplified)
        // file << to_json(last_manifest_);
    }
};

// Usage
int main() {
    BackupSystem backup("/home/user/documents", "/mnt/backup");
    
    backup.perform_backup();  // Incremental backup
    backup.list_backups();
    // backup.restore("1234567890", "/home/user/restored");
}
```

### Duplicate File Finder
```cpp
#include <openssl/md5.h>  // For hashing

class DuplicateFinder {
    struct FileGroup {
        std::uintmax_t size;
        std::vector<fs::path> paths;
    };
    
    std::string compute_md5(const fs::path& path) {
        std::ifstream file(path, std::ios::binary);
        MD5_CTX md5_ctx;
        MD5_Init(&md5_ctx);
        
        char buffer[8192];
        while (file.read(buffer, sizeof(buffer)) || file.gcount()) {
            MD5_Update(&md5_ctx, buffer, file.gcount());
        }
        
        unsigned char hash[MD5_DIGEST_LENGTH];
        MD5_Final(hash, &md5_ctx);
        
        std::ostringstream oss;
        for (int i = 0; i < MD5_DIGEST_LENGTH; ++i) {
            oss << std::hex << std::setw(2) << std::setfill('0') 
                << static_cast<int>(hash[i]);
        }
        return oss.str();
    }
    
public:
    std::vector<std::vector<fs::path>> find_duplicates(const fs::path& root) {
        // Step 1: Group files by size (cheap check)
        std::map<std::uintmax_t, std::vector<fs::path>> size_groups;
        
        std::cout << "Scanning files...\n";
        for (const auto& entry : fs::recursive_directory_iterator(root,
            fs::directory_options::skip_permission_denied)) {
            
            if (entry.is_regular_file()) {
                size_groups[entry.file_size()].push_back(entry.path());
            }
        }
        
        // Step 2: For groups with multiple files, compute hashes
        std::map<std::string, std::vector<fs::path>> hash_groups;
        std::size_t candidates = 0;
        
        for (const auto& [size, paths] : size_groups) {
            if (paths.size() < 2 || size == 0) continue;  // Skip unique files and empty
            
            candidates += paths.size();
        }
        
        std::cout << "Hashing " << candidates << " potential duplicates...\n";
        std::size_t processed = 0;
        
        for (const auto& [size, paths] : size_groups) {
            if (paths.size() < 2 || size == 0) continue;
            
            for (const auto& path : paths) {
                try {
                    auto hash = compute_md5(path);
                    hash_groups[hash].push_back(path);
                    
                    if (++processed % 100 == 0) {
                        std::cout << "Processed " << processed << "/" 
                                 << candidates << '\r' << std::flush;
                    }
                } catch (const std::exception& e) {
                    std::cerr << "Error hashing " << path << ": " 
                             << e.what() << '\n';
                }
            }
        }
        
        // Step 3: Extract duplicate groups
        std::vector<std::vector<fs::path>> duplicates;
        std::uintmax_t wasted_space = 0;
        
        for (const auto& [hash, paths] : hash_groups) {
            if (paths.size() > 1) {
                duplicates.push_back(paths);
                
                auto size = fs::file_size(paths[0]);
                wasted_space += size * (paths.size() - 1);
            }
        }
        
        std::cout << "\nFound " << duplicates.size() << " duplicate groups\n";
        std::cout << "Wasted space: " << (wasted_space / 1024 / 1024) << " MB\n";
        
        return duplicates;
    }
    
    void print_duplicates(const std::vector<std::vector<fs::path>>& duplicates) {
        for (const auto& group : duplicates) {
            auto size = fs::file_size(group[0]);
            std::cout << "\n=== Duplicate group (" << size << " bytes) ===\n";
            for (const auto& path : group) {
                std::cout << "  " << path << '\n';
            }
        }
    }
    
    void remove_duplicates_interactive(const std::vector<std::vector<fs::path>>& duplicates) {
        for (const auto& group : duplicates) {
            std::cout << "\n=== Duplicate group ===\n";
            for (size_t i = 0; i < group.size(); ++i) {
                std::cout << i << ": " << group[i] << '\n';
            }
            
            std::cout << "Keep which file? (0-" << (group.size()-1) 
                     << ", or 's' to skip): ";
            std::string input;
            std::cin >> input;
            
            if (input == "s") continue;
            
            try {
                size_t keep_idx = std::stoul(input);
                if (keep_idx >= group.size()) continue;
                
                for (size_t i = 0; i < group.size(); ++i) {
                    if (i != keep_idx) {
                        std::error_code ec;
                        fs::remove(group[i], ec);
                        if (ec) {
                            std::cerr << "Failed to remove " << group[i] 
                                     << ": " << ec.message() << '\n';
                        }
                    }
                }
            } catch (...) {
                std::cout << "Invalid input, skipping\n";
            }
        }
    }
};

// Usage
int main() {
    DuplicateFinder finder;
    auto duplicates = finder.find_duplicates("/home/user/Downloads");
    finder.print_duplicates(duplicates);
    // finder.remove_duplicates_interactive(duplicates);
}
```

### Build System Cache Manager
```cpp
class BuildCacheManager {
    fs::path cache_dir_;
    std::uintmax_t max_cache_size_;
    
    struct CacheEntry {
        fs::path path;
        fs::file_time_type last_access;
        std::uintmax_t size;
    };
    
public:
    BuildCacheManager(const fs::path& cache_dir, std::uintmax_t max_size_mb)
        : cache_dir_(cache_dir), 
          max_cache_size_(max_size_mb * 1024 * 1024) {
        fs::create_directories(cache_dir_);
    }
    
    std::optional<fs::path> get_cached_object(const std::string& source_hash) {
        auto cache_file = cache_dir_ / (source_hash + ".o");
        
        if (fs::exists(cache_file)) {
            // Update access time
            fs::last_write_time(cache_file, fs::file_time_type::clock::now());
            return cache_file;
        }
        
        return std::nullopt;
    }
    
    void store_object(const std::string& source_hash, const fs::path& object_file) {
        auto cache_file = cache_dir_ / (source_hash + ".o");
        
        // Copy to cache
        fs::copy_file(object_file, cache_file, 
                     fs::copy_options::overwrite_existing);
        
        // Evict old entries if cache is too large
        evict_if_needed();
    }
    
    void evict_if_needed() {
        std::uintmax_t total_size = 0;
        std::vector<CacheEntry> entries;
        
        // Collect all cache entries
        for (const auto& entry : fs::directory_iterator(cache_dir_)) {
            if (entry.is_regular_file()) {
                entries.push_back({
                    entry.path(),
                    entry.last_write_time(),
                    entry.file_size()
                });
                total_size += entry.file_size();
            }
        }
        
        if (total_size <= max_cache_size_) return;
        
        // Sort by last access time (LRU)
        std::sort(entries.begin(), entries.end(),
            [](const CacheEntry& a, const CacheEntry& b) {
                return a.last_access < b.last_access;
            });
        
        // Remove oldest entries until under limit
        for (const auto& entry : entries) {
            if (total_size <= max_cache_size_) break;
            
            std::error_code ec;
            fs::remove(entry.path, ec);
            if (!ec) {
                total_size -= entry.size;
                std::cout << "Evicted: " << entry.path.filename() << '\n';
            }
        }
    }
    
    void print_stats() {
        std::uintmax_t total_size = 0;
        std::size_t file_count = 0;
        
        for (const auto& entry : fs::directory_iterator(cache_dir_)) {
            if (entry.is_regular_file()) {
                total_size += entry.file_size();
                ++file_count;
            }
        }
        
        std::cout << "Cache statistics:\n";
        std::cout << "  Files: " << file_count << '\n';
        std::cout << "  Size: " << (total_size / 1024 / 1024) << " MB / "
                  << (max_cache_size_ / 1024 / 1024) << " MB\n";
        std::cout << "  Usage: " << (total_size * 100 / max_cache_size_) << "%\n";
    }
    
    void clear() {
        std::error_code ec;
        for (const auto& entry : fs::directory_iterator(cache_dir_)) {
            fs::remove(entry.path(), ec);
        }
        std::cout << "Cache cleared\n";
    }
};
```

---

## 21. Integration Patterns

### Logging Filesystem Operations
```cpp
class LoggedFilesystem {
    std::ofstream log_;
    
    void log(const std::string& operation, const fs::path& path, 
             const std::string& result) {
        auto now = std::chrono::system_clock::now();
        auto time_t = std::chrono::system_clock::to_time_t(now);
        
        log_ << std::put_time(std::localtime(&time_t), "%Y-%m-%d %H:%M:%S")
             << " | " << operation << " | " << path << " | " << result << '\n';
        log_.flush();
    }
    
public:
    LoggedFilesystem(const fs::path& log_file) : log_(log_file, std::ios::app) {}
    
    bool create_directory(const fs::path& path) {
        std::error_code ec;
        bool result = fs::create_directory(path, ec);
        
        log("CREATE_DIR", path, ec ? ec.message() : "SUCCESS");
        return result;
    }
    
    bool remove(const fs::path& path) {
        std::error_code ec;
        bool result = fs::remove(path, ec);
        
        log("REMOVE", path, ec ? ec.message() : "SUCCESS");
        return result;
    }
    
    bool copy(const fs::path& from, const fs::path& to) {
        std::error_code ec;
        fs::copy(from, to, ec);
        
        log("COPY", from.string() + " -> " + to.string(), 
            ec ? ec.message() : "SUCCESS");
        return !ec;
    }
    
    // Add more operations as needed...
};
```

### Configuration-Based Path Management
```cpp
class PathResolver {
    std::map<std::string, fs::path> base_paths_;
    
public:
    PathResolver() {
        // Initialize with common paths
        base_paths_["home"] = fs::path(std::getenv("HOME"));
        base_paths_["temp"] = fs::temp_directory_path();
        base_paths_["current"] = fs::current_path();
        
        // Application-specific paths
        base_paths_["config"] = base_paths_["home"] / ".myapp";
        base_paths_["cache"] = base_paths_["home"] / ".cache" / "myapp";
        base_paths_["data"] = base_paths_["home"] / ".local" / "share" / "myapp";
    }
    
    fs::path resolve(const std::string& path_spec) {
        // Support syntax like: ${home}/documents/file.txt
        std::regex var_regex(R"(\$\{([^}]+)\})");
        std::smatch match;
        std::string result = path_spec;
        
        while (std::regex_search(result, match, var_regex)) {
            std::string var_name = match[1].str();
            auto it = base_paths_.find(var_name);
            
            if (it != base_paths_.end()) {
                result.replace(match.position(), match.length(), 
                              it->second.string());
            } else {
                throw std::runtime_error("Unknown path variable: " + var_name);
            }
        }
        
        return fs::path(result);
    }
    
    void add_base_path(const std::string& name, const fs::path& path) {
        base_paths_[name] = path;
    }
    
    void ensure_exists(const std::string& name) {
        auto it = base_paths_.find(name);
        if (it != base_paths_.end()) {
            fs::create_directories(it->second);
        }
    }
};

// Usage
PathResolver resolver;
resolver.ensure_exists("config");
resolver.ensure_exists("cache");

auto config_file = resolver.resolve("${config}/settings.json");
auto cache_dir = resolver.resolve("${cache}/images");
```

---

## 22. Advanced Techniques

### Memory-Mapped Files (Integration with filesystem)
```cpp
#include <sys/mman.h>
#include <fcntl.h>

class MemoryMappedFile {
    void* data_ = nullptr;
    std::size_t size_ = 0;
    int fd_ = -1;
    
public:
    MemoryMappedFile(const fs::path& path) {
        if (!fs::exists(path)) {
            throw std::runtime_error("File doesn't exist");
        }
        
        size_ = fs::file_size(path);
        fd_ = open(path.c_str(), O_RDONLY);
        
        if (fd_ == -1) {
            throw std::system_error(errno, std::system_category());
        }
        
        data_ = mmap(nullptr, size_, PROT_READ, MAP_PRIVATE, fd_, 0);
        
        if (data_ == MAP_FAILED) {
            close(fd_);
            throw std::system_error(errno, std::system_category());
        }
    }
    
    ~MemoryMappedFile() {
        if (data_ != nullptr) {
            munmap(data_, size_);
        }
        if (fd_ != -1) {
            close(fd_);
        }
    }
    
    const char* data() const { return static_cast<const char*>(data_); }
    std::size_t size() const { return size_; }
    
    // Prevent copying
    MemoryMappedFile(const MemoryMappedFile&) = delete;
    MemoryMappedFile& operator=(const MemoryMappedFile&) = delete;
};

// Usage: Fast file processing
void process_large_file(const fs::path& path) {
    MemoryMappedFile mapped(path);
    
    // Process data without loading into memory explicitly
    std::string_view content(mapped.data(), mapped.size());
    
    // Fast searching, parsing, etc.
    auto pos = content.find("pattern");
}
```

### Filesystem Watcher (Platform-Specific)
```cpp
// Linux inotify example
#include <sys/inotify.h>

class FileWatcher {
    int inotify_fd_ = -1;
    std::map<int, fs::path> watch_descriptors_;
    
public:
    FileWatcher() {
        inotify_fd_ = inotify_init();
        if (inotify_fd_ == -1) {
            throw std::system_error(errno, std::system_category());
        }
    }
    
    ~FileWatcher() {
        if (inotify_fd_ != -1) {
            close(inotify_fd_);
        }
    }
    
    void watch_directory(const fs::path& path) {
        int wd = inotify_add_watch(inotify_fd_, path.c_str(),
            IN_CREATE | IN_DELETE | IN_MODIFY | IN_MOVED_FROM | IN_MOVED_TO);
        
        if (wd == -1) {
            throw std::system_error(errno, std::system_category());
        }
        
        watch_descriptors_[wd] = path;
    }
    
    void run(std::function<void(const std::string&, const fs::path&)> callback) {
        char buffer[4096];
        
        while (true) {
            int length = read(inotify_fd_, buffer, sizeof(buffer));
            
            if (length < 0) {
                throw std::system_error(errno, std::system_category());
            }
            
            int i = 0;
            while (i < length) {
                auto* event = reinterpret_cast<inotify_event*>(&buffer[i]);
                
                auto it = watch_descriptors_.find(event->wd);
                if (it != watch_descriptors_.end()) {
                    std::string event_type;
                    
                    if (event->mask & IN_CREATE) event_type = "CREATE";
                    else if (event->mask & IN_DELETE) event_type = "DELETE";
                    else if (event->mask & IN_MODIFY) event_type = "MODIFY";
                    else if (event->mask & IN_MOVED_FROM) event_type = "MOVED_FROM";
                    else if (event->mask & IN_MOVED_TO) event_type = "MOVED_TO";
                    
                    callback(event_type, it->second / event->name);
                }
                
                i += sizeof(inotify_event) + event->len;
            }
        }
    }
};

// Usage
FileWatcher watcher;
watcher.watch_directory("/home/user/watched");

watcher.run([](const std::string& event, const fs::path& path) {
    std::cout << event << ": " << path << '\n';
});
```

---

## 23. Performance Profiling

### Filesystem Operation Profiler
```cpp
class FilesystemProfiler {
    struct Timing {
        std::string operation;
        std::chrono::microseconds duration;
        fs::path path;
    };
    
    std::vector<Timing> timings_;
    
    template<typename Func>
    auto profile(const std::string& op, const fs::path& path, Func&& func) {
        auto start = std::chrono::high_resolution_clock::now();
        
        auto result = func();
        
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        
        timings_.push_back({op, duration, path});
        
        return result;
    }
    
public:
    bool create_directory(const fs::path& path) {
        return profile("create_directory", path, [&] {
            return fs::create_directory(path);
        });
    }
    
    std::uintmax_t file_size(const fs::path& path) {
        return profile("file_size", path, [&] {
            return fs::file_size(path);
        });
    }
    
    void copy_file(const fs::path& from, const fs::path& to) {
        profile("copy_file", from, [&] {
            fs::copy_file(from, to);
            return 0;
        });
    }
    
    void print_report() {
        // Group by operation
        std::map<std::string, std::vector<std::chrono::microseconds>> by_operation;
        
        for (const auto& timing : timings_) {
            by_operation[timing.operation].push_back(timing.duration);
        }
        
        std::cout << "=== Filesystem Operation Profile ===\n";
        for (const auto& [op, durations] : by_operation) {
            auto total = std::accumulate(durations.begin(), durations.end(),
                                        std::chrono::microseconds(0));
            auto avg = total / durations.size();
            auto max = *std::max_element(durations.begin(), durations.end());
            
            std::cout << op << ":\n";
            std::cout << "  Count: " << durations.size() << '\n';
            std::cout << "  Total: " << total.count() << " μs\n";
            std::cout << "  Avg: " << avg.count() << " μs\n";
            std::cout << "  Max: " << max.count() << " μs\n\n";
        }
        
        // Top 10 slowest operations
        std::sort(timings_.begin(), timings_.end(),
            [](const Timing& a, const Timing& b) {
                return a.duration > b.duration;
            });
        
        std::cout << "=== Top 10 Slowest Operations ===\n";
        for (size_t i = 0; i < std::min(size_t(10), timings_.size()); ++i) {
            const auto& t = timings_[i];
            std::cout << t.operation << " on " << t.path 
                     << ": " << t.duration.count() << " μs\n";
        }
    }
};
```

---

## 24. Common Anti-Patterns to Avoid

### Anti-Pattern 1: Excessive Existence Checking
```cpp
// BAD: Redundant checks
if (fs::exists(path)) {
    if (fs::is_regular_file(path)) {
        if (fs::file_size(path) > 0) {
            process(path);
        }
    }
}

// GOOD: Combine checks with error handling
std::error_code ec;
if (fs::is_regular_file(path, ec) && !ec) {
    auto size = fs::file_size(path, ec);
    if (!ec && size > 0) {
        process(path);
    }
}
```

### Anti-Pattern 2: String Concatenation for Paths
```cpp
// BAD: Platform-dependent
std::string path = dir + "/" + filename;  // Wrong on Windows

// GOOD: Use path operator/
fs::path path = fs::path(dir) / filename;  // Works everywhere
```

### Anti-Pattern 3: Ignoring Error Codes
```cpp
// BAD: Silent failures
fs::remove("important_file.txt");  // Did it work? Who knows!

// GOOD: Check results
std::error_code ec;
fs::remove("important_file.txt", ec);
if (ec) {
    std::cerr << "Failed to remove file: " << ec.message() << '\n';
    // Handle error appropriately
}
```

### Anti-Pattern 4: Not Using directory_entry Caching
```cpp
// BAD: Multiple syscalls per file
for (const auto& entry : fs::directory_iterator("dir")) {
    auto path = entry.path();
    if (fs::is_regular_file(path)) {          // Syscall
        auto size = fs::file_size(path);       // Syscall
        auto time = fs::last_write_time(path); // Syscall
        process(path, size, time);
    }
}

// GOOD: Use cached information
for (const auto& entry : fs::directory_iterator("dir")) {
    if (entry.is_regular_file()) {      // Cached
        auto size = entry.file_size();   // Cached
        auto time = entry.last_write_time(); // Cached
        process(entry.path(), size, time);
    }
}
```

### Anti-Pattern 5: Absolute Paths in Portable Code
```cpp
// BAD: Hard-coded absolute paths
fs::path config = "/home/user/.config/app/settings.json";  // Unix only!

// GOOD: Construct paths dynamically
fs::path config = fs::path(std::getenv("HOME")) / ".config" / "app" / "settings.json";

// BETTER: Use proper path resolution
fs::path get_config_path() {
    #ifdef _WIN32
    return fs::path(std::getenv("APPDATA")) / "MyApp" / "settings.json";
    #else
    return fs::path(std::getenv("HOME")) / ".config" / "myapp" / "settings.json";
    #endif
}
```

---

## 25. Quick Reference: Decision Trees

### When to Use Which Iterator?
```
Need to traverse directories?
│
├─ Single level only?
│  └─ Use: fs::directory_iterator
│
└─ Need recursion?
   │
   ├─ Simple full traversal?
   │  └─ Use: fs::recursive_directory_iterator
   │
   └─ Need depth control / custom filtering?
      └─ Use: fs::recursive_directory_iterator with:
         - depth() to check current depth
         - disable_recursion_pending() to skip directories
         - pop() to go back up levels
```

### Error Handling Strategy Selection
```
Choose error handling approach:
│
├─ Expected to fail sometimes? (e.g., checking if file exists)
│  └─ Use: std::error_code version (no exceptions)
│
├─ Failure is exceptional? (e.g., copying config file)
│  └─ Use: throwing version (handle with try-catch)
│
└─ Performance-critical loop?
   └─ Use: std::error_code version (less overhead)
```

### Path Operation Selection
```
Need to resolve a path?
│
├─ Path must exist?
│  └─ Use: fs::canonical() (throws if doesn't exist)
│
├─ Path may not exist?
│  └─ Use: fs::weakly_canonical() (resolves existing parts)
│
├─ Just need absolute path?
│  └─ Use: fs::absolute() (no symlink resolution)
│
└─ Just normalize lexically?
   └─ Use: path::lexically_normal() (no filesystem access)
```

---

## 26. Production-Ready Utilities Library

### Complete Filesystem Utilities Class
```cpp
#include <filesystem>
#include <fstream>
#include <optional>
#include <functional>

namespace fs = std::filesystem;

class FilesystemUtils {
public:
    // Safe file reading with size limit
    static std::optional<std::string> read_file_safe(
        const fs::path& path, 
        std::uintmax_t max_size = 10 * 1024 * 1024) {  // 10MB default
        
        std::error_code ec;
        if (!fs::is_regular_file(path, ec) || ec) {
            return std::nullopt;
        }
        
        auto size = fs::file_size(path, ec);
        if (ec || size > max_size) {
            return std::nullopt;
        }
        
        std::ifstream file(path, std::ios::binary);
        if (!file) {
            return std::nullopt;
        }
        
        std::string content(size, '\0');
        file.read(&content[0], size);
        
        return content;
    }
    
    // Atomic file write
    static bool write_file_atomic(const fs::path& path, const std::string& content) {
        auto temp_path = path;
        temp_path += ".tmp." + std::to_string(std::rand());
        
        try {
            std::ofstream file(temp_path, std::ios::binary);
            if (!file) return false;
            
            file.write(content.data(), content.size());
            file.close();
            
            if (!file) {
                fs::remove(temp_path);
                return false;
            }
            
            std::error_code ec;
            fs::rename(temp_path, path, ec);
            
            if (ec) {
                fs::remove(temp_path);
                return false;
            }
            
            return true;
            
        } catch (...) {
            std::error_code ec;
            fs::remove(temp_path, ec);
            return false;
        }
    }
    
    // Find files matching predicate
    static std::vector<fs::path> find_files(
        const fs::path& root,
        std::function<bool(const fs::directory_entry&)> predicate,
        bool recursive = true) {
        
        std::vector<fs::path> results;
        std::error_code ec;
        
        if (recursive) {
            for (const auto& entry : fs::recursive_directory_iterator(root,
                fs::directory_options::skip_permission_denied, ec)) {
                if (predicate(entry)) {
                    results.push_back(entry.path());
                }
            }
        } else {
            for (const auto& entry : fs::directory_iterator(root, ec)) {
                if (predicate(entry)) {
                    results.push_back(entry.path());
                }
            }
        }
        
        return results;
    }
    
    // Find files by extension
    static std::vector<fs::path> find_by_extension(
        const fs::path& root,
        const std::string& extension,
        bool recursive = true) {
        
        return find_files(root, [&](const fs::directory_entry& entry) {
            return entry.is_regular_file() && 
                   entry.path().extension() == extension;
        }, recursive);
    }
    
    // Find files by name pattern
    static std::vector<fs::path> find_by_pattern(
        const fs::path& root,
        const std::string& pattern,
        bool recursive = true) {
        
        return find_files(root, [&](const fs::directory_entry& entry) {
            return entry.is_regular_file() && 
                   entry.path().filename().string().find(pattern) != std::string::npos;
        }, recursive);
    }
    
    // Calculate directory size
    static std::uintmax_t directory_size(const fs::path& dir) {
        std::uintmax_t size = 0;
        std::error_code ec;
        
        for (const auto& entry : fs::recursive_directory_iterator(dir,
            fs::directory_options::skip_permission_denied, ec)) {
            
            if (entry.is_regular_file(ec) && !ec) {
                size += entry.file_size(ec);
            }
        }
        
        return size;
    }
    
    // Count files and directories
    struct DirectoryStats {
        std::size_t file_count = 0;
        std::size_t dir_count = 0;
        std::uintmax_t total_size = 0;
    };
    
    static DirectoryStats analyze_directory(const fs::path& dir) {
        DirectoryStats stats;
        std::error_code ec;
        
        for (const auto& entry : fs::recursive_directory_iterator(dir,
            fs::directory_options::skip_permission_denied, ec)) {
            
            if (entry.is_regular_file(ec)) {
                stats.file_count++;
                stats.total_size += entry.file_size(ec);
            } else if (entry.is_directory(ec)) {
                stats.dir_count++;
            }
        }
        
        return stats;
    }
    
    // Safe path resolution (prevents path traversal attacks)
    static std::optional<fs::path> resolve_safe(
        const fs::path& base,
        const fs::path& user_path) {
        
        try {
            auto abs_base = fs::canonical(base);
            auto abs_path = fs::weakly_canonical(abs_base / user_path);
            
            // Check if resolved path is within base
            auto [base_it, path_it] = std::mismatch(
                abs_base.begin(), abs_base.end(),
                abs_path.begin(), abs_path.end()
            );
            
            if (base_it != abs_base.end()) {
                return std::nullopt;  // Path escapes base
            }
            
            return abs_path;
            
        } catch (...) {
            return std::nullopt;
        }
    }
    
    // Copy directory with progress callback
    static bool copy_directory(
        const fs::path& source,
        const fs::path& destination,
        std::function<void(const fs::path&, std::uintmax_t)> progress = nullptr) {
        
        try {
            fs::create_directories(destination);
            
            for (const auto& entry : fs::recursive_directory_iterator(source)) {
                auto rel_path = fs::relative(entry.path(), source);
                auto dest_path = destination / rel_path;
                
                std::error_code ec;
                
                if (entry.is_directory()) {
                    fs::create_directories(dest_path, ec);
                } else if (entry.is_regular_file()) {
                    fs::create_directories(dest_path.parent_path(), ec);
                    fs::copy_file(entry.path(), dest_path,
                                 fs::copy_options::overwrite_existing, ec);
                    
                    if (progress) {
                        progress(rel_path, entry.file_size(ec));
                    }
                }
                
                if (ec) return false;
            }
            
            return true;
            
        } catch (...) {
            return false;
        }
    }
    
    // Clean empty directories
    static std::size_t remove_empty_directories(const fs::path& root) {
        std::size_t removed = 0;
        bool found_empty;
        
        do {
            found_empty = false;
            std::error_code ec;
            
            for (const auto& entry : fs::recursive_directory_iterator(root,
                fs::directory_options::skip_permission_denied, ec)) {
                
                if (entry.is_directory(ec) && fs::is_empty(entry.path(), ec)) {
                    fs::remove(entry.path(), ec);
                    if (!ec) {
                        removed++;
                        found_empty = true;
                    }
                }
            }
        } while (found_empty);
        
        return removed;
    }
    
    // Get files modified within time range
    static std::vector<fs::path> find_modified_since(
        const fs::path& root,
        std::chrono::system_clock::time_point since) {
        
        std::vector<fs::path> results;
        std::error_code ec;
        
        for (const auto& entry : fs::recursive_directory_iterator(root,
            fs::directory_options::skip_permission_denied, ec)) {
            
            if (entry.is_regular_file(ec)) {
                auto ftime = entry.last_write_time(ec);
                if (!ec) {
                    // Convert file_time to system_clock time
                    auto sctp = std::chrono::time_point_cast<std::chrono::system_clock::duration>(
                        ftime - fs::file_time_type::clock::now() + 
                        std::chrono::system_clock::now()
                    );
                    
                    if (sctp >= since) {
                        results.push_back(entry.path());
                    }
                }
            }
        }
        
        return results;
    }
    
    // Create unique filename
    static fs::path make_unique_filename(const fs::path& path) {
        if (!fs::exists(path)) {
            return path;
        }
        
        auto stem = path.stem();
        auto ext = path.extension();
        auto parent = path.parent_path();
        
        for (int i = 1; i < 10000; ++i) {
            auto new_path = parent / (stem.string() + "_" + std::to_string(i) + ext.string());
            if (!fs::exists(new_path)) {
                return new_path;
            }
        }
        
        throw std::runtime_error("Could not create unique filename");
    }
    
    // Format file size for display
    static std::string format_size(std::uintmax_t bytes) {
        const char* units[] = {"B", "KB", "MB", "GB", "TB"};
        int unit_index = 0;
        double size = static_cast<double>(bytes);
        
        while (size >= 1024.0 && unit_index < 4) {
            size /= 1024.0;
            unit_index++;
        }
        
        std::ostringstream oss;
        oss << std::fixed << std::setprecision(2) << size << " " << units[unit_index];
        return oss.str();
    }
    
    // Get file extension without dot
    static std::string get_extension_nodot(const fs::path& path) {
        auto ext = path.extension().string();
        if (!ext.empty() && ext[0] == '.') {
            return ext.substr(1);
        }
        return ext;
    }
    
    // Check if path is hidden (starts with dot on Unix)
    static bool is_hidden(const fs::path& path) {
        auto filename = path.filename().string();
        
        #ifdef _WIN32
        // On Windows, check file attributes
        DWORD attrs = GetFileAttributesW(path.c_str());
        return (attrs != INVALID_FILE_ATTRIBUTES) && 
               (attrs & FILE_ATTRIBUTE_HIDDEN);
        #else
        // On Unix, check if starts with dot
        return !filename.empty() && filename[0] == '.';
        #endif
    }
};
```

---

## 27. Complete Working Examples

### Example 1: Log File Rotator
```cpp
class LogRotator {
    fs::path log_file_;
    std::uintmax_t max_size_;
    int max_backups_;
    
public:
    LogRotator(const fs::path& log_file, 
               std::uintmax_t max_size_mb = 10,
               int max_backups = 5)
        : log_file_(log_file),
          max_size_(max_size_mb * 1024 * 1024),
          max_backups_(max_backups) {}
    
    void rotate_if_needed() {
        std::error_code ec;
        
        if (!fs::exists(log_file_, ec) || ec) {
            return;
        }
        
        auto size = fs::file_size(log_file_, ec);
        if (ec || size < max_size_) {
            return;
        }
        
        // Rotate existing backups
        for (int i = max_backups_ - 1; i >= 1; --i) {
            auto old_backup = get_backup_name(i);
            auto new_backup = get_backup_name(i + 1);
            
            if (fs::exists(old_backup)) {
                fs::rename(old_backup, new_backup, ec);
            }
        }
        
        // Move current log to .1
        fs::rename(log_file_, get_backup_name(1), ec);
        
        // Create new empty log
        std::ofstream(log_file_);
        
        std::cout << "Log rotated. Old logs preserved as backups.\n";
    }
    
    void write(const std::string& message) {
        rotate_if_needed();
        
        std::ofstream log(log_file_, std::ios::app);
        auto now = std::chrono::system_clock::now();
        auto time_t = std::chrono::system_clock::to_time_t(now);
        
        log << std::put_time(std::localtime(&time_t), "%Y-%m-%d %H:%M:%S")
            << " " << message << '\n';
    }
    
    void clean_old_backups() {
        std::error_code ec;
        
        for (int i = max_backups_ + 1; i <= max_backups_ + 10; ++i) {
            auto backup = get_backup_name(i);
            if (fs::exists(backup, ec)) {
                fs::remove(backup, ec);
            }
        }
    }
    
private:
    fs::path get_backup_name(int index) const {
        return fs::path(log_file_.string() + "." + std::to_string(index));
    }
};

// Usage
int main() {
    LogRotator logger("app.log", 1);  // 1MB max size
    
    for (int i = 0; i < 100000; ++i) {
        logger.write("Log message " + std::to_string(i));
    }
}
```

### Example 2: Smart Installer/Uninstaller
```cpp
class ApplicationInstaller {
    fs::path install_dir_;
    fs::path manifest_file_;
    std::vector<fs::path> installed_files_;
    
public:
    ApplicationInstaller(const fs::path& install_dir)
        : install_dir_(install_dir),
          manifest_file_(install_dir / ".manifest") {}
    
    bool install(const std::vector<std::pair<fs::path, fs::path>>& files) {
        // files = vector of (source, destination_relative)
        
        std::cout << "Installing to " << install_dir_ << "...\n";
        
        // Create install directory
        std::error_code ec;
        fs::create_directories(install_dir_, ec);
        if (ec) {
            std::cerr << "Failed to create install directory\n";
            return false;
        }
        
        installed_files_.clear();
        
        // Copy files
        for (const auto& [source, dest_rel] : files) {
            auto dest = install_dir_ / dest_rel;
            
            std::cout << "  Installing " << dest_rel << "...\n";
            
            fs::create_directories(dest.parent_path(), ec);
            if (ec) {
                std::cerr << "Failed to create directory for " << dest_rel << '\n';
                rollback();
                return false;
            }
            
            fs::copy_file(source, dest, fs::copy_options::overwrite_existing, ec);
            if (ec) {
                std::cerr << "Failed to copy " << dest_rel << '\n';
                rollback();
                return false;
            }
            
            installed_files_.push_back(dest);
        }
        
        // Save manifest
        save_manifest();
        
        std::cout << "Installation completed successfully!\n";
        return true;
    }
    
    bool uninstall() {
        load_manifest();
        
        if (installed_files_.empty()) {
            std::cerr << "No installation found\n";
            return false;
        }
        
        std::cout << "Uninstalling from " << install_dir_ << "...\n";
        
        // Remove files in reverse order
        for (auto it = installed_files_.rbegin(); it != installed_files_.rend(); ++it) {
            std::error_code ec;
            
            if (fs::exists(*it, ec)) {
                std::cout << "  Removing " << fs::relative(*it, install_dir_) << "...\n";
                fs::remove(*it, ec);
            }
        }
        
        // Remove empty directories
        FilesystemUtils::remove_empty_directories(install_dir_);
        
        // Remove manifest
        std::error_code ec;
        fs::remove(manifest_file_, ec);
        
        // Try to remove install directory if empty
        if (fs::is_empty(install_dir_, ec)) {
            fs::remove(install_dir_, ec);
        }
        
        std::cout << "Uninstallation completed!\n";
        return true;
    }
    
    bool verify() {
        load_manifest();
        
        std::cout << "Verifying installation...\n";
        bool all_ok = true;
        
        for (const auto& file : installed_files_) {
            std::error_code ec;
            if (!fs::exists(file, ec) || ec) {
                std::cout << "  MISSING: " << fs::relative(file, install_dir_) << '\n';
                all_ok = false;
            }
        }
        
        if (all_ok) {
            std::cout << "All files present.\n";
        }
        
        return all_ok;
    }
    
private:
    void save_manifest() {
        std::ofstream manifest(manifest_file_);
        for (const auto& file : installed_files_) {
            manifest << file.string() << '\n';
        }
    }
    
    void load_manifest() {
        installed_files_.clear();
        
        if (!fs::exists(manifest_file_)) {
            return;
        }
        
        std::ifstream manifest(manifest_file_);
        std::string line;
        while (std::getline(manifest, line)) {
            installed_files_.push_back(line);
        }
    }
    
    void rollback() {
        std::cout << "Installation failed. Rolling back...\n";
        
        for (const auto& file : installed_files_) {
            std::error_code ec;
            fs::remove(file, ec);
        }
        
        installed_files_.clear();
    }
};

// Usage
int main() {
    ApplicationInstaller installer("/opt/myapp");
    
    std::vector<std::pair<fs::path, fs::path>> files = {
        {"./bin/myapp", "bin/myapp"},
        {"./lib/libmyapp.so", "lib/libmyapp.so"},
        {"./share/icon.png", "share/icons/myapp.png"}
    };
    
    if (installer.install(files)) {
        installer.verify();
    }
    
    // Later...
    // installer.uninstall();
}
```

### Example 3: Development Environment Setup Tool
```cpp
class DevEnvironment {
    fs::path project_root_;
    
    struct ProjectStructure {
        std::vector<std::string> directories;
        std::map<std::string, std::string> files;  // path -> content
    };
    
public:
    DevEnvironment(const fs::path& project_root) 
        : project_root_(project_root) {}
    
    bool create_cpp_project(const std::string& project_name) {
        std::cout << "Creating C++ project: " << project_name << '\n';
        
        ProjectStructure structure = {
            // Directories
            {
                "src",
                "include/" + project_name,
                "tests",
                "build",
                "docs",
                "examples",
                ".vscode"
            },
            // Files with content
            {
                {"README.md", generate_readme(project_name)},
                {"CMakeLists.txt", generate_cmake(project_name)},
                {".gitignore", generate_gitignore()},
                {"src/main.cpp", generate_main_cpp(project_name)},
                {"include/" + project_name + "/version.h", generate_version_h(project_name)},
                {"tests/test_main.cpp", generate_test_cpp()},
                {".vscode/settings.json", generate_vscode_settings()},
                {"docs/API.md", "# API Documentation\n"}
            }
        };
        
        return create_structure(structure);
    }
    
    bool create_python_project(const std::string& project_name) {
        std::cout << "Creating Python project: " << project_name << '\n';
        
        ProjectStructure structure = {
            {
                project_name,
                "tests",
                "docs",
                ".vscode"
            },
            {
                {"README.md", generate_readme(project_name)},
                {"setup.py", generate_setup_py(project_name)},
                {"requirements.txt", ""},
                {".gitignore", generate_python_gitignore()},
                {project_name + "/__init__.py", "__version__ = '0.1.0'\n"},
                {project_name + "/main.py", generate_python_main()},
                {"tests/__init__.py", ""},
                {"tests/test_main.py", generate_python_test()}
            }
        };
        
        return create_structure(structure);
    }
    
private:
    bool create_structure(const ProjectStructure& structure) {
        try {
            // Create directories
            for (const auto& dir : structure.directories) {
                auto path = project_root_ / dir;
                std::cout << "  Creating " << dir << "/\n";
                fs::create_directories(path);
            }
            
            // Create files
            for (const auto& [file_path, content] : structure.files) {
                auto path = project_root_ / file_path;
                std::cout << "  Creating " << file_path << '\n';
                
                fs::create_directories(path.parent_path());
                std::ofstream file(path);
                file << content;
            }
            
            std::cout << "Project created successfully at " << project_root_ << '\n';
            return true;
            
        } catch (const std::exception& e) {
            std::cerr << "Error: " << e.what() << '\n';
            return false;
        }
    }
    
    std::string generate_readme(const std::string& name) {
        return "# " + name + "\n\n"
               "## Description\n\n"
               "Project description here.\n\n"
               "## Building\n\n"
               "```bash\n"
               "mkdir build && cd build\n"
               "cmake ..\n"
               "make\n"
               "```\n";
    }
    
    std::string generate_cmake(const std::string& name) {
        return "cmake_minimum_required(VERSION 3.10)\n"
               "project(" + name + " VERSION 0.1.0)\n\n"
               "set(CMAKE_CXX_STANDARD 17)\n"
               "set(CMAKE_CXX_STANDARD_REQUIRED ON)\n\n"
               "include_directories(include)\n\n"
               "add_executable(" + name + " src/main.cpp)\n";
    }
    
    std::string generate_gitignore() {
        return "build/\n"
               "*.o\n"
               "*.exe\n"
               "*.out\n"
               ".vscode/\n"
               ".vs/\n";
    }
    
    std::string generate_main_cpp(const std::string& name) {
        return "#include <iostream>\n"
               "#include \"" + name + "/version.h\"\n\n"
               "int main() {\n"
               "    std::cout << \"" + name + " v\" << VERSION << std::endl;\n"
               "    return 0;\n"
               "}\n";
    }
    
    std::string generate_version_h(const std::string& name) {
        std::string guard = name;
        std::transform(guard.begin(), guard.end(), guard.begin(), ::toupper);
        
        return "#ifndef " + guard + "_VERSION_H\n"
               "#define " + guard + "_VERSION_H\n\n"
               "#define VERSION \"0.1.0\"\n\n"
               "#endif\n";
    }
    
    std::string generate_test_cpp() {
        return "#define CATCH_CONFIG_MAIN\n"
               "// #include <catch2/catch.hpp>\n\n"
               "// TEST_CASE(\"Example test\") {\n"
               "//     REQUIRE(true);\n"
               "// }\n";
    }
    
    std::string generate_vscode_settings() {
        return "{\n"
               "    \"files.associations\": {\n"
               "        \"*.h\": \"cpp\",\n"
               "        \"*.cpp\": \"cpp\"\n"
               "    }\n"
               "}\n";
    }
    
    std::string generate_setup_py(const std::string& name) {
        return "from setuptools import setup, find_packages\n\n"
               "setup(\n"
               "    name='" + name + "',\n"
               "    version='0.1.0',\n"
               "    packages=find_packages(),\n"
               ")\n";
    }
    
    std::string generate_python_gitignore() {
        return "__pycache__/\n"
               "*.py[cod]\n"
               "*.so\n"
               ".venv/\n"
               "dist/\n"
               "build/\n"
               "*.egg-info/\n";
    }
    
    std::string generate_python_main() {
        return "def main():\n"
               "    print('Hello, World!')\n\n"
               "if __name__ == '__main__':\n"
               "    main()\n";
    }
    
    std::string generate_python_test() {
        return "import unittest\n"
               "# from myproject import main\n\n"
               "class TestMain(unittest.TestCase):\n"
               "    def test_example(self):\n"
               "        self.assertTrue(True)\n";
    }
};

// Usage
int main() {
    DevEnvironment env("/home/user/projects/mynewproject");
    env.create_cpp_project("mynewproject");
    // or
    // env.create_python_project("mynewproject");
}
```

---

## 28. Memory and Performance Cheat Sheet

### Memory Footprint
```cpp
sizeof(fs::path)                    // ~32 bytes (implementation-dependent)
sizeof(fs::directory_entry)         // ~40-80 bytes (includes cached metadata)
sizeof(fs::directory_iterator)      // ~16-32 bytes
sizeof(fs::recursive_directory_iterator) // ~40-60 bytes (includes stack)
```

### Performance Characteristics
```
Operation                           | Complexity | Notes
------------------------------------|------------|------------------------
path construction                   | O(1)       | String copy
path concatenation (/)              | O(n)       | Creates new path
path.filename()                     | O(1)       | Returns view/copy
path.extension()                    | O(1)       | Returns view/copy
lexically_normal()                  | O(n)       | String manipulation only
canonical()                         | O(syscalls)| Multiple filesystem accesses
exists()                            | O(1 syscall)| Single stat() call
file_size()                         | O(1 syscall)| Single stat() call
directory_iterator construction     | O(1 syscall)| opendir()
directory_iterator increment        | O(1 syscall)| readdir()
directory_entry.is_regular_file()   | O(1)       | Cached from readdir()
fs::is_regular_file(path)           | O(1 syscall)| New stat() call
copy_file()                         | O(filesize)| Read + write
recursive copy                      | O(n * filesize)| n = number of files
```

### Optimization Quick Wins
1. **Use directory_entry methods** instead of free functions (cached)
2. **Batch error code operations** to avoid exception overhead
3. **Group operations by directory** for cache locality
4. **Use lexically_normal()** instead of canonical() when possible
5. **Precompute absolute paths** if used multiple times
6. **Avoid redundant existence checks** - try the operation directly

---

## 29. Final Best Practices Summary

### DO's ✅
- Use `operator/` for path concatenation (portable)
- Use error code overloads in loops and expected failures
- Cache `fs::canonical()` results if reused
- Use `directory_entry` cached methods
- Check error codes explicitly
- Use RAII for resource management (temp files, etc.)
- Validate user-provided paths against base directories
- Use `weakly_canonical()` for paths that may not exist
- Test on both Windows and Unix if targeting both
- Handle symlinks explicitly (decide to follow or not)
- Use `.generic_string()` for portable output/logging
- Create parent directories before writing files
- Implement atomic writes (write to temp, then rename)
- Profile filesystem operations in performance-critical code

### DON'Ts ❌
- Don't check existence before every operation (TOCTOU)
- Don't concatenate paths with string operations
- Don't ignore error codes
- Don't use exceptions in destructors
- Don't assume directory iteration order
- Don't hard-code absolute paths
- Don't forget about symlink security implications
- Don't use `canonical()` on non-existent paths
- Don't call free functions on paths in tight loops
- Don't forget path length limits (especially Windows)
- Don't assume case sensitivity/insensitivity
- Don't mix string encodings (UTF-8, UTF-16)
- Don't perform filesystem operations in signal handlers
- Don't assume filesystem operations are atomic (except rename on same FS)

---

## 30. Complete Method Reference Table

### Path Methods (Observers)
| Method | Returns | Description | Throws |
|--------|---------|-------------|--------|
| `native()` | `native_string` | Platform-native format | No |
| `c_str()` | `const value_type*` | C-style string | No |
| `string()` | `std::string` | Convert to string | No |
| `wstring()` | `std::wstring` | Convert to wide string | No |
| `u8string()` | `std::u8string` | UTF-8 string | No |
| `u16string()` | `std::u16string` | UTF-16 string | No |
| `u32string()` | `std::u32string` | UTF-32 string | No |
| `generic_string()` | `std::string` | Always uses `/` | No |
| `generic_wstring()` | `std::wstring` | Wide version | No |
| `generic_u8string()` | `std::u8string` | UTF-8 generic | No |

### Path Methods (Decomposition)
| Method | Returns | Description | Example |
|--------|---------|-------------|---------|
| `root_name()` | `path` | Drive/network name | `C:` on Windows, `` on Unix |
| `root_directory()` | `path` | Root directory | `/` or `\` |
| `root_path()` | `path` | `root_name() + root_directory()` | `C:/` or `/` |
| `relative_path()` | `path` | Path without root | `home/user/file.txt` |
| `parent_path()` | `path` | Parent directory | `/home/user` |
| `filename()` | `path` | Last component | `file.txt` |
| `stem()` | `path` | Filename without extension | `file` |
| `extension()` | `path` | Last extension (with dot) | `.txt` |

### Path Methods (Queries)
| Method | Returns | Description |
|--------|---------|-------------|
| `empty()` | `bool` | True if path is empty |
| `has_root_name()` | `bool` | Has root name component |
| `has_root_directory()` | `bool` | Has root directory |
| `has_root_path()` | `bool` | Has root path |
| `has_relative_path()` | `bool` | Has relative path |
| `has_parent_path()` | `bool` | Has parent path |
| `has_filename()` | `bool` | Has filename |
| `has_stem()` | `bool` | Has stem |
| `has_extension()` | `bool` | Has extension |
| `is_absolute()` | `bool` | Is absolute path |
| `is_relative()` | `bool` | Is relative path |

### Path Methods (Modifiers)
| Method | Returns | Description | Example |
|--------|---------|-------------|---------|
| `clear()` | `void` | Clear path | `p.clear()` → `` |
| `make_preferred()` | `path&` | Use native separators | `/a/b` → `\a\b` (Windows) |
| `remove_filename()` | `path&` | Remove filename | `/a/b/c.txt` → `/a/b/` |
| `replace_filename(p)` | `path&` | Replace filename | `/a/b/c.txt` → `/a/b/d.txt` |
| `replace_extension(e)` | `path&` | Replace extension | `file.txt` → `file.md` |
| `swap(other)` | `void` | Swap with other path | - |

### Path Methods (Lexical Operations)
| Method | Returns | Description | Filesystem Access |
|--------|---------|-------------|-------------------|
| `lexically_normal()` | `path` | Normalize (remove `.` and `..`) | No |
| `lexically_relative(base)` | `path` | Relative path from base | No |
| `lexically_proximate(base)` | `path` | Proximate path | No |

### Filesystem Operations (Path Resolution)
| Function | Description | Requires Existence | Follows Symlinks |
|----------|-------------|-------------------|------------------|
| `absolute(p)` | Make absolute | No | N/A |
| `canonical(p)` | Resolve to canonical form | Yes | Yes |
| `weakly_canonical(p)` | Partial canonical | Partial | Yes (existing) |
| `relative(p, base)` | Relative path | No | No |
| `proximate(p, base)` | Proximate path | No | No |

### Filesystem Operations (Queries)
| Function | Returns | Description | Follows Symlinks |
|----------|---------|-------------|------------------|
| `exists(p)` | `bool` | Does path exist | Yes (default) |
| `is_regular_file(p)` | `bool` | Is regular file | Yes |
| `is_directory(p)` | `bool` | Is directory | Yes |
| `is_symlink(p)` | `bool` | Is symbolic link | No |
| `is_empty(p)` | `bool` | Is empty file/directory | Yes |
| `is_block_file(p)` | `bool` | Is block device | Yes |
| `is_character_file(p)` | `bool` | Is character device | Yes |
| `is_fifo(p)` | `bool` | Is named pipe | Yes |
| `is_socket(p)` | `bool` | Is socket | Yes |
| `is_other(p)` | `bool` | Is other type | Yes |
| `equivalent(p1, p2)` | `bool` | Same filesystem object | Yes |
| `file_size(p)` | `uintmax_t` | File size in bytes | Yes |
| `hard_link_count(p)` | `uintmax_t` | Number of hard links | Yes |
| `last_write_time(p)` | `file_time_type` | Last modification time | Yes |
| `status(p)` | `file_status` | Get file status | Yes |
| `symlink_status(p)` | `file_status` | Get symlink status | No |
| `space(p)` | `space_info` | Filesystem space info | - |
| `read_symlink(p)` | `path` | Target of symlink | No |

### Filesystem Operations (Modifications)
| Function | Description | Can Throw | Atomic |
|----------|-------------|-----------|--------|
| `copy(from, to, opts)` | Copy file/directory | Yes | No |
| `copy_file(from, to, opts)` | Copy single file | Yes | Platform-dependent |
| `copy_symlink(from, to)` | Copy symlink | Yes | Yes |
| `create_directory(p)` | Create directory | Yes | Yes |
| `create_directories(p)` | Create directory tree | Yes | No |
| `create_hard_link(target, link)` | Create hard link | Yes | Yes |
| `create_symlink(target, link)` | Create symlink | Yes | Yes |
| `create_directory_symlink(target, link)` | Create directory symlink | Yes | Yes |
| `current_path()` / `current_path(p)` | Get/set current directory | Yes | - |
| `permissions(p, prms, opts)` | Set permissions | Yes | Yes |
| `remove(p)` | Remove file/empty dir | Yes | Yes |
| `remove_all(p)` | Remove recursively | Yes | No |
| `rename(old, new)` | Rename/move | Yes | Yes (same FS) |
| `resize_file(p, size)` | Change file size | Yes | Yes |
| `temp_directory_path()` | Get temp directory | Yes | - |

### Copy Options Flags
| Flag | Description |
|------|-------------|
| `none` | Default behavior |
| `skip_existing` | Don't overwrite existing files |
| `overwrite_existing` | Overwrite existing files |
| `update_existing` | Overwrite if source is newer |
| `recursive` | Copy directories recursively |
| `copy_symlinks` | Copy symlinks as symlinks |
| `skip_symlinks` | Skip symlinks |
| `directories_only` | Copy directory structure only |
| `create_symlinks` | Create symlinks instead of copying |
| `create_hard_links` | Create hard links instead of copying |

### Directory Options Flags
| Flag | Description |
|------|-------------|
| `none` | Default (throw on permission denied) |
| `follow_directory_symlink` | Follow symlinks to directories |
| `skip_permission_denied` | Skip inaccessible directories |

### Permission Flags (Octal)
| Flag | Value | Description |
|------|-------|-------------|
| `owner_read` | 0400 | Owner has read permission |
| `owner_write` | 0200 | Owner has write permission |
| `owner_exec` | 0100 | Owner has execute permission |
| `owner_all` | 0700 | Owner has all permissions |
| `group_read` | 040 | Group has read permission |
| `group_write` | 020 | Group has write permission |
| `group_exec` | 010 | Group has execute permission |
| `group_all` | 070 | Group has all permissions |
| `others_read` | 04 | Others have read permission |
| `others_write` | 02 | Others have write permission |
| `others_exec` | 01 | Others have execute permission |
| `others_all` | 07 | Others have all permissions |
| `all` | 0777 | All permissions |
| `set_uid` | 04000 | Set user ID on execution |
| `set_gid` | 02000 | Set group ID on execution |
| `sticky_bit` | 01000 | Restricted deletion flag |

---

## 31. Platform-Specific Implementation Details

### Windows-Specific Considerations
```cpp
// Windows path peculiarities
fs::path win_path = "C:\\Users\\Name\\file.txt";  // Backslashes
fs::path win_unc = "\\\\server\\share\\file.txt"; // UNC paths
fs::path win_long = "\\\\?\\C:\\very\\long\\path..."; // Long path prefix

// Case insensitivity
fs::path p1 = "File.txt";
fs::path p2 = "file.txt";
// p1 == p2 on Windows (usually)

// Reserved names (cannot be used)
// CON, PRN, AUX, NUL, COM1-9, LPT1-9
// These fail even with extensions: "CON.txt" is invalid

// Path separator
char sep = fs::path::preferred_separator; // '\\' on Windows

// Permission model
// Windows uses simpler model: mostly just read-only flag
// Most perms values map to read-only or not

// Symlinks
// Require admin privileges (or Developer Mode on Win10+)
// SeCreateSymbolicLinkPrivilege needed
```

### Unix/Linux-Specific Considerations
```cpp
// Unix path conventions
fs::path unix_path = "/home/user/file.txt";  // Forward slashes

// Case sensitivity
fs::path p1 = "File.txt";
fs::path p2 = "file.txt";
// p1 != p2 on most Unix systems (ext4, etc.)

// Hidden files
// Files starting with '.' are hidden
fs::path hidden = ".config";

// Path separator
char sep = fs::path::preferred_separator; // '/' on Unix

// Full permission model
// All perms values are meaningful
// User, group, other with read, write, execute

// Symlinks
// Fully supported without special privileges
// Can link to files and directories

// Special files
// Block devices, character devices, FIFOs, sockets all supported
```

### macOS-Specific Considerations
```cpp
// macOS is Unix-based but has peculiarities

// HFS+ filesystem is case-insensitive but case-preserving
// APFS can be either case-sensitive or insensitive
// Test: fs::equivalent("file.txt", "File.txt")

// Extended attributes (xattr)
// Not directly accessible via std::filesystem
// Use platform APIs if needed

// Resource forks
// Legacy feature, mostly deprecated
// May cause issues with file sizes

// Application bundles (.app directories)
// Treated as directories by filesystem library
// OS treats them as files in Finder
```

---

## 32. Debugging and Troubleshooting Guide

### Common Error Messages and Solutions

```cpp
// Error: "No such file or directory"
// Causes:
// 1. Path doesn't exist
// 2. Typo in path
// 3. Wrong current directory
// Solution:
std::cout << "Current path: " << fs::current_path() << '\n';
std::cout << "Absolute path: " << fs::absolute(path) << '\n';
std::cout << "Exists: " << fs::exists(path) << '\n';

// Error: "Permission denied"
// Causes:
// 1. Insufficient permissions
// 2. File in use by another process
// 3. Parent directory not writable
// Solution:
auto status = fs::status(path);
std::cout << "Permissions: " << std::oct << static_cast<int>(status.permissions()) << '\n';
// Check parent directory permissions too

// Error: "Is a directory"
// Cause: Tried to open directory as file
// Solution:
if (fs::is_directory(path)) {
    std::cerr << "Path is a directory, not a file\n";
}

// Error: "Not a directory"
// Cause: Path component is a file, not directory
// Solution:
auto parent = path.parent_path();
if (!fs::is_directory(parent)) {
    std::cerr << "Parent is not a directory\n";
}

// Error: "File exists"
// Cause: Tried to create file that already exists
// Solution:
if (fs::exists(path)) {
    // Decide: overwrite, rename, or fail
    fs::remove(path);  // or
    path = make_unique_path(path);
}

// Error: "Cross-device link"
// Cause: rename() across different filesystems
// Solution:
try {
    fs::rename(src, dst);
} catch (const fs::filesystem_error& e) {
    if (e.code() == std::errc::cross_device_link) {
        // Fall back to copy + delete
        fs::copy(src, dst);
        fs::remove(src);
    }
}

// Error: "Directory not empty"
// Cause: Tried to remove non-empty directory with remove()
// Solution:
fs::remove_all(path);  // Instead of fs::remove(path)

// Error: "Too many open files"
// Cause: File descriptor leak
// Solution:
// Ensure files are closed properly
// Use RAII (std::ifstream, not manual FILE*)
// Check ulimit -n (Unix)

// Error: "Filename too long"
// Cause: Path exceeds system limit
// Solution:
#ifdef _WIN32
// Use long path prefix
fs::path long_path = "\\\\?\\" + path.string();
#endif
// Or break operation into shorter path segments
```

### Debugging Helper Functions
```cpp
class FilesystemDebugger {
public:
    static void print_path_info(const fs::path& path) {
        std::cout << "=== Path Info ===\n";
        std::cout << "Input: " << path << '\n';
        std::cout << "Absolute: " << fs::absolute(path) << '\n';
        
        std::error_code ec;
        std::cout << "Canonical: ";
        try {
            std::cout << fs::canonical(path) << '\n';
        } catch (...) {
            std::cout << "(cannot resolve)\n";
        }
        
        std::cout << "Exists: " << fs::exists(path, ec) << '\n';
        std::cout << "Is file: " << fs::is_regular_file(path, ec) << '\n';
        std::cout << "Is directory: " << fs::is_directory(path, ec) << '\n';
        std::cout << "Is symlink: " << fs::is_symlink(path, ec) << '\n';
        
        if (fs::exists(path, ec) && !ec) {
            auto status = fs::status(path);
            std::cout << "Permissions: " << std::oct 
                     << static_cast<int>(status.permissions()) << std::dec << '\n';
            
            if (fs::is_regular_file(path, ec)) {
                std::cout << "Size: " << fs::file_size(path, ec) << " bytes\n";
            }
        }
        
        std::cout << "Components:\n";
        std::cout << "  root_name: " << path.root_name() << '\n';
        std::cout << "  root_directory: " << path.root_directory() << '\n';
        std::cout << "  relative_path: " << path.relative_path() << '\n';
        std::cout << "  parent_path: " << path.parent_path() << '\n';
        std::cout << "  filename: " << path.filename() << '\n';
        std::cout << "  stem: " << path.stem() << '\n';
        std::cout << "  extension: " << path.extension() << '\n';
    }
    
    static void print_directory_contents(const fs::path& dir) {
        std::cout << "=== Directory Contents: " << dir << " ===\n";
        std::error_code ec;
        
        for (const auto& entry : fs::directory_iterator(dir, ec)) {
            std::cout << "  ";
            
            if (entry.is_directory(ec)) std::cout << "[DIR]  ";
            else if (entry.is_symlink(ec)) std::cout << "[LINK] ";
            else if (entry.is_regular_file(ec)) std::cout << "[FILE] ";
            else std::cout << "[????] ";
            
            std::cout << entry.path().filename();
            
            if (entry.is_regular_file(ec)) {
                std::cout << " (" << entry.file_size(ec) << " bytes)";
            }
            
            std::cout << '\n';
        }
        
        if (ec) {
            std::cout << "Error: " << ec.message() << '\n';
        }
    }
    
    static void trace_path_resolution(const fs::path& path) {
        std::cout << "=== Path Resolution Trace ===\n";
        std::cout << "Original: " << path << '\n';
        
        if (path.is_relative()) {
            std::cout << "Relative to: " << fs::current_path() << '\n';
            auto abs = fs::absolute(path);
            std::cout << "Absolute: " << abs << '\n';
            path = abs;
        }
        
        fs::path current = path.root_path();
        std::cout << "\nTracing components:\n";
        std::cout << "  " << current << " [ROOT]\n";
        
        for (const auto& component : path.relative_path()) {
            current /= component;
            std::error_code ec;
            
            std::cout << "  " << current;
            
            if (!fs::exists(current, ec)) {
                std::cout << " [DOES NOT EXIST]\n";
                break;
            } else if (fs::is_symlink(current, ec)) {
                auto target = fs::read_symlink(current, ec);
                std::cout << " [SYMLINK -> " << target << "]\n";
            } else if (fs::is_directory(current, ec)) {
                std::cout << " [DIR]\n";
            } else if (fs::is_regular_file(current, ec)) {
                std::cout << " [FILE]\n";
            }
        }
    }
};

// Usage
int main() {
    FilesystemDebugger::print_path_info("./test.txt");
    FilesystemDebugger::print_directory_contents(".");
    FilesystemDebugger::trace_path_resolution("../other/file.txt");
}
```

---

## 33. Testing Strategies

### Unit Testing Filesystem Code
```cpp
#include <gtest/gtest.h>

class FilesystemTest : public ::testing::Test {
protected:
    fs::path test_dir;
    
    void SetUp() override {
        // Create unique test directory
        test_dir = fs::temp_directory_path() / 
                   ("test_" + std::to_string(std::time(nullptr)));
        fs::create_directories(test_dir);
    }
    
    void TearDown() override {
        // Clean up
        std::error_code ec;
        fs::remove_all(test_dir, ec);
    }
    
    fs::path create_test_file(const std::string& name, const std::string& content = "") {
        auto path = test_dir / name;
        std::ofstream(path) << content;
        return path;
    }
    
    fs::path create_test_dir(const std::string& name) {
        auto path = test_dir / name;
        fs::create_directories(path);
        return path;
    }
};

TEST_F(FilesystemTest, FileCreation) {
    auto file = create_test_file("test.txt", "hello");
    
    EXPECT_TRUE(fs::exists(file));
    EXPECT_TRUE(fs::is_regular_file(file));
    EXPECT_EQ(fs::file_size(file), 5);
}

TEST_F(FilesystemTest, DirectoryIteration) {
    create_test_file("file1.txt");
    create_test_file("file2.txt");
    create_test_dir("subdir");
    
    int file_count = 0;
    int dir_count = 0;
    
    for (const auto& entry : fs::directory_iterator(test_dir)) {
        if (entry.is_regular_file()) file_count++;
        if (entry.is_directory()) dir_count++;
    }
    
    EXPECT_EQ(file_count, 2);
    EXPECT_EQ(dir_count, 1);
}

TEST_F(FilesystemTest, PathTraversalPrevention) {
    auto malicious_path = fs::path("..") / ".." / "etc" / "passwd";
    auto resolved = FilesystemUtils::resolve_safe(test_dir, malicious_path);
    
    EXPECT_FALSE(resolved.has_value());
}

TEST_F(FilesystemTest, AtomicWrite) {
    auto file = test_dir / "atomic.txt";
    
    EXPECT_TRUE(FilesystemUtils::write_file_atomic(file, "test data"));
    EXPECT_TRUE(fs::exists(file));
    
    auto content = FilesystemUtils::read_file_safe(file);
    EXPECT_TRUE(content.has_value());
    EXPECT_EQ(*content, "test data");
}

TEST_F(FilesystemTest, ErrorHandling) {
    auto non_existent = test_dir / "does_not_exist.txt";
    
    std::error_code ec;
    auto size = fs::file_size(non_existent, ec);
    
    EXPECT_TRUE(ec);
    EXPECT_EQ(ec, std::errc::no_such_file_or_directory);
}
```

---

## 34. Interview Questions & Answers

### Q1: What's the difference between `canonical()` and `weakly_canonical()`?
**A:** `canonical()` requires all path components to exist and resolves all symlinks, throwing if the path doesn't exist. `weakly_canonical()` resolves existing components and normalizes non-existing ones lexically, making it useful for paths that may not exist yet.

```cpp
// File exists: /home/user/real.txt -> symlink.txt points to it
fs::canonical("/home/user/symlink.txt");  // Returns /home/user/real.txt
weakly_canonical("/home/user/symlink.txt/nonexistent");  // Partially resolves
```

### Q2: Why should you use `directory_entry` methods instead of free functions?
**A:** `directory_entry` caches metadata retrieved during directory iteration, avoiding repeated syscalls. Free functions make new syscalls each time.

```cpp
// Bad: 3 syscalls per file
for (auto& e : fs::directory_iterator(dir)) {
    if (fs::is_regular_file(e.path())) {        // syscall
        auto size = fs::file_size(e.path());     // syscall
    }
}

// Good: Metadata already cached
for (auto& e : fs::directory_iterator(dir)) {
    if (e.is_regular_file()) {    // cached
        auto size = e.file_size(); // cached
    }
}
```

### Q3: How do you safely handle filesystem operations that might fail?
**A:** Use error code overloads instead of exceptions for expected failures, and always validate results.

```cpp
std::error_code ec;
fs::remove(path, ec);
if (ec) {
    if (ec == std::errc::no_such_file_or_directory) {
        // Already gone, that's fine
    } else {
        // Real error
        log_error(ec.message());
    }
}
```

### Q4: What's the TOCTOU problem and how do you avoid it?
**A:** Time-Of-Check-Time-Of-Use: checking a condition and then acting on it creates a race window. Solution: attempt the operation directly and handle errors.

```cpp
// WRONG: Race condition
if (fs::exists(path)) {
    fs::remove(path);  // File could be deleted between check and remove
}

// RIGHT: Just try it
std::error_code ec;
fs::remove(path, ec);
// Handle ec if you care
```

### Q5: How would you implement a safe file replacement?
**A:** Write to a temporary file, then atomically rename it.

```cpp
void safe_replace(const fs::path& target, const std::string& data) {
    auto temp = target;
    temp += ".tmp";
    
    // Write to temp
    std::ofstream(temp) << data;
    
    // Atomic replace (on same filesystem)
    fs::rename(temp, target);
}
```

---

## 35. Real-World Project Checklist

When using `<filesystem>` in production:

**✅ Error Handling**
- [ ] All operations use error codes or try-catch
- [ ] Errors are logged with context
- [ ] Failures have fallback strategies
- [ ] User-facing errors are human-readable

**✅ Security**
- [ ] User paths are validated (no traversal attacks)
- [ ] Symlinks are handled explicitly
- [ ] Permissions are checked before operations
- [ ] Sensitive files have restrictive permissions (0600)

**✅ Performance**
- [ ] directory_entry methods used instead of free functions
- [ ] canonical() results are cached when reused
- [ ] Bulk operations use error codes (not exceptions)
- [ ] Large directories are processed incrementally

**✅ Portability**
- [ ] operator/ used for path concatenation
- [ ] No hard-coded absolute paths
- [ ] Code tested on target platforms
- [ ] Path length limits considered
- [ ] Case sensitivity handled appropriately

**✅ Resource Management**
- [ ] RAII used for temp files
- [ ] No file descriptor leaks
- [ ] Large file operations are chunked
- [ ] Iterators don't outlive their scope

**✅ Testing**
- [ ] Unit tests with temp directories
- [ ] Error paths are tested
- [ ] Platform-specific behavior tested
- [ ] Edge cases covered (empty dirs, symlinks, etc.)

**✅ Documentation**
- [ ] Path assumptions documented
- [ ] Error conditions documented
- [ ] Platform limitations noted
- [ ] Usage examples provided

---

## 36. Further Learning Resources

### Official Documentation
- C++ Reference: https://en.cppreference.com/w/cpp/filesystem
- ISO C++17 Standard: [filesystems] chapter

### Books
- "C++17 - The Complete Guide" by Nicolai Josuttis (Chapter on filesystem)
- "Professional C++" by Marc Gregoire (Modern C++ features)

### Practice Projects
1. **File Organizer**: Categorize downloads by type
2. **Backup Tool**: Incremental backup with deduplication
3. **Build System**: Dependency-aware compilation
4. **Log Rotator**: Size-based rotation with compression
5. **Duplicate Finder**: Hash-based file deduplication
6. **Directory Diff**: Compare directory trees
7. **Project Template Generator**: Scaffold new projects
8. **File Synchronizer**: Two-way sync between directories

### Key Takeaways
- Filesystem operations are **expensive** - cache when possible
- **Race conditions** are real - design with concurrency in mind
- **Platform differences** matter - test everywhere
- **Error handling** is critical - never ignore failures
- **Security** is paramount - validate all user inputs
- **Performance** requires awareness - profile your code

---

## Summary

You now have complete knowledge of C++'s `<filesystem>` library:

**Core Concepts:**
- Path representation and manipulation
- Directory iteration (cached!)
- File queries and modifications
- Error handling strategies
- Platform-specific behaviors

**Advanced Topics:**
- Concurrency and thread safety
- Performance optimization
- Security considerations
- Real-world patterns
- Production-ready utilities

**Mastery:**
- Know when to use each operation
- Understand the tradeoffs
- Handle errors gracefully
- Write portable code
- Optimize for performance

The filesystem library is powerful but requires careful use. Always validate inputs, handle errors, test thoroughly, and profile performance-critical code. Build abstractions that make the common case easy and the edge cases safe.

**Happy coding!** 🚀
