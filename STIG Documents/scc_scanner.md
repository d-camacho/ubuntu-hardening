# DISA SCC Scanner — Ubuntu 24.04 LTS (CLI)

> [!NOTE]
> These instructions are for Ubuntu 24.04 **CLI only** (no display server required).

> [!NOTE]
> SCC (SCAP Compliance Checker) is DISA's official scanning tool for STIG compliance validation.
> It is strongly recommended to run SCC both before hardening and after hardening:
> before hardening for a baseline scan, and after hardening for final compliance evidence.

> [!IMPORTANT]
> Run SCC **after** `ansible-playbook os_hardening_main.yml` completes and the host has rebooted.
> This scan is the manual post-hardening validation step for compliance evidence collection.

---

## 1. Install Dependencies

Prevents SCC from crashing in headless/CLI mode.

```bash
sudo apt update && sudo apt install -y \
  unzip \
  libfuse2t64 \
  libgtk2.0-0t64 \
  libgdk-pixbuf2.0-0 \
  libxss1 \
  libasound2t64 \
  libnss3 \
  libx11-xcb1
```

## 2. Download and transfer SCC bundle

On a different computer, download the DISA SCC for Ubuntu 22/24 AMD64 from the [DISA Cyber Exchange Website](https://www.cyber.mil/stigs/SCAP/) and transfer it to the target VM using SCP. Once transferred, unzip the SCC bundle.

```bash
sudo apt install unzip
unzip scc-5.14_ubuntu22_ubuntu24_amd64_bundle.zip -d ~/SCC
```

This creates a directory with the following extracted files:

```
~/SCC/scc-5.14_ubuntu22_amd64/
    scc-5.14.ubuntu.22_amd64.deb
    scc-5.14.ubuntu.22_amd64.deb.sig
```

## 3. Install SCC

```bash
sudo dpkg -i ~/SCC/scc-5.14_ubuntu22_amd64/scc-5.14.ubuntu.22_amd64.deb
sudo apt -f install   # run only if dpkg reports missing dependencies
```

SCC installs to `/opt/scc`.

## 4. Confirm SCAP Content Exists

```bash
ls /opt/scc/Resources/Content/SCAP12_Content
```

You should see:

```
U_CAN_Ubuntu_24-04_LTS_V1R3_STIG_SCAP_1-4_Benchmark-enhancedV5-signed.xml
```

This is the DISA STIG benchmark for Ubuntu 24.04.

## 5. Run a Local Scan

To run a scan, execute `cscc` with no arguments:

```bash
sudo /opt/scc/cscc
```

`cscc` automatically:
- Loads the default configuration profile
- Loads all enabled benchmarks (Ubuntu STIG is enabled by default)
- Runs a full local scan and generates results
- Creates a timestamped session directory

If the target was hardened by this project, run SCC locally on the hardened host rather than from the control node.

## 6. Find the Results

After the scan completes, SCC prints the output location:

```
Session Results (XML, HTML, Text) are located in:
/home/<user>/SCC/Sessions/<timestamp>/Results/SCAP

Session Logs are located in:
/home/<user>/SCC/Sessions/<timestamp>/Logs/
```

Example:

```
~/SCC/Sessions/2026-03-06_160954/Results/
    results.html
    results.xml
    results.ckl      <- load this into STIG Viewer
```

