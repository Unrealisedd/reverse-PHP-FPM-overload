# Reverse PHP-FPM Overload - A Stealthy PHP Persistence Technique  

## Introduction  
**Reverse PHP-FPM Overload** is a novel persistence technique that abuses PHP-FPM's worker pool to maintain a **memory-resident backdoor** without leaving traces on disk. This method ensures persistence by dynamically reloading the backdoor even after a PHP-FPM restart, making it **extremely difficult to detect** using traditional forensic methods.  

By modifying PHP-FPM’s process pool, an attacker can **ensure that a specific worker executes a hidden payload** when triggered via a seemingly normal request. This method is designed to:  
- **Avoid traditional file-based detection** (the payload never touches disk).  
- **Persist across PHP-FPM restarts** through systemd/init modifications.  
- **Evade common security tools** by operating entirely in memory.  

This write-up explores how the technique works, how it can be improved, and what defenders can do to detect or mitigate it.  

---

## Concept Breakdown  

### 🛠️ How Reverse PHP-FPM Overload Works  
1. **Hijacking PHP-FPM Worker Processes**  
   - PHP-FPM manages multiple worker processes to handle incoming PHP requests.  
   - By injecting a **hidden payload into one worker**, attackers can ensure execution upon a specific trigger (e.g., an invalid API call).  

2. **Memory-Only Execution**  
   - The payload **never writes to disk**, avoiding detection by antivirus or integrity monitoring tools.  
   - This is achieved by leveraging **opcache abuse, shared memory injection, or process hijacking**.  

3. **Automatic Reloading via systemd/init**  
   - Even if PHP-FPM restarts, the payload **reloads dynamically** via a **hidden systemd or init script**.  
   - This ensures persistence without requiring additional files.  

---

## 🔥 Advanced Techniques for Stealthier Execution  

### 1️⃣ **Hiding in PHP-FPM’s Shared Memory**  
- Instead of using disk-based persistence, attackers can **store payloads in shared memory** (`shmop_open()`, `sysvshm`).  
- This allows **execution across PHP-FPM worker restarts** without ever writing malicious code to a file.  

### 2️⃣ **Abusing Opcache for Memory Execution**  
- Opcache can be manipulated via `opcache_compile_file()` to cache a malicious script in memory.  
- Attackers can also **invalidate and replace opcache entries** dynamically, ensuring a memory-resident shell.  

### 3️⃣ **Hijacking PHP-FPM Worker Pools**  
- A separate worker pool (`php-fpm.d/stealth.conf`) can be created to execute only when a **secret API request is received**.  
- The pool listens on an **internal Unix socket**, making it invisible to external monitoring.  

### 4️⃣ **Triggering via System Signals**  
- Instead of using a standard HTTP request, the attacker can **send a Unix signal** to activate the payload:  
  - Example: `kill -USR1 $(pidof php-fpm)`  
  - The backdoor only executes when **a specific signal is received**, making detection even harder.  

---

## ⚠️ Detection Challenges  
Reverse PHP-FPM Overload is **exceptionally difficult to detect** due to the following reasons:  

| Challenge                  | Explanation |
|----------------------------|-------------|
| **No files on disk**       | The payload is never stored in traditional locations like `/var/www/html`. |
| **Runs in memory**         | Memory-only execution means forensic tools that rely on file hashes won’t find it. |
| **Evades WAFs**           | The trigger request appears as a **harmless API call**, avoiding traditional rule-based detection. |
| **Hidden in PHP-FPM’s config** | Attackers can modify worker pools and systemd scripts to reload the payload secretly. |

---

## 🛡️ Detection & Mitigation Strategies  

### ✅ **How to Detect Reverse PHP-FPM Overload**  
1. **Monitor PHP-FPM Worker Behavior**  
   - Look for **long-running worker processes** that shouldn’t exist.  
   - Use `strace` to monitor suspicious system calls from PHP-FPM workers.  

2. **Analyze Opcache Entries**  
   - Run `opcache_get_status(true)` and check for **unexpected cached scripts**.  
   - Regularly clear opcache to remove potential injected payloads.  

3. **Check for Hidden Worker Pools**  
   - Run `php-fpm -tt` to inspect worker pool configurations for unauthorized modifications.  

### 🚨 **How to Mitigate the Attack**  
✅ **Use Read-Only Filesystem for PHP Configs**: Prevent unauthorized changes to `php-fpm.conf`.  
✅ **Restrict Opcache Execution**: Disable `opcache_compile_file()` and regularly flush opcache.  
✅ **Monitor for Systemd Modifications**: Use `auditctl` to track changes to `/etc/systemd/system/php-fpm.service.d/`.  

---

## 📜 Proof of Concept (PoC) Outline  

This technique can be tested in a controlled environment using the following steps:  

1. **Setup a PHP-FPM Server**  
   - Install and configure PHP-FPM with a standard worker pool.  

2. **Inject a Memory-Only Payload**  
   - Use `opcache_compile_file()` or `shmop_open()` to store a hidden backdoor.  

3. **Trigger Execution via a Secret Request**  
   - Send a crafted HTTP request (`/api/fake`) to activate the payload.  

4. **Ensure Persistence via Systemd/Init**  
   - Modify `php-fpm.service` to **reload the backdoor on restart**.  

---

## 📌 Conclusion  
Reverse PHP-FPM Overload is a **stealthy persistence method** that leverages PHP-FPM's architecture to maintain an **in-memory backdoor**. By using **shared memory, opcache, and systemd hijacking**, attackers can achieve persistence without touching disk, making detection extremely difficult.  

**Security teams should monitor PHP-FPM worker behavior, track opcache modifications, and audit systemd/init scripts to detect and mitigate this threat.**  

🚀 **Future research could explore kernel-level rootkits or syscall hooking to make detection even harder.**  

---

## 📚 References  
- [PHP-FPM Documentation](https://www.php.net/manual/en/install.fpm.php)  
- [Opcache Internals](https://www.php.net/manual/en/book.opcache.php)  
- [Memory-Based Malware Techniques](https://research.security)  

---

**Disclaimer:** This research is for educational purposes only. Unauthorized use against live systems is illegal.  
