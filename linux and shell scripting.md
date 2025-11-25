# Linux and Shell Scripting: Complete Guide

---

## 1. Basic Linux Commands

### 1.1 List Files
```bash
ls              # List files
ls -l           # Long listing with permissions
ls -a           # All files (including hidden)
ls -lh          # Human-readable sizes
```
*Lists all files and directories in the current directory, supports options for details and hidden files.*

### 1.2 Print Working Directory
```bash
pwd
```
*Shows the full path to your current directory.*

### 1.3 Change Directory
```bash
cd /path/to/directory
cd ~        # Home directory
cd ..       # Up one directory
```
*Changes to the specified directory. `..` is parent, `~` home.*

### 1.4 Create/Remove Directory
```bash
mkdir my_folder        # Create
touch file.txt         # Create file
rmdir empty_folder     # Remove empty folder
rm -r my_folder/       # Remove folder and contents
```
*Make/remove directories or files. rmdir only removes empty folders, rm -r removes recursively.*

### 1.5 Remove/Move/Copy Files
```bash
rm file.txt                                # Delete file
mv oldname.txt newname.txt                 # Rename or move
cp source.txt destination.txt              # Copy file
cp -r srcdir dstdir                        # Copy directory recursively
```
*Core file operations.*

### 1.6 File Information
```bash
touch new_file.txt    # Create empty file
stat file.txt         # File info (size, owner, modified time, etc)
file file.txt         # Display file type
```
*Get info about files/directories.*

### 1.7 Symbolic & Hard Links
```bash
ln file.txt hardlink.txt           # Hard link
ln -s file.txt symlink.txt         # Symbolic link
```
*Hard and symbolic links for advanced file references.*

### 1.8 Disk Usage
```bash
du -sh *        # Directory/file sizes
lsblk           # List block devices
```
*Disk usage and device management.*

### 1.9 See Manual Pages
```bash
man ls
```
*Read documentation for commands (quit with `q`).*

---

## 2. File Viewing and Editing

### 2.1 Display/Preview Files
```bash
cat file.txt                       # Print full file
head -n 5 file.txt                 # First 5 lines
tail -f file.txt                   # Last lines (and follow updates)
less file.txt                      # Scrollable view, q to quit
```
*Quickly display/monitor file contents.*

### 2.2 Edit Files
```bash
nano file.txt      # Simple editor
vim file.txt       # Advanced editor
```
*For simple/advanced in-terminal editing.*

---

## 3. File Permissions & Ownership

### 3.1 Change Permissions
```bash
chmod 755 script.sh            # Owner rwx, group rx, others rx
chmod +x script.sh             # Add execute
chmod -x script.sh             # Remove execute
```

### 3.2 Change Ownership
```bash
chown user:group file.txt      # Change owner and group
sudo chown root file.txt       # Change owner to root
```

### 3.3 View Permissions
```bash
ls -l
```
*See file permissions, owner, group.*

---

## 4. Searching, Filtering & Sorting

### 4.1 Search with Grep
```bash
grep 'pattern' file.txt                 # Basic search
grep -i 'linux' file.txt                # Case-insensitive
grep -r 'main' ./                       # Recursively search all files
grep -n 'TODO' file.txt                 # With line numbers
```

### 4.2 Find Files
```bash
find . -name "*.sh"                  # Find all .sh files
find /etc -type d                    # All directories in /etc
find . -mtime -1                     # Modified in last 24h
```

### 4.3 Sort/Unique/Text Processing
```bash
sort file.txt                        # Sort lines
uniq file.txt                        # Remove duplicate lines
cut -d":" -f1 /etc/passwd            # Extract columns
awk '{print $1, $3}' file.txt        # Print col1 and col3
sed 's/old/new/g' file.txt           # Find and replace
```

### 4.4 Count
```bash
wc -l file.txt    # lines
wc -w file.txt    # words
wc -c file.txt    # bytes
```

---

## 5. Process & System Management

### 5.1 Process Status & Signals
```bash
ps aux                             # All running processes
top                                # Dynamic process manager
htop                               # Enhanced top (if installed)
pkill -f script.sh                 # Kill by name
kill -9 12345                      # Force kill PID 12345
```

### 5.2 Disk & Memory
```bash
df -h              # Filesystems free/used space
du -sh *           # Size of all files and folders
free -h            # Free/used RAM
```

---

## 6. Networking

### 6.1 Interfaces & Configuration
```bash
ifconfig (ip a)                    # View addresses
ip link                            # List all interfaces
hostname                           # Show system hostname
uname -a                           # System info
```

### 6.2 Connectivity
```bash
ping google.com                    # Test host
traceroute google.com              # Trace packet route
netstat -tulnp                     # Ports in use
ss -tuln                           # Socket summary
```

### 6.3 Downloading
```bash
wget https://file
curl -O https://file
```

### 6.4 SSH & SCP
```bash
ssh user@host                   # Connect via SSH
scp file user@host:/path/       # Copy file remotely
```

---

## 7. User & Permissions Management

### 7.1 User Management
```bash
whoami      # Current user
id          # User/group info
sudo su     # Become superuser (root)
su - user   # Change to different user
passwd      # Change your password
```

### 7.2 Add/Delete Users or Groups
```bash
sudo adduser newuser          # Add user
sudo deluser someuser         # Remove user
sudo groupadd newgroup        # Add group
```

---

## 8. Package Management

### 8.1 Debian/Ubuntu
```bash
sudo apt update
sudo apt install packagename
sudo apt remove packagename
```

### 8.2 Red Hat/CentOS
```bash
sudo yum install packagename
sudo yum remove packagename
```

### 8.3 Arch Linux
```bash
sudo pacman -S packagename
```

---

## 9. Bash Scripting Essentials

### 9.1 Comments & Shebang
```bash
#!/bin/bash  # Script interpreter
# This is a comment
```

### 9.2 Variables
```bash
name="Linux"        # Define
echo $name          # Use
readonly x=5        # Read-only (constant)
export VAR1=abc     # Export variable to environment
```

### 9.3 Reading Input
```bash
read -p "Name: " username
echo "Hello $username"
```

### 9.4 Arithmetic & String
```bash
x=$((3 + 2))
echo $x                              # 5
let y=4*5
echo $y                              # 20
str="Hello, world"
len=${#str}
echo $len                            # prints string length
```

### 9.5 Conditionals
```bash
if [ "$x" -ge 5 ]; then
  echo "x is large"
fi

if [[ "$name" == "Linux" ]]; then
  echo "Name is Linux"
fi
```

### 9.6 Loops
```bash
for i in 1 2 3; do
  echo $i
done

for file in *.txt; do
  echo $file
done

while true; do
  echo "Press [CTRL+C] to stop"
done
```

### 9.7 Functions
```bash
greet() {
  echo "Hi $1"
}
greet "DevOps"
```

### 9.8 Script Arguments & Special Vars
```bash
$0     # Script name
$1     # First argument
$#     # Number of args
$@     # All args
$?     # Exit code of last command
$$     # Script PID
$!     # Last background PID
```

### 9.9 Arrays
```bash
arr=(one two three)
echo ${arr[1]}  # two
for el in "${arr[@]}"; do echo $el; done
```

### 9.10 Case Statement
```bash
case "$1" in
  start) echo "Start" ;;
  stop) echo "Stop" ;;
  *) echo "Unknown" ;;
esac
```

### 9.11 Output Redirection
```bash
ls > out.txt 2> err.txt
command >> append.log         # Append output
```

### 9.12 Command Substitution
```bash
today=$(date)
echo $today
```

### 9.13 Error Handling & Exiting
```bash
command || { echo "Error!"; exit 1; }
set -e     # Exit on any error
trap 'echo Failed!; exit 1' ERR
```

### 9.14 Advanced Text Processing
```bash
awk '{print $1}' file.txt         # Print 1st column
sed 's/foo/bar/g' file.txt        # Replace foo with bar
cut -d":" -f3 /etc/passwd         # Extract 3rd field
tr a-z A-Z < file.txt             # Lowercase to uppercase
join f1.txt f2.txt                # Join two files
paste f1.txt f2.txt               # Paste columns together
uniq file.txt                     # Deduplicate lines
diff f1.txt f2.txt                # Differences between files
```

### 9.15 Schedule Tasks
```bash
crontab -l                # List cron jobs
crontab -e                # Edit scheduled jobs
at now + 1 minute         # Schedule a task to run once
```

### 9.16 Traps: Handle Signals
```bash
trap 'echo Signal received!' SIGINT
```

### 9.17 Debugging
```bash
bash -x script.sh         # Debug mode
set -x                    # Enable for current session
```

---

## 10. Miscellaneous Useful Commands

```bash
echo $PATH                # Current PATH variable
history                   # Command history
alias gs='git status'     # Define alias
unalias gs                # Remove alias
ps -ef | grep python      # Find processes
pushd . && cd /tmp        # Go, then return with popd
popd
locate filename           # Fast file search (needs updatedb)
mount /dev/sdb1 /mnt      # Mount a drive
umount /mnt               # Unmount drive
ln -s /path/to/target linkname     # Sym-link
```

---

## 11. Tips & Keyboard Shortcuts

- `Ctrl + C` – Kill current command
- `Ctrl + Z` – Suspend command (bg/fg to resume)
- `Ctrl + D` – Logout of terminal
- `Tab` – Autocomplete files/commands
- `Ctrl + R` – Reverse search command history
- `Ctrl + A/E` – Go to line start/end
- `Ctrl + U/K` – Delete to start/end
- `!!` – Run last command
- `!$` – Last argument of previous command

---

## Resources
- man [command] : Get help for any command
- https://www.gnu.org/software/bash/manual/bash.html
- https://cheatography.com/davechild/cheat-sheets/linux-command-line/
- https://tldp.org/HOWTO/Bash-Prog-Intro-HOWTO.html
- https://devhints.io/bash

---
