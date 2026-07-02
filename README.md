# Android Reverse Engineering Toolkit

[![GitHub stars](https://img.shields.io/github/stars/ykrishhh/android-reverse-engineering?style=social)](https://github.com/ykrishhh/android-reverse-engineering)
[![GitHub forks](https://img.shields.io/github/forks/ykrishhh/android-reverse-engineering?style=social)](https://github.com/ykrishhh/android-reverse-engineering/network/members)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](https://opensource.org/licenses/MIT)
[![Platform](https://img.shields.io/badge/Platform-Android-3DDC84?logo=android)](https://www.android.com/)
[![Frida](https://img.shields.io/badge/Dynamic%20Analysis-Frida-black)](https://frida.re)
[![Xposed](https://img.shields.io/badge/Framework-Xposed-yellow)](https://github.com/rovo89/Xposed)
[![Discord](https://img.shields.io/badge/Discord-Community-5865F2?logo=discord)](https://discord.gg/android-re)

> Android reverse engineering from static analysis to dynamic hooking — jadx, apktool, Frida, Xposed, Smali patching, native library analysis, and anti-debug bypasses.

Everything I wish I had when I started reverse engineering Android apps. Practical workflows, copy-paste code snippets, and tool configs that actually work.

---

## Table of Contents

- [Setup & Tools](#setup--tools)
- [APK Analysis Workflow](#apk-analysis-workflow)
- [Static Analysis](#static-analysis)
- [Smali Basics](#smali-basics)
- [Frida Dynamic Instrumentation](#frida-dynamic-instrumentation)
- [Xposed Framework](#xposed-framework)
- [Kernel Hooking Concepts](#kernel-hooking-concepts)
- [DEX File Format](#dex-file-format)
- [Native Library Analysis](#native-library-analysis)
- [Anti-Debug Bypasses](#anti-debug-bypasses)
- [Practice Apps](#practice-apps)
- [Resources](#resources)
- [Disclaimer](#disclaimer)

---

## Setup & Tools

Essential tools for Android reverse engineering on any platform.

### Core Tools Installation

```bash
# --- Linux / macOS ---

# jadx — Dex to Java decompiler
brew install jadx          # macOS
sudo apt install jadx      # Debian/Ubuntu

# apktool — APK resource decoder and repackager
brew install apktool       # macOS
sudo apt install apktool   # Debian/Ubuntu

# Android SDK (adb, dexdump, etc.)
# Download from https://developer.android.com/studio

# Frida
pip install frida-tools frida

# dex2jar
# Download from https://github.com/pxb1988/dex2jar/releases

# --- Termux (Android) ---
pkg install jadx apktool dex2jar
pip install frida-tools frida objection
```

### Recommended Tools Matrix

| Tool | Purpose | Install |
|------|---------|---------|
| **jadx** | Decompile DEX → readable Java source | `brew install jadx` |
| **apktool** | Decode/rebuild APK resources, Smali | `brew install apktool` |
| **Frida** | Dynamic instrumentation and hooking | `pip install frida-tools` |
| **Xposed Framework** | System-level hooking without modifying APK | [GitHub](https://github.com/rovo89/Xposed) |
| **Objection** | Frida-powered mobile security toolkit | `pip install objection` |
| **dex2jar** | Convert DEX to JAR for Java analysis | [GitHub](https://github.com/pxb1988/dex2jar) |
| **JEB Pro** | Commercial Android decompiler (AST, Smali, Java) | [Website](https://www.pnfsoftware.com/) |
| **Ghidra** | Free RE suite — decompile native .so libraries | [GitHub](https://github.com/NationalSecurityAgency/ghidra) |
| **IDA Pro** | Industry-standard disassembler for native code | [Hex-Rays](https://hex-rays.com/ida-pro/) |
| **Burp Suite** | HTTP/HTTPS proxy for API interception | [PortSwigger](https://portswigger.net/burp) |

---

## APK Analysis Workflow

The step-by-step process I follow for every APK. Works every time.

```
APK File
    │
    ├─→ 1. RENAME to .zip, EXTRACT
    │      └─→ AndroidManifest.xml, classes.dex, lib/, res/
    │
    ├─→ 2. DECOMPILE with jadx
    │      └─→ Readable Java source code
    │
    ├─→ 3. DECODE RESOURCES with apktool
    │      └─→ Smali code, decoded XML, layouts
    │
    ├─→ 4. ANALYZE with dex2jar + JD-GUI
    │      └─→ Browse decompiled JAR interactively
    │
    ├─→ 5. HOOK at runtime with Frida
    │      └─→ Bypass SSL pinning, dump data, trace calls
    │
    └─→ 6. PATCH Smali and REBUILD
           └─→ apktool b modified_apk/ -o patched.apk
```

### Quick Start Commands

```bash
# Decompile to Java source
jadx -d output/ target.apk

# Decode to Smali + resources
apktool d target.apk -o decoded/

# Convert DEX to JAR
d2j-dex2jar target.apk -o target-d2j.jar

# Rebuild modified APK
apktool b decoded/ -o patched.apk

# Sign the rebuilt APK
apksigner sign --ks my-key.keystore patched.apk

# Install on device
adb install patched.apk
```

---

## Static Analysis

Reading and understanding code without execution.

### Manifest Analysis

```bash
# Decode AndroidManifest.xml
apktool d target.apk
cat decoded/AndroidManifest.xml
```

Key items to inspect:
- **Permissions**: `uses-permission` declarations reveal app capabilities
- **Exported components**: `android:exported="true"` — accessible to other apps
- **Intent filters**: define which intents trigger components
- **Debuggable flag**: `android:debuggable="true"` enables attachment
- **Network security config**: cleartext traffic rules

### jadx Deep Dive

```bash
# Decompile with resources
jadx --deobf -d output/ target.apk

# Search for specific string across all classes
grep -r "api_key\|secret\|password\|token" output/sources/

# Find hardcoded URLs
grep -r "http://" output/sources/ | grep -v "https://"
```

### Hidden API Enumeration

```bash
# List all exported activities, services, receivers
aapt dump badging target.apk | grep -E "activity|service|receiver"

# Full permission dump
aapt dump permissions target.apk
```

---

## Smali Basics

Smali is the assembly language for Android's Dalvik/ART virtual machine. Learning Smali is the difference between "I can read the code" and "I can change the code."

### Smali Syntax Reference

```smali
# Class declaration
.class public Lcom/example/MyClass;
.super Ljava/lang/Object;
.source "MyClass.java"

# Method signature
# method_name(param1_type, param2_type)return_type
.method public login(Ljava/lang/String;Ljava/lang/String;)Z
    .locals 4
    # .locals defines register count (v0..v3 + p0..p1)
    # p0 = "this" (for non-static methods)
    # p1, p2 = method parameters

    # String assignment
    const-string v0, "admin"

    # Method invocation
    invoke-virtual {p1, v0}, Ljava/lang/String;->equals(Ljava/lang/Object;)Z
    move-result v1

    # Conditional branch
    if-eqz v1, :fail

    # Return true
    const/4 v0, 0x1
    return v0

    :fail
    const/4 v0, 0x0
    return v0
.end method
```

### Common Smali Instructions

| Instruction | Meaning |
|-------------|---------|
| `const-string v0, "text"` | Load string into register |
| `const/4 v0, 0x1` | Load integer constant |
| `invoke-virtual {v0, v1}, L...;->method(L...;)V` | Call instance method |
| `invoke-static {v0}, L...;->method()V` | Call static method |
| `move-result v0` | Move return value to register |
| `return v0` | Return value from method |
| `return-void` | Return void |
| `if-eqz v0, :label` | Branch if register equals zero |
| `if-nez v0, :label` | Branch if register not zero |
| `goto :label` | Unconditional jump |
| `iget-object v0, p0, L...;->field:L...;` | Read object field |
| `iput v0, p0, L...;->field:I` | Write to int field |

### Practical: Bypass Login Check (Smali Patch)

```smali
# BEFORE (original login method):
.method public checkCredentials(Ljava/lang/String;)Z
    .locals 2
    invoke-virtual {p0, p1}, Lcom/app/Auth;->validate(Ljava/lang/String;)Z
    move-result v0
    return v0
.end method

# AFTER (patched — always return true):
.method public checkCredentials(Ljava/lang/String;)Z
    .locals 2
    const/4 v0, 0x1
    return v0
.end method
```

---

## Frida Dynamic Instrumentation

Frida injects JavaScript into running processes for real-time hooking and tracing. This is the tool I reach for first on almost every engagement.

### Setup

```bash
pip install frida-tools frida

# Verify server is running on device
adb push frida-server-<version>-android-<arch> /data/local/tmp/
adb shell "chmod 755 /data/local/tmp/frida-server"
adb shell "/data/local/tmp/frida-server &"
```

### SSL Pinning Bypass

```javascript
// ssl_bypass.js — Bypass SSL certificate pinning
Java.perform(function() {
    var TrustManagerImpl = Java.use("com.android.org.conscrypt.TrustManagerImpl");
    TrustManagerImpl.verifyChain.implementation = function(untrustedChain, trustAnchorChain, host, clientAuth, ocspData, tlsSctData) {
        console.log("[+] SSL pinning bypassed for: " + host);
        return untrustedChain;
    };

    var SSLContext = Java.use("javax.net.ssl.SSLContext");
    SSLContext.init.overload("[Ljavax.net.ssl.KeyManager;", "[Ljavax.net.ssl.TrustManager;", "java.security.SecureRandom").implementation = function(km, tm, sr) {
        console.log("[+] SSLContext initialized with custom TrustManager");
        this.init(km, tm, sr);
    };

    // OkHttp3 pinning
    try {
        var CertificatePinner = Java.use("okhttp3.CertificatePinner");
        CertificatePinner.check.overload("java.lang.String", "java.util.List").implementation = function(hostname, peerCertificates) {
            console.log("[+] OkHttp3 pinning bypassed for: " + hostname);
        };
    } catch(e) {}
});
```

```bash
frida -U -l ssl_bypass.js -f com.target.app
```

### Method Tracing

```javascript
// trace_methods.js — Log all calls to a class
Java.perform(function() {
    var TargetClass = Java.use("com.target.app.AuthManager");

    var methods = TargetClass.class.getDeclaredMethods();
    methods.forEach(function(method) {
        var methodName = method.getName();
        TargetClass[methodName].overloads.forEach(function(overload) {
            overload.implementation = function() {
                console.log("[*] " + methodName + "(" + Array.from(arguments).join(", ") + ")");
                var result = this[methodName].apply(this, arguments);
                console.log("[*] " + methodName + " returned: " + result);
                return result;
            };
        });
    });
});
```

### String Decryption

```javascript
// decrypt_strings.js — Hook encryption methods and log plaintext
Java.perform(function() {
    var CryptoUtils = Java.use("com.target.app.CryptoUtils");
    CryptoUtils.decrypt.overload("java.lang.String").implementation = function(encrypted) {
        var result = this.decrypt(encrypted);
        console.log("[DECRYPT] " + encrypted + " -> " + result);
        return result;
    };
});
```

### Root Detection Bypass

```javascript
// root_bypass.js — Circumvent common root checks
Java.perform(function() {
    var RootDetection = Java.use("com.target.app.Security");

    // Bypass isRooted()
    RootDetection.isRooted.implementation = function() {
        console.log("[+] isRooted() forced false");
        return false;
    };

    // Bypass SafetyNet / Play Integrity
    try {
        var SafetyNet = Java.use("com.google.android.gms.safetynet.SafetyNetApi");
        SafetyNet.attest.overload().implementation = function() {
            console.log("[+] SafetyNet attest bypassed");
            return null;
        };
    } catch(e) {}
});
```

### Objection — Frida Made Easy

```bash
pip install objection

# Start objection session
objection -g com.target.app explore

# Bypass SSL pinning
android sslpinning disable

# Root detection bypass
android root disable

# List activities
android hooking list activities

# Monitor file access
android hooking watch file_access
```

---

## Xposed Framework

Xposed allows system-wide hooking by modifying the Android runtime (ART) at the framework level. LSPosed is what you want — the original Xposed is basically dead.

### Installation (Rooted Devices)

```bash
# Install LSPosed (modern Xposed fork)
# Download from https://github.com/LSPosed/LSPosed/releases

# Install via recovery or adb
adb push LSPosed-v1.9.2.zip /sdcard/
# Flash via TWRP recovery

# Or use Magisk Manager to install Zygisk + LSPosed module
```

### Sample Xposed Module

```java
// Example: Intercept WebView URL loading
package com.example.xposedhook;

import de.robv.android.xposed.IXposedHookLoadPackage;
import de.robv.android.xposed.XC_MethodHook;
import de.robv.android.xposed.XposedBridge;
import de.robv.android.xposed.XposedHelpers;
import de.robv.android.xposed.callbacks.XC_LoadPackage;

public class WebViewHook implements IXposedHookLoadPackage {
    @Override
    public void handleLoadPackage(XC_LoadPackage.LoadPackageParam lpparam) {
        if (!lpparam.packageName.equals("com.target.app")) return;

        // Hook WebView.loadUrl()
        XposedHelpers.findAndHookMethod(
            "android.webkit.WebView",
            lpparam.classLoader,
            "loadUrl",
            String.class,
            new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) {
                    String url = (String) param.args[0];
                    XposedBridge.log("[WebView] Loading: " + url);

                    // Redirect to our proxy
                    if (url.contains("target.com")) {
                        param.args[0] = url.replace("target.com", "attacker-proxy.com");
                    }
                }
            }
        );
    }
}
```

### Hook SharedPreferences

```java
// Intercept SharedPreferences reads
XposedHelpers.findAndHookMethod(
    "android.app.SharedPreferencesImpl",
    lpparam.classLoader,
    "getString",
    String.class, String.class,
    new XC_MethodHook() {
        @Override
        protected void afterHookedMethod(MethodHookParam param) {
            String key = (String) param.args[0];
            String value = (String) param.getResult();
            XposedBridge.log("[SharedPrefs] " + key + " = " + value);
        }
    }
);
```

---

## Kernel Hooking Concepts

For deep system-level instrumentation when userspace hooks aren't enough. Most people won't need this, but when you do, there's no substitute.

### Linux Kernel Tracepoints

```c
// Example: Using kprobes to trace system calls
#include <linux/kprobes.h>
#include <linux/module.h>

static int handler_pre(struct kprobe *p, struct pt_regs *regs) {
    printk(KERN_INFO "[TRACE] openat called with arg: %s\n",
           (char *)regs->regs[1]);
    return 0;
}

static struct kprobe kp = {
    .symbol_name = "do_sys_openat2",
    .pre_handler = handler_pre,
};

static int __init trace_init(void) {
    return register_kprobe(&kp);
}

static void __exit trace_exit(void) {
    unregister_kprobe(&kp);
}
```

### eBPF (Extended Berkeley Packet Filter)

```c
// eBPF program to trace file opens (BCC syntax)
#include <uapi/linux/ptrace.h>
#include <linux/sched.h>

struct event_t {
    u32 pid;
    char filename[256];
};

BPF_PERF_OUTPUT(events);

int trace_open(struct pt_regs *ctx) {
    struct event_t event = {};
    event.pid = bpf_get_current_pid_tgid() >> 32;
    bpf_probe_read_user_str(&event.filename, sizeof(event.filename),
                            (void *)PT_REGS_PARM1(ctx));
    events.perf_submit(ctx, &event, sizeof(event));
    return 0;
}
```

### Concepts Summary

| Layer | Technique | Tool |
|-------|-----------|------|
| Userspace | Function hooking, JNI interception | Frida, Xposed |
| Framework | ART method replacement | LSPosed modules |
| Native (ELF) | GOT/PLT patching, inline hooks | Frida Gadget, Substrate |
| Kernel | kprobes, tracepoints, eBPF | bcc, bpftrace |
| Hardware | JTAG/SWD, memory dumps | OpenOCD, JTAGulator |

---

## DEX File Format

Understanding the Dalvik Executable format is essential for advanced patching. Most people skip this and it bites them later.

### DEX Structure Overview

```
DEX File
├── Header (magic, checksum, SHA-1, file size)
├── String IDs (all string literals)
├── Type IDs (class/interface references)
├── Proto IDs (method prototypes)
├── Field IDs (field references)
├── Method IDs (method references)
├── Class Definitions (one per class)
│   ├── class_idx
│   ├── access_flags
│   ├── superclass_idx
│   ├── source_file_idx
│   ├── class_data
│   │   ├── static_fields
│   │   ├── instance_fields
│   │   ├── direct_methods
│   │   └── virtual_methods
│   └── code_item (bytecode)
├── Data Section (annotations, etc.)
└── Map List (section offsets)
```

### Parsing DEX with Python

```python
#!/usr/bin/env python3
"""Minimal DEX header parser"""
import struct

def parse_dex_header(filepath):
    with open(filepath, 'rb') as f:
        magic = f.read(8)
        checksum = struct.unpack('<I', f.read(4))[0]
        signature = f.read(20)
        file_size = struct.unpack('<I', f.read(4))[0]
        header_size = struct.unpack('<I', f.read(4))[0]
        endian_tag = struct.unpack('<I', f.read(4))[0]

        print(f"Magic: {magic}")
        print(f"Checksum: 0x{checksum:08x}")
        print(f"File size: {file_size} bytes")
        print(f"Header size: {header_size} bytes")
        print(f"Endian: {'Little' if endian_tag == 0x12345678 else 'Big'}")

        # String IDs section
        f.seek(56)
        string_ids_size = struct.unpack('<I', f.read(4))[0]
        string_ids_off = struct.unpack('<I', f.read(4))[0]
        print(f"String IDs: {string_ids_size} entries at offset {string_ids_off}")

parse_dex_header("target_classes.dex")
```

### Using dexdump

```bash
# Android SDK dexdump
dexdump -d target.dex           # Disassemble
dexdump -f target.dex           # Full dump
dexdump -l plain target.dex     # Human-readable
```

---

## Native Library Analysis

Most security-critical code lives in compiled native libraries (.so files). This is where the interesting stuff hides.

### Ghidra Workflow

```bash
# Launch Ghidra
ghidraRun &

# Import the .so file
# Analyze → Auto Analysis → ARM:LE:32:v8 (for 32-bit ARM)

# Search for string references
Search → For Strings → "password", "key", "secret"

# Cross-reference interesting functions
Right-click function → Show References To

# Decompile to pseudo-C
Window → Decompile ( Decompiler )
```

### Ghidra Headless Analysis Script

```java
// analyze_native.java — Run in Ghidra Script Manager
import ghidra.app.script.GhidraScript;
import ghidra.program.model.listing.*;
import ghidra.program.model.symbol.*;
import java.util.ArrayList;

public class analyze_native extends GhidraScript {
    @Override
    public void run() throws Exception {
        println("[*] Starting native analysis...");

        // Find all exported functions
        var funcs = currentProgram.getFunctionManager().getFunctions(true);
        while (funcs.hasNext()) {
            var func = funcs.next();
            if (func.isExternal()) continue;

            var body = func.getBody();
            var addrSet = body.getAddressRanges();
            while (addrSet.hasNext()) {
                var addr = addrSet.next().getMinAddress();
                var instr = getInstructionAt(addr);
                if (instr == null) continue;

                // Look for string loads into registers
                if (instr.getMnemonicString().equals("LDR")) {
                    println("[STR] " + func.getName() + ": " + instr);
                }
            }
        }
    }
}
```

### Frida Native Hooking

```javascript
// native_hook.js — Hook native functions
Interceptor.attach(Module.findExportByName("libtarget.so", "Java_com_target_MainActivity_authenticate"), {
    onEnter: function(args) {
        this.env = args[0];
        this.context = args[1];
        var username = Java.vm.getEnv().getStringUtfChars(args[2]);
        var password = Java.vm.getEnv().getStringUtfChars(args[3]);
        console.log("[NATIVE] Username: " + username);
        console.log("[NATIVE] Password: " + password);
    },
    onLeave: function(retval) {
        console.log("[NATIVE] authenticate returned: " + retval);
        retval.replace(1);  // Force return true
    }
});
```

### ELF Analysis Commands

```bash
# Read ELF headers
readelf -h libtarget.so

# List shared library dependencies
readelf -d libtarget.so | grep NEEDED

# Disassemble specific function
objdump -d -M intel libtarget.so | grep -A 50 "authenticate"

# Search for interesting strings
strings -n 8 libtarget.so | grep -iE "key|token|api|secret|http"

# Check for anti-debugging
grep -c "ptrace" libtarget.so
```

---

## Anti-Debug Bypasses

Every good app has anti-debugging. Here's how to get around the common ones. I use these scripts on almost every engagement.

### ptrace Self-Attach Detection

```javascript
// bypass_ptrace.js — Prevent anti-debug via ptrace
Interceptor.attach(Module.findExportByName("libc.so", "ptrace"), {
    onEnter: function(args) {
        this.request = args[0];
    },
    onLeave: function(retval) {
        if (this.request.toInt32() === 0) {  // PTRACE_TRACEME
            console.log("[+] ptrace(TRACEME) intercepted — returning 0");
            retval.replace(ptr(0));
        }
    }
});
```

### Debug Flag Detection

```javascript
// bypass_debug_check.js
Java.perform(function() {
    // Hook Debug.isDebuggerConnected()
    var Debug = Java.use("android.os.Debug");
    Debug.isDebuggerConnected.implementation = function() {
        console.log("[+] isDebuggerConnected() → false");
        return false;
    };

    // Hook ApplicationInfo.debuggable
    var ApplicationInfo = Java.use("android.content.pm.ApplicationInfo");
    ApplicationInfo.debuggable.value = false;
});
```

### Common Anti-Debug Methods

| Technique | Detection | Bypass |
|-----------|-----------|--------|
| `ptrace(PTRACE_TRACEME)` | Single-attach prevention | Hook `ptrace()` in libc |
| `Debug.isDebuggerConnected()` | Java debug check | Hook return value |
| `android:debuggable=true` manifest check | App self-check | Modify manifest or hook |
| `/proc/self/status` TracerPid check | Kernel-level detection | Hook `open()` to fake response |
| `timediff` execution checks | Runtime timing | Hook `SystemClock.elapsedRealtime()` |
| `kill(getpid(), SIGTRAP)` | Signal-based trap | Hook `kill()` to suppress signal |
| `inotify` file monitoring | Detect instrumentation | Hook `inotify_add_watch()` |

### Full Anti-Debug Bypass Script

```javascript
// full_bypass.js — Comprehensive anti-debug collection
Java.perform(function() {
    // 1. Debug flag
    var Debug = Java.use("android.os.Debug");
    Debug.isDebuggerConnected.implementation = function() { return false; };

    // 2. ptrace
    Interceptor.attach(Module.findExportByName("libc.so", "ptrace"), {
        onEnter: function(args) { this.req = args[0]; },
        onLeave: function(retval) {
            if (this.req.toInt32() === 0) retval.replace(ptr(0));
        }
    });

    // 3. TracerPid
    var File = Java.use("java.io.File");
    File.$init.overload("java.lang.String").implementation = function(path) {
        if (path.endsWith("/status")) {
            console.log("[+] Redirecting /proc/self/status");
            this.$init("/dev/null");
            return;
        }
        this.$init(path);
    };

    // 4. Timing check
    var SystemClock = Java.use("android.os.SystemClock");
    SystemClock.elapsedRealtime.implementation = function() {
        return 1000;  // Return constant
    };

    console.log("[+] All anti-debug measures bypassed");
});
```

---

## Practice Apps

Legal targets for honing your Android RE skills. Start with DIVA if you're new.

| App | Focus Area | Difficulty | Download |
|-----|-----------|------------|----------|
| **DIVA (Damn Insecure Vulnerable App)** | Insecure storage, crypto, auth | Beginner | [GitHub](https://github.com/payatu/diva-android) |
| **InsecureShop** | OWASP Mobile Top 10 vulnerabilities | Intermediate | [GitHub](https://github.com/abhishake100/InsecureShop) |
| **Vuldroid** | SSL pinning, root detection, tampering | Intermediate | [GitHub](https://github.com/jaiswalakshans/Vuldroid) |
| **Crackmes.one** | Community-submitted crackme challenges | Varies | [Website](https://crackmes.one/) |
| **OWASP UnCrackable Apps** | Anti-reverse engineering, obfuscation | Advanced | [GitHub](https://github.com/OWASP/owasp-mstg/tree/master/Crackmes) |
| **Frida Labs** | Frida-specific hooking exercises | Intermediate | [GitHub](https://github.com/ChiChou/frida-labs) |
| **Hancito's Android CTF** | Mobile CTF challenges | Intermediate | [GitHub](https://github.com/nicokosi/android-reverse-engineering) |

---

## Resources

Essential references for deepening Android RE knowledge.

### Books
- *Android Hacker's Handbook* — Joshua Drake et al.
- *The Mobile Application Hacker's Handbook* — Dominic Chell et al.
- *Learning Android Mobile Application Development* — beginner-friendly intro

### Courses
- [SANS SEC575](https://www.sans.org/courses/sec575-mobile-device-security) — Mobile Device Security
- [Coursera Mobile Security](https://www.coursera.org/learn/mobile-security) — Android platform security
- [Frida.re docs](https://frida.re/docs/examples/) — Official Frida tutorials

### Community
- [XDA Developers](https://xdadevelopers.com/) — Android modding community
- [r/AndroidRE](https://reddit.com/r/AndroidRE) — Reverse engineering subreddit
- [Android Security Awesome](https://github.com/ashishb/android-security-awesome) — Curated tool list

---

## Disclaimer

This toolkit is for **educational and authorized security testing purposes only**. Do not use these techniques on applications you do not own or have explicit permission to test. Unauthorized reverse engineering may violate local laws.

---

## License

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](https://opensource.org/licenses/MIT)

MIT License. See [LICENSE](LICENSE).

---

<p align="center">
  <sub>Maintained by <a href="https://github.com/ykrishhh">ykrishhh</a> — Security Researcher & Developer</sub>
</p>
